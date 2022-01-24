---
title: 如何发包到中央仓库-完整教程
categories: Maven
tags: [Maven]
author: Mingshan
date: 2022-01-21
---

相信很多小伙伴自己都会有一些包，我们可以将包发布中央仓库，然后直接通过maven来使用这些包，下面就整理下如何将包发到中央仓库。

<!-- more -->

## 新建项目

第一步，我们需要在[sonatype](https://issues.sonatype.org/)上注册一个账号，这一步就不再演示了。

注册好账户之后，然后我们来创建项目。点击**新建**按钮，问题类型 选择 **New Project**，如下所示：

![image](https://user-images.githubusercontent.com/23411433/150465809-2b2f0628-bf0a-49ab-b717-c139e2fff60d.png)

在该页面中，注意需要添加**groupId**，项目的描述信息等，填写完毕后进行保存。用过jira的同学对这个系统肯定十分熟悉，因为就是jira。

注意**如果groupId以前没有在这个上面使用过，那么第一次使用需进行注册，然后才可以发包**，新建好的项目会收到以下评论：

![image](https://user-images.githubusercontent.com/23411433/150466057-4169ec4c-6a6d-4700-aa6a-21528a6d8542.png)

翻译过来有以下几点：

如果你有自己的域名，groupId就可以是将域名反转过来的，比如`fun.mingshan`，那么你可以：

- 在你的域名添加一条DNS解析，需要是**TXT** 类型的，指向当前创建的issue
- 重新打开这个一条issue

如果你没有自己的域名，那么你可以：

- 使用github这种开放的域名，不过需要创建一个仓库来验证github账户的权限

大概是这样，我选用有域名添加一条DNS解析这种操作，十分快捷。DNS解析规则如下：

![image](https://user-images.githubusercontent.com/23411433/150467006-bef1f0f2-42a5-4cc5-b982-7009f2d46b75.png)

上面的操作完，等issue的状态变为**已解决**时，会收到一条评论：

![image](https://user-images.githubusercontent.com/23411433/150467080-02c034c3-56de-43bb-88c8-3ee15e130085.png)

我们接下来就可以将包发送到中央仓库了。

# 发送步骤

## 更新pom.xml文件

增加开源协议，坐着，SCM信息等，具体信息可以参考：

```xml
  <groupId>fun.mingshan</groupId>
  <artifactId>markdown4j</artifactId>
  <version>1.0-SNAPSHOT</version>

  <name>markdown4j</name>
  <description>To solve the problem of not quickly generating Markdown files at the Java level, using builder mode, very simple to use.</description>
  <url>https://github.com/mstao/markdown4j</url>

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

  <issueManagement>
    <system>Github Issue</system>
    <url>https://github.com/mstao/markdown4j/issues</url>
  </issueManagement>

  <scm>
    <connection>scm:git:https://github.com/mstao/markdown4j.git</connection>
    <developerConnection>scm:git:https://github.com/mstao/markdown4j.git</developerConnection>
    <url>https://github.com/mstao/markdown4j</url>
    <tag>${project.version}</tag>
  </scm>

  <inceptionYear>2022</inceptionYear>
```

增加插件：

```xml
 <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <encoding>UTF-8</encoding>
          <source>${java.version}</source>
          <target>${java.version}</target>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>2.18.1</version>
        <configuration>
          <skipTests>true</skipTests>
        </configuration>
      </plugin>
      <plugin>
        <artifactId>maven-source-plugin</artifactId>
        <version>2.4</version>
        <configuration>
          <attach>true</attach>
        </configuration>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>jar-no-fork</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-site-plugin</artifactId>
        <version>3.7.1</version>
      </plugin>
      <!-- Javadoc -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-javadoc-plugin</artifactId>
        <version>3.0.1</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>jar</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      <!-- Gpg Signature -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-gpg-plugin</artifactId>
        <version>1.6</version>
        <executions>
          <execution>
            <id>sign-artifacts</id>
            <phase>verify</phase>
            <goals>
              <goal>sign</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
```

增加distributionManagement：

```xml
  <distributionManagement>
    <snapshotRepository>
      <id>oss</id>
      <url>https://oss.sonatype.org/content/repositories/snapshots/</url>
    </snapshotRepository>
    <repository>
      <id>oss</id>
      <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
    </repository>
  </distributionManagement>
```

## 更新maven settings.xml文件

```xml
<settings>
  <servers>
    <server>
      <id>ossrh</id>
      <username>your-jira-id</username>
      <password>your-jira-pwd</password>
    </server>
  </servers>
</settings>
```

## 配置PGP公匙信息

![image](https://user-images.githubusercontent.com/23411433/150723396-aa5cea1f-3135-4635-a251-937846ab6151.png)

![image](https://user-images.githubusercontent.com/23411433/150723548-66173f98-6990-46c2-8be5-959467301513.png)


# 参考

- https://central.sonatype.org/publish/publish-maven/
- https://www.cnblogs.com/songyz/p/11387978.html
- https://blog.csdn.net/pdsu161530247/article/details/105429597
