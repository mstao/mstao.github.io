---
title: 在Service层进行事务控制
categories: Java
tags: java
author: Mingshan
date: 2017-09-15
---
## 利用接口回调实现JDBCTemplate

1. 设计一个回调接口JDBCCallback<T>, 用来设置参数和获取结果集, 代码如下：<br>
```java
public interface JDBCCallback<T> {

	T rsToObject(ResultSet rs) throws SQLException;

	void setParams(PreparedStatement pstmt) throws SQLException;
}
```

2. 设计一个抽象类JDBCAbstractCallBack<T>，该类实现JDBCCallback<T>接口，重写接口中的两个方法，
    不需要具体实现，只需要重写一下就可以了，这样在DAO层用的时候不用这两个方法全部都要实现，代码如下：
```java
/**
 * JDBC abstract callback class, implements {@link JDBCCallback} interface.
 * @author Mingshan
 *
 * @param <T>
 */
public abstract class JDBCAbstractCallBack<T> implements JDBCCallback<T> {

	@Override
	public T rsToObject(ResultSet rs) throws SQLException {
		return null;
	}

	@Override
	public void setParams(PreparedStatement pstmt) throws SQLException {
		// NOOP
	}
}
```

3. 设计一个JDBCTemplate类，该类实现增删改查的基本方法，把公共的代码抽取出来，以便DAO层去调用JDBCTemplate来实现具体的业务，部分代码如下：
<!-- more -->
```java
/**
 * JDBC Template, applys for delete, query, update, save functions.
 * @author Mingshan
 *
 * @param <T>
 */
public class JDBCTemplate<T> {

   /**
    * Querys data by sql.
    */
    public List<T> query(String sql, JDBCCallback<T> jdbcCallback) {
        Connection conn = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;
        List<T> data = new ArrayList<T>();
        boolean needMyClose = false;

        try {
            // Gets connection of JDBC.
            ConnectionHolder connectionHolder = (ConnectionHolder) AppContext.getAppContext()
                    .getObject(Constants.APP_REQUEST_THREAD_CONNECTION);
            if (connectionHolder != null) {
                conn = connectionHolder.getConn();
            }
            if (conn == null) {
                conn = DB.getConn();
                needMyClose = true;
            }
            pstmt = DB.getPrepareStatement(conn, sql);
            // Sets parameters for PreparedStatement.
            jdbcCallback.setParams(pstmt);
            rs = pstmt.executeQuery();

            while (rs.next()) {
                // Gets data from database.
                T object = jdbcCallback.rsToObject(rs);
                data.add(object);
            }
        } catch (Exception e) {
            e.printStackTrace();
            throw new DBException();
        } finally {
            DB.close(rs);
            DB.close(pstmt);
            if (needMyClose) {
                DB.close(conn);
            }

        }

        return data;
    }

    public void update(String sql, JDBCCallback<T> jdbcCallback) {
        Connection conn = null;
        PreparedStatement pstmt = null;
        boolean needMyClose = false;
        try {
            ConnectionHolder connectionHolder = (ConnectionHolder) AppContext.getAppContext()
                    .getObject(Constants.APP_REQUEST_THREAD_CONNECTION);
            if (connectionHolder != null) {
                conn = connectionHolder.getConn();
            }
            if (conn == null) {
                conn = DB.getConn();
                needMyClose = true;
            }

            pstmt = DB.getPrepareStatement(conn, sql);
            jdbcCallback.setParams(pstmt);
            pstmt.execute();
        } catch (SQLException e) {
            e.printStackTrace();
            throw new DBException();
        } finally {
            DB.close(pstmt);
            if (needMyClose) {
                DB.close(conn);
            }
        }
    }

}
```

其他的增删改查可以按照上面的模式进行扩展，就不写了。

