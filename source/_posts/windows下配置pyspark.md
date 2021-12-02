---
title: windows下配置pyspark
date: 2021-12-02 18:58:21
tags: big data
---

# windows下配置pyspark

## 安装和配置spark

+ Spark版本: spark-3.0.3-bin-without-hadoop

+ Hadoop版本: hadoop-3.2.2
+ 下载winutils([链接](https://github.com/cdarlint/winutils)), 将winutils和hadoop.dll放入HADOOP_HOME/bin目录下

+ 在SPARK_HOME/conf下新建spark-env.cmd, 添加以下内容, 解决spark-shell启动时找不到log4j的错误

  1. 第1行: 关闭命令回显
  2. 第2行: 设置spark错误提示信息为中文

  2. 第3行: 设置spark寻找hadoop的jar包的路径

```cmd
@echo off
set JAVA_TOOL_OPTIONS="-Duser.language=en"
FOR /F %%i IN ('hadoop classpath') DO @set SPARK_DIST_CLASSPATH=%%i
```

## 解决vscode找不到pyspark或其他自定义包

+ 如果提示找不到py4j,可以把SPARK_HOME/python/lib/py4j-0.10.9-src.zip加入pythonpath中

+ 在.vscode的settings.json中, 添加以下内容, 解决terminal运行python程序时找不到自定义模块的问题

```json
"python.autoComplete.extraPaths": ["${workspaceFolder}", "${env:SPARK_HOME}/python"],
"terminal.integrated.env.windows": {
        "PYTHONPATH": "${env:SPARK_HOME}/python/lib/py4j-0.10.9-src.zip;${env:SPARK_HOME}/python;${workspaceFolder};${env:PYTHONPATH};",
        "SPARK_HOME":"${env:SPARK_HOME}"
}
```

+ 在.vscode的launch.json中, 添加以下内容, 解决调试模式找不到自定义模块的问题

```json
"env": {"PYTHONPATH":"${workspaceFolder};${env:SPARK_HOME}/python/lib/py4j-0.10.9-src.zip;${env:SPARK_HOME/python};${env:PYTHONPATH}",
                "SPARK_HOME": "${env:SPARK_HOME}"}
```

### 说明

 `python.autoComplete.extraPaths`指定vscode的python插件寻找其他包的位置, `terminal.integrated.env.windows`指定terminal的pythonpath环境变量, pycharm不会报找不到自定义包的原因是pycharm会自动把项目路径添加到pythonpath环境变量中, vscode则需要手动添加

