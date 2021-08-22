---
title: 自定义乐观锁插件结合tkmybatis使用
tags: [java,tkmybatis,乐观锁]
author: Mingshan
categories: Java
date: 2021-08-22
---

在tkmybatis中使用版本号实现乐观并发控制并不很方便，实际使用时需要根据具体的业务具体来封装，这里提供一个简单的封装，来实现对特定方法的拦截。

<!-- more -->

首先定义一个实体，只有一个版本号属性：
```java
public class VersionPEntity {
    @Version
    private Long version;

    public Long getVersion() {
        return version;
    }

    public void setVersion(Long version) {
        this.version = version;
    }
}
```
## 拦截器编写思路
mybatis提供拦截器机制可对sql进行复杂操作（比如改写sql），所以这里我们利用这个特性来自动给更新和删除的sql语句加上版本号控制。拦截器内部逻辑：


1. 获取sql的参数实体，实体必须继承自VersionPEntity，否则不进行sql改写。这个地方逻辑可以根据自己的业务来实现
1. 获取VersionPEntity中版本号，作为当前的版本号 `currentVersion` 
1. 下面进入到SQL改写逻辑



### SQL 改写逻辑
sql中必须有where 条件，且不能有in关键字，用正则来判断


判断sql类型，只改写update 和 delete语句
#### DELETE 语句改写
如果是delete，那么直接  在sql后拼接 ` and verision = currentVersion `


#### UPDATE语句改写


如果是update语句，考虑到sql中需要对version字段赋值操作，需要考虑原先sql有没有`version = ?`赋值操作


- 如果原先sql 中已有 `version = ?`， 那么我们就不需要进行拼接version赋值操作，只需要对实体中的version字段进行+1操作，需要用反射来实现。
- 如果没有，那么需要根据where 将sql分为前后两部分，前半部分拼接 ` , version = version + 1 ` 



最后两种都要拼接` and verision = currentVersion `
            