4. 在DAO层使用JDBCTemplate，部分代码如下：
```java
public class UserDaoImpl implements UserDao {
    private JDBCTemplate<User> jdbcTemplate;

    public void setJdbcTemplate(JDBCTemplate<User> jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public User findUserByUserName(final String userName) {
        User user = null;
        String sql = "SELECT * FROM user WHERE user_name = ?";
        user = jdbcTemplate.queryOne(sql, new JDBCAbstractCallBack<User>() {

            @Override
            public User rsToObject(ResultSet rs) throws SQLException {
                User user = new User();
                user.setPassword(rs.getString(Constants.USER_PASSWORD));
                user.setUserName(rs.getString(Constants.USER_USER_NAME));
                user.setId(rs.getInt(Constants.USER_ID));
                return user;
            }

            @Override
            public void setParams(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, userName);
            }

        });

        return user;
    }
}

```

—————————————————————————OVER————————————————————————————
* * *

## 如何在Service层进行事务控制？

1. 设计一个ConnectionHolder类，用来存放Connection。
	该类有两个成员变量 Connection conn, boolean isOpenTransaction, 并且提供getter和setter方法。
	Connection conn 用来存放Connection, boolean isOpenTransaction 用来判断需不需要开启事务

2. 编写ConnectionProxy类，并实现InvocationHandler接口，该类用来真正实现事务控制，具体解析如下：
   1. 首先需要获取配置的事务传播，用来判断哪些方法需要进行事务控制，哪些不需要，可以参考Spring配置事务的代码，这里先模拟一下，XML配置信息如下，然后需要对XML配置信息进行解析，然后以map形式返回，这时我们就可以按照我们的需求来判断当前要调用Service层的方法到底是属于哪一种，进而判断是否要进行事务控制。如果需要关闭数据库连接，那么数据库连接应在代理类中关闭。

   2. 判读connectionHolder对象是否已被创建，如果已被创建，直接使用，然后进行事务控制判断；如果不存在，那么在这里创建，需要拿到数据库连接，然后进行事务控制判断。这里用到了Connection共用，即一个request只有一个Connection。

   3. 利用反射调用方法， 此时方法会出现异常， 需要进行捕获。如果事务开启，并且调用方法出现异常了，那么就需要事     务回滚，最后关闭连接。

   4. DAO层的写法请参考JDBCTemplate, 为了防止数据库连接中断，需要DAO层再进行一次连接判断，此时数据库的连接就     需要在DAO层关闭了。

事务配置：
```XML

<!-- TransactionInterceptor -->
<bean id="TransactionInterceptor">
    <property name="transactionAttributes">
        <prop key="update*" propagation="REQUIRED"></prop>
        <prop key="delete*" propagation="REQUIRED"></prop>
        <prop key="save*" propagation="REQUIRED"></prop>
        <prop key="find*" propagation="SUPPORTS"></prop>
        <prop key="select*" propagation="SUPPORTS"></prop>
    </property>
</bean>

```

