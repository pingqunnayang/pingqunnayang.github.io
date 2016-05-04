---
layout: post
title:  异常收集录
titlename: 异常收集录
date:   2015-05-04 15:04:23 
category: 技术
tags: Error exception
description:
---
### 1.Nginx缓存文件夹proxy_temp问题

>**描述**：

<p style="text-indent: 2em">项目升级一个文件分享功能，之前是用H5分享，因为H5打印依赖浏览器导致打印的格式千奇百怪，所以本次升级采用PDF的格式分享，升级后在测试环境没有问题，但是上线后访问response一直未null，查看图文服务器也没有问题，后端也没看到异常日志。

>**解决步骤**：

排查后发现nginx有error日志，如下：

```java
2016/04/28 21:18:22 [crit] 18184#0: *67761916 open() "/usr/local/nginx-1.6.2/proxy_temp/3/33/0000494333" failed (13: Permission denied) while reading upstream, client: 202.85.216.162, server: static.zufangzi.com, request: "GET /GET/downloader/file/T233DE69AEE79451FAADC928AB73057C0.do HTTP/1.1", upstream: "http://*******:8080/GET/downloader/file/T233DE69AEE79451FAADC928AB73057C0.do",host:"static.******.com"
```
日志中发现是nginx的权限问题报错,让OP君帮忙修改了proxy_temp的权限，访问正常，谷歌后发现nginx有一项配置proxy_temp_file_write_size来设置缓存文件夹大小，当你的文件超过该参数设置的大小时，nginx会先将文件写入临时目录(缺省为nginx安装目下/proxy_temp目录)，op之前启动nginx用ruser，后面修改为suer启动，proxy_temp没有修改导致suer没有权限写入proxy_temp，再加上我们分享之前用H5格式文件比较小，升级后PDF文件比较大，所以抛错，`为什么抛错后前端返回状态还是200？还需要调研。`

### 2.tomcat启动项目报jdk版本错误

>**描述**：

<p style="text-indent: 2em">帮同事部署项目，从一台服务器转移到另一台服务器，把整个tomcat和项目都搬过来了，启动没有问题，但是访问发现报`Unsupported major.minor version 51.0`错误，查看两台机器jdk版本，都是1.7.0只是小版本不同。

>**解决步骤**：

<p style="text-indent: 2em">先同步了两台服务器的jdk版本，重启tomcat发现还是报相同的错误，猜想如果机器环境没有问题，那么就有可能是tomcat的问题，执行tomcat的下面命令：

```java
[liqianqian@IP-27-11 webapps]$ ../bin/configtest.sh 
Using CATALINA_BASE:   /home/liqianqian/android/apache-tomcat-7.0.55
Using CATALINA_HOME:   /home/liqianqian/android/apache-tomcat-7.0.55
Using CATALINA_TMPDIR: /home/liqianqian/android/apache-tomcat-7.0.55/temp
Using JRE_HOME:        /usr/local/jdk1.6.0_45/jre
Using CLASSPATH:       /home/liqianqian/android/apache-tomcat-7.0.55/bin/bootstrap.jar:/home/liqianqian/android/apache-tomcat-7.0.55/bin/tomcat-juli.jar
2016-5-4 16:12:07 org.apache.catalina.core.AprLifecycleListener init
信息: The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: /usr/local/jdk1.6.0_45/jre/lib/amd64/server:/usr/local/jdk1.6.0_45/jre/lib/amd64:/usr/local/jdk1.6.0_45/jre/../lib/amd64:/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2016-5-4 16:12:07 org.apache.coyote.AbstractProtocol init
信息: Initializing ProtocolHandler ["http-bio-8080"]
2016-5-4 16:12:07 org.apache.coyote.AbstractProtocol init
信息: Initializing ProtocolHandler ["ajp-bio-8009"]
2016-5-4 16:12:07 org.apache.catalina.startup.Catalina load
信息: Initialization processed in 584 ms
```
找到原因，发现tomcat使用`Using JRE_HOME:        /usr/local/jdk1.6.0_45/jre`这个配置，查看了tomcat的catalina.sh发现如下说明：

```xml
#JAVA_HOME       Must point at your Java Development Kit installation.
#                Required to run the with the "debug" argument.
#
#JRE_HOME        Must point at your Java Runtime installation.
#                Defaults to JAVA_HOME if empty. If JRE_HOME and      
#                JAVA_HOME are both set, JRE_HOME is used.
#                   
```
如果服务器配置了JAVA\_HOME和JRE\_HOME环境变量，优先使用JRE\_HOME，查看服务器发现新服务器的JRE_HOME指向jdk1.6，修改了tomcat的配置问题解决。