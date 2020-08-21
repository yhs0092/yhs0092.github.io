---
title: 记录一个Spring ApplicationContext重复加载的问题
date: 2018-12-29 20:29:31
tags: [Spring]
keywords: [Spring]
categories:
- [软件技术]
- [踩坑]
---

Spring ApplicationContext重复加载可能造成各种奇怪问题，如无法自动注入Spring Bean、Spring Bean重复加载报错等等。仔细观察启动日志通常能从Spring应用上下文加载日志中看出一点端倪。
<!-- more -->
# 记录一个Spring ApplicationContext重复加载的问题

## 问题现象

有同事开发了一个demo服务，服务包含前端页面和ServiceComb开发的REST后端服务两部分。打成war包部署在Tomcat中，发现这个服务会在服务中心注册两个地址相同的实例。观察日志，发现Spring Context加载了两遍，并且Tomcat的webapps目录下存在一个与war包同名的目录和一个ROOT目录，两个目录中的内容是相同的。

## 问题原因

Tomcat在启动时会将webapps目录下的war包解压到一个同名目录下，将其作为一个context加载。而在Tomcat的`conf/server.xml`文件中，又额外定义了一个Context：
```xml
<Context path="/" reloadable="true" docBase="${war包的名字}"/>
```
于是该war包会作为root context 又被加载一次。

## 解决方案

删掉`conf/server.xml`文件中定义的context，把拷贝到webapps目录下的war包改名为`ROOT.war`。Tomcat启动时会自动将war包解压作为root context加载。