代理类代码如下：
```java
/**
 * JDBC ConnectionProxy.
 * @author Mingshan
 *
 */
public class ConnectionProxy implements InvocationHandler {
    private Object target;
    private TransactionConfig transactionConfig = AppContext.getAppContext().getTransactionConfig();

    public void setTarget(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result = null;
        // If current thread close connection.
        boolean needMyClose = false;
        boolean isCommitOrRollBackTran = false;
        Map<String, String> tranAttributeMap = transactionConfig.getTranAttributeMap();
        String[] allowed = getTransactionAttribute(transactionConfig);
        String originKey = method.getName() + "*";
        // Before advice.
        ConnectionHolder connectionHolder = (ConnectionHolder) AppContext.getAppContext()
                .getObject(Constants.APP_REQUEST_THREAD_CONNECTION);

        if (connectionHolder == null) {
            Connection conn = DB.getConn();
            connectionHolder = new ConnectionHolder();
            connectionHolder.setConn(conn);

            if (StringUtil.matchStr(method.getName(), allowed) && (tranAttributeMap.get(originKey).equals("REQUIRED"))) {
                connectionHolder.setOpenTran(true);
                DB.setAutoCommit(conn, false);
                isCommitOrRollBackTran = true;
            }

            AppContext.getAppContext().addObject(Constants.APP_REQUEST_THREAD_CONNECTION, connectionHolder);
            isCommitOrRollBackTran = true;
            needMyClose = true;
        } else {
            if (StringUtil.matchStr(method.getName(), allowed) && (tranAttributeMap.get(originKey).equals("REQUIRED"))) {
                if (!connectionHolder.isOpenTran()) {
                    connectionHolder.setOpenTran(true);
                    DB.setAutoCommit(connectionHolder.getConn(), false);
                    isCommitOrRollBackTran = true;
                }
            }
        }

        try {
            result = method.invoke(target, args);

            if (isCommitOrRollBackTran) {
                DB.commit(connectionHolder.getConn());
            }
        } catch (Throwable throwable) {
            if (isCommitOrRollBackTran) {
                DB.rollback(connectionHolder.getConn());
            }
        } finally {
            // After advice.
            if (needMyClose) {
                connectionHolder = (ConnectionHolder) AppContext.getAppContext()
                        .getObject(Constants.APP_REQUEST_THREAD_CONNECTION);
                DB.close(connectionHolder.getConn());
            }
        }

        return result;
    }

    /**
     * Stores key in map as an array.
     * @param transactionConfig
     * @return String[]
     */
    public String[] getTransactionAttribute(TransactionConfig transactionConfig) {
        Map<String, String> tranAttributeMap = transactionConfig.getTranAttributeMap();
        Set<String> keySet = tranAttributeMap.keySet();
        String[] methodPrefixs =new String[keySet.size()];
        int i = 0;

        for (String key : keySet) {
            int index = key.indexOf("*");
            if (index == -1) {
                key = key.substring(0, key.length() - 1);
                methodPrefixs[i] = key;
            } else {
                methodPrefixs[i] = key;
            }
            i++;
        }

        return methodPrefixs;
    }

}

```
## 附：ApplicationContextFilter

```java
/**
 * Application Context Filter, include {@link HttpServletRequest} request,
 * {@link HttpServletResponse} response, {@link Connection} JDBC Connection.
 * @author Mingshan
 *
 */
public class AppContextFilter implements Filter {
    private TransactionConfig transactionConfig = null;
    public AppContextFilter() {}

    public void init(FilterConfig fConfig) throws ServletException {
        ServletContext servletContext = fConfig.getServletContext();
        transactionConfig = (TransactionConfig) servletContext.getAttribute("transactionConfig");
    }

    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        AppContext appContext = AppContext.getAppContext();
        appContext.addObject(Constants.APP_CONTEXT_REQUEST, request);
        appContext.addObject(Constants.APP_CONTEXT_RESPONSE, response);
        appContext.setTransactionConfig(transactionConfig);
        ConnectionHolder connectionHolder = (ConnectionHolder) AppContext.getAppContext()
                .getObject(Constants.APP_REQUEST_THREAD_CONNECTION);
        boolean needMyClose = false;
        if(connectionHolder == null) {
            connectionHolder = new ConnectionHolder();
            Connection conn = DB.getConn();
            connectionHolder.setConn(conn);
            AppContext.getAppContext().addObject(Constants.APP_REQUEST_THREAD_CONNECTION, connectionHolder);
            needMyClose = true;
        }

        try {
            chain.doFilter(request, response);
        } catch (IOException ioException) {
            throw ioException;
        } catch (ServletException servletException) {
            throw servletException;
        } catch (RuntimeException runntimeException) {
            throw runntimeException;
        } finally {
            if (needMyClose) {
                connectionHolder = (ConnectionHolder) AppContext.getAppContext()
                        .getObject(Constants.APP_REQUEST_THREAD_CONNECTION);
                DB.close(connectionHolder.getConn());
            }
            appContext.clear();
        }

    }

    public void destroy() {
        // NOOP
    }
}

```
—————————————————————————OVER————————————————————————————
