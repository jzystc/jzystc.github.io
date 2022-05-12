---
title: 解决Windows Tomcat的乱码问题
date: 2022-05-12 12:36:00
categories:
    - java
tags:
---

# 解决Windows Tomcat的乱码问题

## 常规解决方法

Tomcat9的log默认编码格式为utf-8, 而windows的cmd默认编码格式为gbk, 因此出现乱码问题
查阅网上资料, 解决方法主要有3种:

+ **修改cmd默认编码格式 (经测试无效)**
  
1. ctrl+r打开运行, 输入regedit
2. 定位到HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Command Processor
3. 添加字符串值, 名称为autorun, 值为chcp 65001

+ **修改tomcat控制台窗口的默认编码 (经测试有效)**
  
1. ctrl+r打开运行, 输入regedit
2. 定位到HKEY_CURRENT_USER\Console\Tomcat, 若没有则新建项
3. 添加DWORD (32位)值, 名称为CodePage, 选择十进制, 值为65001

+ **修改tomcat/conf/logging.properties (经测试有效, 但可能导致get/post调试参数出现乱码)**

1. 设置java.util.logging.ConsoleHandler.encoding = GBK

## 设置tomcat默认语言为英语

如果不需要中文提示, 也可以将tomcat默认语言改为英语, 方法如下: 

1. 打开tomcat/conf/bin/catalina.bat
2. 在setlocal这行代码之后添加rem set "CATALINA_OPTS=%CATALINA_OPTS% -Duser.language=en"

## IDEA控制台中文乱码

虽然tomcat启动时没有出现乱码, 但自己的代码通过idea控制台输出的中文依然可能出现乱码, 解决方法:

1. 点击Edit Configurations, 打开tomcat配置界面
2. 在vm options中添加-Dfile.encoding=UTF-8
