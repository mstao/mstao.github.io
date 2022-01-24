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
          <source>1.8</source>
          <target>1.8</target>
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

        <configuration>
          <!-- jdk1.8要加上，1.7要去掉，否则会报错 -->
          <additionalJOptions>
            <additionalJOption>-Xdoclint:none</additionalJOption>
          </additionalJOptions>
        </configuration>
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
      <id>ossrh</id>
      <url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
    </snapshotRepository>
    <repository>
      <id>ossrh</id>
      <url>https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/</url>
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

Windows下使用gpg4win来进行配置。下载地址 https://www.gpg4win.org/get-gpg4win.html。

下载安装完后，打开命令行执行：`gpg --gen-key`，来生成公匙信息，注意此步需要输入sonatype 网站上的用户名，邮箱信息，必须保持一致。如下所示：

![image](https://user-images.githubusercontent.com/23411433/150752069-e67c6dd9-4ffb-460b-892b-65b8e795b2e0.png)

生成的过程中，会有个弹框要求输入**Passphase**信息，这个是密钥的密码，同样需要记牢。十分重要，后续在进行发布包的时候要用到的，如下：

![image](https://user-images.githubusercontent.com/23411433/150723396-aa5cea1f-3135-4635-a251-937846ab6151.png)

上面操作完之后，会生成公匙，需要将公钥发布到 PGP 密钥服务器，两个密钥服务器都发布一下：

```
gpg --keyserver hkp://pool.sks-keyservers.net --send-keys 公钥ID
gpg --keyserver hkp://keyserver.ubuntu.com:11371 --send-keys 公钥ID
```

查询公钥是否发布成功

```
gpg --keyserver hkp://pool.sks-keyservers.net --recv-keys 公钥ID
gpg --keyserver hkp://keyserver.ubuntu.com:11371 --recv-keys 公钥ID
```

使用 `gpg --list-keys` 命令查询配置好的公私钥信息

![image](https://user-images.githubusercontent.com/23411433/150723548-66173f98-6990-46c2-8be5-959467301513.png)


## 打包推送

使用`mvn clean deploy`进行编译部署，会将当前SNAPSHOT包推送到OSS 里面，注意这一步需要用到**Passphase**。

## 发布RELEASE

大致流程：

```
Maven发布 -> Sonatype Close  -> Sonatype Release
```

### Maven发布
关于如何利用maven进行发布的操作，这里就不详细介绍了，可以参考另一篇文档: [Maven发布RELEASE包到云效私有仓库并打TAG](https://mingshan.fun/2019/11/13/maven-release/)

主要有两步操作：`mvn release:prepare` 和 `mvn release:perform`。

### Sonatype Close

发布完正式RELEASE 后，此时会将正式包推送到 **staging Repositories**，进入[oss.sonatype](https://s01.oss.sonatype.org/#stagingRepositories)，选择你刚才发布的包，然后点击**Close** 按钮，会弹出确认框，如下所示：

![image](https://user-images.githubusercontent.com/23411433/150745859-598a7d06-f59e-478b-815d-18c5fef48062.png)

### Sonatype Release

Close 完之后，我们就可以进行Release了。点击**Release**按钮，需要等待一会，可以点击下边的Activity选项卡中查看状态。等待一会，会收到邮件，告诉你已经同步中央仓库了。

![image](https://user-images.githubusercontent.com/23411433/150746724-2631ecf4-2b4b-41dd-84a4-548c8d516de5.png)

![image](https://user-images.githubusercontent.com/23411433/150762858-305b1b13-b532-40c2-a25a-293e52880d8a.png)


至此，流程已经走完了。同步到中央仓库的时间比较漫长，耐心等待即可。

## 参考

- https://central.sonatype.org/publish/publish-maven/
- https://www.cnblogs.com/songyz/p/11387978.html
- https://blog.csdn.net/pdsu161530247/article/details/105429597
