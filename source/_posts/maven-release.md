---
title: Maven发布RELEASE包到云效私有仓库并打TAG
categories: Maven
tags: [Git, Maven]
author: Mingshan
date: 2019-11-13
---

我们平时把开源代码托管在github上，当代码发展到某一程度，准备打个RELEASE包放到内网私有仓库或者Maven中央仓库，这时就需要用到**Maven Release Plugin**，这个插件专为打包准备的，我们要知道怎么集成到自己的项目中和正确打包。这里以将RELEASE包发布到[阿里云云效私有仓库](https://repomanage.rdc.aliyun.com/my/repo)为例，记录下打包的流程。

<!-- more -->

## Maven相关配置

由于要打包到远程仓库，所以本地的maven需要配置，根据不同的远程仓库的不同，配置也不一样。这里以阿里云的云效私有仓库为例配置settings.xml，下面是手动配置步骤：

**1.在servers节点添加如下配置**

```xml
<servers>
    <server>
        <id>rdc-releases</id>
        <username>etfqHW</username>
        <password>******</password>
    </server>
    <server>
        <id>rdc-snapshots</id>
        <username>etfqHW</username>
        <password>******</password>
    </server>
</servers>
```

**2. 在profiles节点添加如下配置**

```xml
<profile>
    <id>rdc-private-repo</id>
    <repositories>
        <repository>
            <id>rdc-releases</id>
            <url>https://repo.rdc.aliyun.com/repository/110401-release-HjFVTW/</url>
        </repository>
        <repository>
            <id>rdc-snapshots</id>
            <url>https://repo.rdc.aliyun.com/repository/110401-snapshot-n3LRba/</url>
        </repository>
    </repositories>
</profile>
```

## 主pom文件配置

**1. 制品上传配置**

在代码库根目录下的pom.xml加入以下配置:

```xml
  <distributionManagement>
    <repository>
      <id>rdc-releases</id>
      <url>https://repo.rdc.aliyun.com/repository/110401-release-HjFVTW/</url>
    </repository>
    <snapshotRepository>
      <id>rdc-snapshots</id>
      <url>https://repo.rdc.aliyun.com/repository/110401-snapshot-n3LRba/</url>
    </snapshotRepository>
  </distributionManagement>
```

**2. scm 相关配置**

接着需要配置项目的License，Scm地址以及开发者信息，如下：

```xml
  <licenses>
    <license>
      <name>Apache License, Version 2.0</name>
      <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
      <distribution>repo</distribution>
    </license>
  </licenses>

  <developers>
    <developer>
      <name>Walker Han</name>
      <email>walkerhan@126.com</email>
    </developer>
  </developers>

  <scm>
    <connection>scm:git:https://github.com/mstao/hutils.git</connection>
    <developerConnection>scm:git:https://github.com/mstao/hutils.git</developerConnection>
    <url>https://github.com/mstao/hutils</url>
    <tag>${project.version}</tag>
  </scm>

  <inceptionYear>2019</inceptionYear>
```

**3. 配置编译插件**

需要用到：`maven-compiler-plugin`编译插件，并指定编译的JDK版本；`maven-source-plugin`源码打包；`maven-release-plugin`发布插件，并指定TAG，子模块打包选项；`maven-deploy-plugin`部署插件，将制品上传到指定仓库。

```xml
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.6.2</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>
      <!--配置生成源码包-->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-source-plugin</artifactId>
        <version>3.0.1</version>
        <executions>
          <execution>
            <id>attach-sources</id>
            <goals>
              <goal>jar</goal>
            </goals>
          </execution>
        </executions>
      </plugin>

      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-release-plugin</artifactId>
        <version>2.5.3</version>
        <configuration>
          <tagNameFormat>v@{project.version}</tagNameFormat>
          <autoVersionSubmodules>true</autoVersionSubmodules>
        </configuration>
      </plugin>

      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-deploy-plugin</artifactId>
        <version>2.8.2</version>
      </plugin>
    </plugins>
  </build>
```

## Maven Release步骤

下面是官方给出的使用到**Maven Release Plugin**的步骤：

- **release:clean** Clean up after a release preparation.
- **release:prepare** Prepare for a release in SCM.
- **release:prepare-with-pom** Prepare for a release in SCM, and generate release POMs that record the fully resolved projects used.
- **release:rollback** Rollback a previous release.
- **release:perform** Perform a release from SCM.
- **release:stage** Perform a release from SCM into a staging folder/repository.
- **release:branch** Create a branch of the current project with all versions updated.
- **release:update-versions** Update the versions in the POM(s).

平时用的时候，只会用到以下三个命令

- release:prepare 
- release:rollback
- release:perform

### release:prepare

这个是RELEASE的准备阶段。通常会带几个参数（`-DautoVersionSubmodules=true`子模块也进行RELEASE，`-Darguments="-DskipTests"`跳过单元测试），如下：

```
mvn release:prepare -DautoVersionSubmodules=true -Darguments="-DskipTests"
```

该阶段做的事情比较多，参考官网的信息，主要有以下步骤：

1. 检查有没有已更改但未提交的代码，有的话，直接报错
2. 检查有没有三方库的SNAPSHOT版本依赖
3. 将当前的快照版本变为正式版本，比如当前版本是1.0-SNAPSHOT，那么正式版本是1.0，这个过程会提示你输入确认的版本号，同时也提示你输入新的快照版本号，如下所示：

```
[INFO] Checking dependencies and plugins for snapshots ...
What is the release version for "hutils"? (me.mingshan.util:hutils) 1.0: : 1.0
What is SCM release tag or label for "hutils"? (me.mingshan.util:hutils) v1.0: : v1.0
What is the new development version for "hutils"? (me.mingshan.util:hutils) 1.1-SNAPSHOT: : 1.1-SNAPSHOT
```

4. 为源码库打TAG，更新POM，并运行单元测试保证运行无误
5. 提交上述更新到远程仓库
6. 更新源码库为新的快照版本，比如1.1-SNAPSHOT，提交POM到远程仓库

当然上面只是大致的准备流程，在真正运行时，还会生成许多的准备文件，这个我们不用管它。

### release:rollback

当`release:prepare`执行完，我们发现版本号设置得不合理，想要回滚，这时不能手动删除文件，因为文件太多，这时就用到`release:rollback`来回滚上面的操作。执行如下命令：

```
mvn release:rollback
```

回滚操作会删除生成的缓存文件，还原更改过的版本号，但不会删除已经打好TAG，而TAG对一个仓库来说是唯一的，如果不删除，那么下次再用这个TAG就会报错，所以需要将远程仓库的TAG删除，执行如下命令：

```
git tag -d [tag];  
git push origin :[tag]  
```

比如我要删除`v1.0`这个TAG，那么执行以下命令就好了：

```
git tag -d v1.0
git push origin :v1.0
```

### release:perform

如果不打算回滚，那么就可以执行`release:perform`了，命令如下：

```
mvn release:perform
```

该命令会做以下事情：

1. 从远程仓库拉取当前TAG的源码，进行编译。注意会编译新的快照版本的代码和将要RELEASE的代码，并运行单元测试
2. 执行相关插件（source, javadoc），注意如果注释写的不规范，运行javadoc插件是会报错的，比如注释里面有`<br/>`，`->` 等
3. 将打包好的文件上传到配置的仓库。对于RELEASE版本，默认是只能上传一次的，再次上传就会报错。

最后就可以在项目中直接依赖刚才打包好的RELEASE版本啦。

## References：

- [Maven Release Plugin](http://maven.apache.org/maven-release/maven-release-plugin/index.html)
