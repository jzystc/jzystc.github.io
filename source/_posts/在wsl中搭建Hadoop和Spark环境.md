---
title: 在wsl中搭建Hadoop和Spark环境
date: 2021-08-09 12:48:21
tags: big data
---
## 常见问题与注意事项

- wsl默认与windows共享环境变量, 如果在windows中已经配置过hadoop环境,在wsl中可能会出错,所以需要禁止wsl与windows共享环境变量

```bash
sudo vim /etc/wsl.conf
# 将下面的内容添加到wsl.conf中

[interop]
appendWindowsPath = false

#重启wsl, 在powershell中运行以下命令
wslconfig /t Ubuntu
```

## 配置JAVA环境

> 版本兼容性问题

1. hadoop 3.3及以上版本 支持java 8和java 11
2. hadoop 3.0.x - 3.2.x 支持java 8
3. hadoop 2.7.x - 2.10.x 支持java 7和java 8

### 下载openjdk 8

版本: adopt-openjdk-8u302
[下载地址](https://adoptium.net/?variant=openjdk8&jvmVariant=hotspot)

### 解压到指定目录

```bash
# 创建目录
mkdir /home/jzy/opt 
# -x 解压; -v 显示过程; -z 有gz属性的; -f 使用档案名
# --strip-components 数字 去除目录结构,为1表示解压第一个目录下的所有文件
sudo tar -xvzf --strip-components 1 OpenJDK8U-jdk_x64_linux_hotspot_8u302b08.tar.gz -C /home/jzy/opt/jdk8
```

### 配置Java环境变量

```bash
vim ~/.bashrc

# 文件末尾写入以下内容
export JAVA_HOME=/home/jzy/opt/jdk8
export PATH=$PATH:$JAVA_HOME/bin

# 应用设置
source ~/.bashrc

# 查看jdk版本
java -version
```

## 安装Hadoop

### 安装依赖包

```bash
sudo apt-get install ssh
sudo apt-get install pdsh
sudo apt-get install openssh-client openssh-server
```

- 下载Hadoop 2.10.1 ([下载地址](https://hadoop.apache.org/releases.html))

### 解压到指定目录

```bash
# -x 解压; -v 显示过程; -z 有gz属性的; -f 使用档案名
# --strip-components 数字 去除目录结构,为1表示解压第一个目录下的所有文件
tar -xvzf hadoop-2.10.1.tar.gz -C /home/jzy/opt
```

### 配置Hadoop环境变量

- 配置`hadoop-env.sh`
```
vim /home/jzy/opt/hadoop-2.10.1/etc/hadoop/hadoop-env.sh
# 空白位置加入以下内容
export JAVA_HOME=/home/jzy/opt/jdk8
```
- 配置`.bashrc`
  
```bash
vim ~/.bashrc
# 文件末尾加入以下内容
export HADOOP_HOME=/home/jzy/opt/hadoop-3.2.2
export PATH=$PATH:$HADOOP_HOME/bin

# 应用设置
source ~/.bashrc
# 查看版本
hadoop v

```

### 单机伪分布式部署hadoop

#### 配置hdfs

- 配置core-site.xml
``` bash
vim /home/jzy/opt/hadoop-2.10.1/etc/hadoop/core-site.xml
```
```xml
<configuration>
   <property>
      <name>fs.defaultFS</name>
      <value>hdfs://localhost:9000</value>
   </property>
   <property>
      <name>hadoop.tmp.dir</name>
      <value>/home/jzy/hadoop/tmp</value>
   </property>
</configuration>
```

- 配置hdfs-site.xml
```bash
vim /home/jzy/opt/hadoop-2.10.1/etc/hadoop/hdfs-site.xml
```
```xml
<configuration>
    <property>
         <name>dfs.replication</name>
         <value>1</value>
    </property>
    <property>
         <name>dfs.permissions</name>
         <value>false</value>
    </property>
    <property>
         <name>dfs.namenode.name.dir</name>
         <value>file:/home/jzy/hadoop/name</value>
    </property>
    <property>
         <name>dfs.datanode.data.dir</name>
         <value>file:/home/jzy/hadoop/data</value>
    </property>
</configuration>
```

#### 配置yarn
- mapred-site.xml
```bash
cd /home/jzy/opt/hadoop-2.10.1/etc/hadoop/
cp mapred-site.xml.template mapred-site.xml
vim mapred-site.xml
```
```xml
<configuration>
   <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
   </property>
</configuration>
```
- yarn-site.xml

```xml
<configuration>
<!-- Site specific YARN configuration properties -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

### 配置无密码登录
hadoop不提供输入密码的方式访问集群中的节点, 因此需要配置无密码登录


```bash
ssh-keygen –t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
ssh localhsot
```

> SSH常见问题
- `ssh localhost`无法连接
  - 尝试重启ssh服务
    ```bash
    sudo service ssh restart
    ```
  - 重装ssh服务
    ```bash
    sudo apt install --reinstall openssh-client
    sudo apt install --reinstall openssh-service
    ```

### 启动Hadoop
1. 格式化namenode
```bash
hdfs namenode -format
```

2. 启动hdfs: sbin/start-dfs.sh
```bash
cd /home/jzy/opt/hadoop-2.10.1
sbin/start-dfs.sh
```

3. 在浏览器里查看dfs: http://localhost:50070

4. 启动YARN
```bash
sbin/start-yarn.sh
```

5. 在浏览器里查看YARN: http://localhost:8088

6. `jps`命令检查启动是否成功
```
# 正常情况
872 Jps
746 SecondaryNameNode
522 DataNode
333 NameNode
# 启动了YARN
947 ResourceManager
1097 NodeManager
```
   
### 常见问题
- 排查故障
    1. 首先用jps确定哪个服务没有启动
    2. 然后根据服务名定位日志文件, 日志位于hadoop安装目录下的logs文件夹内
    ```
    cd /home/jzy/hadoop-2.10.1/logs
    cat 
    ```
- namenode未格式化成功,启动失败
  - 删除/home/hadoop/name目录重新格式化
    ``` bash
    sudo rm -rf /home/hadoop/name
    hdfs namenode -format
    ```

## 配置Scala
### 下载scala
scala 2.12.14与spark 3.1.2 兼容
[下载地址](https://www.scala-lang.org/download/2.12.14.html)
### 解压

```bash
tar -xvzf scala-2.12.14.tgz -C /home/jzy/opt/
```
### 环境变量
```bash
vim ~/.bashrc
# 在.bashrc中加入下方内容
export SCALA_HOME=/home/jzy/opt/scala-2.12.14
export PATH=$PATH:$SCALA_HOME/bin
```
## 配置Spark


### 下载Spark
选择版本 3.1.2 pre-built with user-provided hadoop
可与hadoop 2.10.1兼容
[下载地址](https://spark.apache.org/downloads.html)

### 解压
```bash
tar -xvzf spark-3.1.2-bin-without-hadoop.tgz -C /home/jzy/opt/
```

### 配置Spark与Hadoop关联

- conf/spark-env.sh
```bash
# 添加一行
export SPARK_DIST_CLASSPATH=$(hadoop classpath)
```

### mysql
1. 安装apt源
2. 安装mysql
3. 安装jdbc驱动
/usr/share/java/mysql-connector-java-8.0.26.jar


### HBase
- ~/.bashrc
```bash
export HBASE_HOME=/home/jzy/opt/hbase-2.4.5
export PATH=$PATH:$HBASE_HOME/bin
```

- conf/hbase-site.xml
```xml
<configuration>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>false</value>
  </property>
  <property>
    <name>hbase.tmp.dir</name>
    <value>/home/jzy/hbase/tmp</value>
  </property>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://localhost:9000/hbase</value>
  </property>
  <property>
    <name>hbase.unsafe.stream.capability.enforce</name>
    <value>false</value>
  </property>
</configuration>
```

### Hive
- ~/.bashrc
```bash
export HIVE_HOME=/home/jzy/opt/hive-2.4.9
export PATH=$PATH:$HIVE_HOME/bin
```

- conf/hive-site.xml
```xml
<configuration>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.cj.jdbc.Driver</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>0731</value>
  </property>
  <property>
    <name>hive.metastore.schema.verification</name>
    <value>false</value>
  </property>
</configuration>
```
