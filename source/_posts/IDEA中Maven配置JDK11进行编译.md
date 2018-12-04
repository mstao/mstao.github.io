---
title: IDEA中Maven配置JDK11进行编译
tags: [java]
author: Mingshan
categories: [Java]
date: 2018-10-13
---

前几天JDK11发布了不是，所以赶紧下载体验一番，我用的是IDEA编辑器(IntelliJ IDEA 2018.2.4 x64)，注意IDEA也要更新到最新版，用Maven编译的话需要进行相关配置，在此记录一下。

<!-- more -->

首先需要配置JDK的环境变量，配置好了显示如下：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/jdk11-version.png?raw=true)

然后进入到IDEA中，配置项目使用的JDK版本，依次点击`File -> Project Structure`，然后找到`Project SDK`选项，配置如下：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/jdk11-project-structure.png?raw=true)

接下来配置Maven的JDK版本，点击`File -> Settings -> Build, Execution, Deployment -> Build Tools -> Maven`，然后选择`Runner`，配置Maven使用的JDK版本，配置页面如下：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/jdk11-maven-settings.png?raw=true)

最后配置下maven的pom.xml文件，将目标编译版本改为JDK11，配置如下

```xml
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <dependencies>

    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
    </dependency>
  </dependencies>

  <build>
    <finalName>java11-tutorial</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>11</source>
          <target>11</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
```

刷新maven项目，试着编译下，可以看到已经使用JDK11编译了：

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/jdk11-build-output.png?raw=true)

看下编译后的字节码的大小版本号，利用HexPad编辑器打开class文件，如下。可以看到minor_version为 0x0000, major_version为0x0037，而0x0037的十进制表示为55，即用jdk11编译的，准确无误。

![image](https://github.com/ZZULI-TECH/interview/blob/master/images/jdk11-class-version.png?raw=true)