### 代码如下：
```java
import me.mingshan.util.StringUtil;
import me.mingshan.util.orm.entity.VersionPEntity;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.*;
import org.apache.ibatis.plugin.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
import java.util.Properties;
import java.util.regex.Pattern;

/**
 * 版本号控制拦截器
 * <p>
 * <p>
 * DB层仅支持以下方法：
 * <ul>
 *    <li>1. updateByPrimaryKey</li>
 *    <li>2. delete</li>
 *    <li>3. updateByPrimaryKeySelective</li>
 *    <li>4. updateByExample</li>
 *    <li>5. updateByExampleSelective</li>
 * </ul>
 * <p>
 * <p>
 * 注意点：<br/>
 * <ul>
 *     <li>1. 数据库持久化对象需要继承{@link VersionPEntity}</li>
 *     <li>2. SQL语句必须带where</li>
 *     <li>3. SQL语句不能出现in操作</li>
 * </ul>
 * <p>
 * 参考：<br/>
 * https://www.jianshu.com/p/0a72bb1f6a21
 *
 * @author hanjuntao
 * @date 2021/8/2
 */
@Intercepts({@Signature(
        type = Executor.class,
        method = "update", // update 包括了最常用的 insert/update/delete 三种操作
        args = {MappedStatement.class, Object.class})})
public class OptimisticLockerInterceptor implements Interceptor {
    private static final Logger LOGGER = LoggerFactory.getLogger(OptimisticLockerInterceptor.class);
    /**
     * 支持拦截的方法
     */
    private static final Map<String, String> SUPPORT_METHODS_MAP = new HashMap<>();

    /*
     * 初始化
     */
    static {
        SUPPORT_METHODS_MAP.put("updateByPrimaryKey", null);
        SUPPORT_METHODS_MAP.put("delete", null);
        SUPPORT_METHODS_MAP.put("updateByPrimaryKeySelective", null);
        SUPPORT_METHODS_MAP.put("updateByExample", null);
        SUPPORT_METHODS_MAP.put("updateByExampleSelective", null);
    }

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameterObject = args[1];

        Object versionEntity = fetchVersionEntity(parameterObject);
        if (versionEntity == null) {
            return invocation.proceed();
        }

        // id为执行的mapper方法的全路径名，如com.mapper.UserMapper.updateByPrimaryKey
        String id = ms.getId();
        LOGGER.info("乐观锁，ID: {}", id);

        // 仅支持拦截特定的方法
        String[] targetMethodArr = id.split("\\.");
        String method = targetMethodArr[targetMethodArr.length - 1];
        boolean methodSupportIntercept = SUPPORT_METHODS_MAP.containsKey(method);
        if (!methodSupportIntercept) {
            return invocation.proceed();
        }

        // sql语句类型 select、delete、insert、update
        String sqlCommandType = ms.getSqlCommandType().toString();
        LOGGER.info("乐观锁，sqlCommandType: {}", sqlCommandType);

        BoundSql boundSql = ms.getBoundSql(parameterObject);
        String origSql = boundSql.getSql();
        LOGGER.info("乐观锁，原始SQL: {}", origSql);

        Long version = ((VersionPEntity) versionEntity).getVersion();

        // 改写SQL
        String newSql = rewriteSql(origSql, version, ms.getSqlCommandType(), versionEntity);
        if (origSql.equals(newSql)) {
            return invocation.proceed();
        }

        // 重新new一个查询语句对象
        BoundSql newBoundSql = new BoundSql(ms.getConfiguration(), newSql,
                boundSql.getParameterMappings(), boundSql.getParameterObject());

        // 把新的查询放到statement里
        MappedStatement newMs = newMappedStatement(ms, new BoundSqlSqlSource(newBoundSql));
        for (ParameterMapping mapping : boundSql.getParameterMappings()) {
            String prop = mapping.getProperty();
            if (boundSql.hasAdditionalParameter(prop)) {
                newBoundSql.setAdditionalParameter(prop, boundSql.getAdditionalParameter(prop));
            }
        }

        Object[] queryArgs = invocation.getArgs();
        queryArgs[0] = newMs;

        LOGGER.info("乐观锁，改写的SQL: {}", newSql);

        return invocation.proceed();
    }

    /**
     * 从参数中拿到 继承{@link VersionPEntity}的对象
     *
     * @param parameterObject 参数
     * @return 继承自VersionPEntity的对象
     */
    private static Object fetchVersionEntity(Object parameterObject) {
        Object versionEntity = null;

        // 必须是继承自VersionPEntity才可以使用版本号
        if (!(parameterObject instanceof VersionPEntity)) {
            if (Objects.nonNull(parameterObject) && parameterObject instanceof Map) {
                Object firstValue = ((Map<?, ?>) parameterObject).values().stream()
                        .filter(Objects::nonNull).findFirst().orElse(null);
                if (firstValue instanceof VersionPEntity) {
                    versionEntity = firstValue;
                } else {
                    return null;
                }
            }
        } else {
            versionEntity = parameterObject;
        }

        return versionEntity;
    }

    /**
     * 支持重新 update 语句和 delete 语句
     * <p>
     * 注意两者原始语句必须有 where ，没有where条件不处理，存在in的，也不处理
     * <p>
     * 场景1：
     * <p>
     * 原始SQL:     update tableName set name = 'AA', code = 'zz' where id = 1;
     * 改写后的SQL:  update tableName set name = 'AA', code = 'zz', version = version + 1 where id = 1 and version = oldVersion;
     * <p>
     * <p>
     * 场景2：
     * <p>
     * 原始SQL:     delete tableName where id = 1;
     * 改写后的SQL:  delete tableName where id = 1 and version = oldVersion;
     *
     * @param origSql        原始的sql
     * @param version        当前版本号
     * @param sqlCommandType sql类型
     * @param versionEntity  实体
     * @return 改写后的sql
     */
    private static String rewriteSql(String origSql, Long version, SqlCommandType sqlCommandType,
                                     Object versionEntity) {
        if (StringUtil.isEmpty(origSql) || version == null
                || sqlCommandType == null || versionEntity == null) {
            return origSql;
        }

        String lowCaseOrigSql = origSql.toLowerCase();

        // 判断sql是否有where条件，无where，不重写
        boolean existWhere = lowCaseOrigSql.contains("where");
        if (!existWhere) {
            return origSql;
        }

        // 判断是否有in，有in，不重写
        // 校验规则：in左边必须有空格，in右边可以没空格，没空格必须是(
        // 场景1：select  * from wbt_data_recovery where id in(39);    通过
        // 场景2：select  * from wbt_data_recovery where id in (39);   通过
        // 场景3：select  * from wbt_data_recovery where idIn = '39';  不通过
        // 场景4：select  * from wbt_data_recovery where inX = '39';   不通过
        String inReg = "^.*(\\s)+in(\\s|\\(){1}.*$";
        boolean existIn = Pattern.matches(inReg, lowCaseOrigSql);
        if (existIn) {
            return origSql;
        }

        // 更新sql改写
        if (SqlCommandType.UPDATE.equals(sqlCommandType)) {
            String[] sqlArr = origSql.split("(?i)where");

            String s1 = sqlArr[0];
            String s2 = sqlArr[1];

            String versionReg = "^.*(\\s)+version(\\s|=){1}.*$";
            boolean existVersion = Pattern.matches(versionReg, lowCaseOrigSql);
            if (existVersion) {
                try {
                    // 获取父类版本号属性
                    Field versionFiled = versionEntity.getClass().getSuperclass().getDeclaredField("version");
                    versionFiled.setAccessible(true);

                    // 获取属性值
                    Long value = (Long) versionFiled.get(versionEntity);
                    value = value + 1;
                    versionFiled.set(versionEntity, value);
                } catch (Exception e) {
                    return origSql;
                }

                s2 += " and version = " + version;
            } else {
                s1 += " , version = version + 1 ";
                s2 += " and version = " + version;
            }

            return s1 + " where " + s2;
        }
        // 更新sql改写
        if (SqlCommandType.DELETE.equals(sqlCommandType)) {
            String versionReg = "^.*(\\s)+version(\\s|=){1}.*$";
            boolean existVersion = Pattern.matches(versionReg, lowCaseOrigSql);

            if (!existVersion) {
                origSql += " and version = " + version;
            }

            return origSql;
        }

        return origSql;
    }

    /**
     * 定义一个内部辅助类，作用是包装 SQL
     */
    static class BoundSqlSqlSource implements SqlSource {
        private final BoundSql boundSql;

        public BoundSqlSqlSource(BoundSql boundSql) {
            this.boundSql = boundSql;
        }

        public BoundSql getBoundSql(Object parameterObject) {
            return boundSql;
        }

    }

    private MappedStatement newMappedStatement(MappedStatement ms, SqlSource newSqlSource) {
        MappedStatement.Builder builder = new
                MappedStatement.Builder(ms.getConfiguration(), ms.getId(), newSqlSource, ms.getSqlCommandType());
        builder.resource(ms.getResource());
        builder.fetchSize(ms.getFetchSize());
        builder.statementType(ms.getStatementType());
        builder.keyGenerator(ms.getKeyGenerator());
        if (ms.getKeyProperties() != null && ms.getKeyProperties().length > 0) {
            builder.keyProperty(ms.getKeyProperties()[0]);
        }
        builder.timeout(ms.getTimeout());
        builder.parameterMap(ms.getParameterMap());
        builder.resultMaps(ms.getResultMaps());
        builder.resultSetType(ms.getResultSetType());
        builder.cache(ms.getCache());
        builder.flushCacheRequired(ms.isFlushCacheRequired());
        builder.useCache(ms.isUseCache());
        return builder.build();
    }

    @Override
    public Object plugin(Object target) {
        if (target instanceof Executor) {
            return Plugin.wrap(target, this);
        }
        return target;
    }

    @Override
    public void setProperties(Properties properties) {
    }
}
```
## 使用方式


