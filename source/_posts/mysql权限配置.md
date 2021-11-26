---
title: mysql 权限配置
date: 2021-11-18 13:55:00
categories:
    - mysql
tags:
---

# mysql 权限配置

## 删除root权限如何恢复

### 安全模式启动mysql

```bash
# 安全模式启动mysql服务
vim /etc/mysql/mysql.conf.d/mysqld.cnf
# 在[mysql]处添加下述代码
[mysql]
skip-grant-tables
# 重启mysql服务
sudo service mysql restart
```

### 恢复权限

```sql
# 报错使用flush privileges命令
mysql> grant all privileges on *.* to root@"localhost";
ERROR 1290 (HY000): The MySQL server is running with the --skip-grant-tables option so it cannot execute this statement
mysql> flush privileges;
# 重新赋权
mysql> grant all privileges on *.* to root@"localhost";
mysql> flush privileges;
```

### 还原配置文件

```bash
vim /etc/mysql/mysql.conf.d/mysqld.cnf

[mysql]
# skip-grant-tables

# 重启mysql服务
sudo service mysql restart
```

## 无密码登录问题

```sql
use mysql;

# mysql > 5.7
update user set password=password('PASSWORD') where user='root';

# mysql <= 5.7
update user set authentication_string=password("PASSWORD") where user='root';
# 更改密码认证方式
update user set plugin="mysql_native_password";

# 刷新
flush privileges;
```

## 权限管理常用命令

### 创建用户

```sql
CREATE USER 'user1'@'host' IDENTIFIED BY '123456';
CREATE USER 'user2'@'%' IDENTIFIED BY '';
GRANT SELECT, INSERT ON test.user TO 'user'@'%';
```

### 查看密码认证方式

```sql
SELECT user,host,plugin from mysql.user where user='root';
```

### 修改密码认证方式

```sql
alter user 'user'@'host' identified with mysql_native_password by 'pa';
flush privileges;
```

### 修改密码

```sql
use mysql;
# mysql version <= 5.7.5
SET PASSWORD FOR 'user'@'host' = PASSWORD('newpassword');
# mysql version > 5.7.5
SET PASSWORD FOR 'user'@'host' = PASSWORD('newpassword');
```

### 删除用户

```sql
DROP USER 'user'@'host';
```

### grant 赋权

```sql
# privileges可省略
mysql> grant all on *.* to 'user'@'ip' identified by "password";  
# 192.168.1.%表示一个网段
mysql> grant all privileges on *.* to user@'192.168.1.%' identified by "123456";
mysql> grant insert,select,update,delete,drop,create,alter on 'database'.'table' to user@'%' identified by "123456";
# 刷新
mysql> flush privilege
```

### revoke 撤销权限

```sql
mysql> revoke all on *.* from user@'ip';
mysql> revoke all privileges on *.* from user@'ip';         
mysql> revoke insert,select,update,delete,drop,create,alter on database.table from user@'%';
mysql> flush privileges;
```

### 查看权限

```bash
# 查看权限
show grants;
show grants for user@'%'; 
# 查看user和ip
SELECT User, Host, plugin FROM mysql.user;
```