如果传入的版本号与数据库的版本号不一致，会更新/删除失败，返回行数为0，所以我们需要进行统一校验，封装 `**withVersion` 方法供业务层使用，代表这些方法都会走版本号拦截，代码如下：
```java
import me.mingshan.util.DaoUtils;
import tk.mybatis.mapper.common.Mapper;
import tk.mybatis.mapper.common.MySqlMapper;
import tk.mybatis.mapper.entity.Example;

/**
 * @author hanjuntao
 * @date 2021/8/10
 */
public interface MyMapper<T> extends Mapper<T>, MySqlMapper<T> {

    default int deleteWithVersion(T t){
        int result = delete(t);
        DaoUtils.checkAffectRows(result, 1);
        return result;
    }

    default int updateByPrimaryKeyWithVersion(T t){
        int result = updateByPrimaryKey(t);
        DaoUtils.checkAffectRows(result, 1);
        return result;
    }

    default int updateByPrimaryKeySelectiveWithVersion(T t) {
        int result = updateByPrimaryKeySelective(t);
        DaoUtils.checkAffectRows(result, 1);
        return result;
    }

    default int updateByExampleWithVersion(T t, Example example){
        int result = updateByExample(t, example);
        DaoUtils.checkAffectRows(result, 1);
        return result;
    }

    default int updateByExampleSelectiveWithVersion(T t, Example example){
        int result = updateByExampleSelective(t, example);
        DaoUtils.checkAffectRows(result, 1);
        return result;
    }

}
```
