---
title: mysql 小技巧
date: 2021-12-03 17:12:00
categories:
    - mysql
tags:
---

# mysql 小技巧

## varchar类型的时间如何进行比较

使用str\_to\_date(str,format)函数, 该函数返回一个datetime值

假设有一张user表:

| id | username | login\_time |
| -- | -------- | ----------- |
| 1  | aaa      | 2021-12-03  |
| 2  | bbb      | 2020-11-22  |
| 3  | ccc      | 2019-10-01  |
| 4  | ddd      | 2018-04-28  |

```mysql
# 方法一
select user,login_time from user where str_to_date(login_time,'%Y-%m-%d') between '2020-01-01' and '2020-12-31';
# 方法二
select user,login_time from user where str_to_date(login_time,'%Y-%m-%d') between str_to_date('2020-01-01','%Y-%m-%d') and str_to_date('2020-12-31','%Y-%m-%d');
```

### 使用dbeaver数据库等工具时执行任务卡在0%
+ 远程登录mysql服务器并切断所有远程连接
```sql
# 列出所有连接
mysql> show full processlist;
# 生成kill语句
mysql> select concat('kill ', id, ';') from information_schema.processlist where command != 'Sleep' and Host!='localhost';
# kill 对应进程
mysql> kill pid
```

## pymysql

### 异常

1. `pymysql.err.interfaceerror: (0, '')`
+ 使用`pymysql.Connection.ping()`方法, 在执行sql之前测试连接并自动重连
  
    ```
    conn=pymysql.connect(...)
    conn.ping(reconnect=True)
    cursor = connection.cursor()
    cursor.execute(query)
    cursor.close()
    ```
### 数据库连接池
+ 引入需要的包
```python
import os
import pymysql
from dbutils.pooled_db import PooledDB
from pymysql.cursors import DictCursor
import configparser
import logging
logging.basicConfig(format="%(levelname)s:%(message)s", level=logging.WARNING)
```

+ Config 封装配置文件解析器
```python
class Config(configparser.ConfigParser):
    """
    Config().get_content("MYSQL")
    配置文件里面的参数
    [MYSQL]
    HOST = localhost
    PORT = 3306
    USER = root
    PASSWORD = 123456
    CHARSET = utf8
    """

    def __init__(self, config_filepath):
        super(Config, self).__init__()
        self.read(config_filepath)

    def get_sections(self):
        return self.sections()

    def get_options(self, section):
        return self.options(section)

    def get_content(self, section):
        result = {}
        for option in self.get_options(section):
            value = self.get(section, option)
            result[option] = int(value) if value.isdigit() else value
        return result

    def optionxform(self, optionstr):
    # 重写optionxform()防止key被自动转换成小写
        return optionstr
```

+ DBUtil 数据库连接池工具类
```python
class DBUtil:
    # 连接池对象
    __pool = None
    # 初始化
    def __init__(self, host, user, password, database, port=3306, charset="utf8"):
        self.host = host
        self.port = int(port)
        self.user = user
        self.password = str(password)
        self.database = database
        self.charset = charset
        self.conn = self.__get_connection()
        self.cursor = self.conn.cursor()

    def __get_connection(self):
        if DBUtil.__pool is None:
            __pool = PooledDB(
                creator=pymysql,
                mincached=1,
                maxcached=20,
                host=self.host,
                port=self.port,
                user=self.user,
                passwd=self.password,
                db=self.database,
                use_unicode=False,
                charset=self.charset,
            )
        return __pool.connection()

    def begin(self):
        """
        开启事务
        """
        self.conn.autocommit(0)

    def end(self, option="commit"):
        """
        结束事务
        """
        if option == "commit":
            self.conn.commit()
        else:
            self.conn.rollback()

    def connect(self):
        """
        直接连接数据库
        :return conn: pymysql连接
        """
        try:
            conn = pymysql.connect(
                host=self.host,
                user=self.user,
                password=self.password,
                database=self.database,
                port=self.port,
                use_unicode=True,
                charset=self.charset,
                cursor=DictCursor,
            )
            return conn
        except Exception as e:
            logging.error("Error connecting to mysql server.")
            raise

    # 关闭数据库连接
    def close(self):
        try:
            self.cursor.close()
            self.conn.close()
        except Exception as e:
            logging.error("Error closing connection to mysql server")

    # 查询操作，查询单条数据
    def get_one(self, sql):
        # res = None
        try:
            # self.connect()
            self.cursor.execute(sql)
            res = self.cursor.fetchone()
            self.close()
            return res
        except Exception as e:
            raise

    # 查询操作，查询多条数据
    def get_all(self, sql):
        try:
            # self.connect()
            self.cursor.execute(sql)
            res = self.cursor.fetchall()
            self.close()
            return res
        except Exception as e:
            raise

    # 查询数据库对象
    def get_all_obj(self, sql, table_name, *args):
        resList = []
        fields_list = []
        try:
            if len(args) > 0:
                for item in args:
                    fields_list.append(item)
            else:
                fields_sql = (
                    "select COLUMN_NAME from information_schema.COLUMNS where table_name = '%s' and table_schema = '%s'"
                    % (table_name, self.conn_name)
                )
                fields = self.get_all(fields_sql)
                for item in fields:
                    fields_list.append(item[0])

            # 执行查询数据sql
            res = self.get_all(sql)
            for item in res:
                obj = {}
                count = 0
                for x in item:
                    obj[fields_list[count]] = x
                    count += 1
                resList.append(obj)
            return resList
        except Exception as e:
            raise

    def insert(self, sql, params=None):
        """
        插入操作
        :return count: 影响的行数
        """
        return self.__edit(sql, params)

    def update(self, sql, params=None):
        """
        更新操作
        :return count: 影响的行数
        """
        return self.__edit(sql, params)

    def delete(self, sql, params=None):
        """
        删除操作
        :return count: 影响的行数
        """
        return self.__edit(sql, params)

    def __edit(self, sql, params=None):
        max_attempts = 3
        attempt = 0
        count = 0
        while attempt < max_attempts:
            try:
                self.conn = self.__get_connection()
                self.cursor = self.conn.cursor()
                if params is None:
                    count = self.cursor.execute(sql)
                else:
                    count = self.cursor.execute(sql, params)
                self.conn.commit()
                self.close()
            except Exception as e:
                logging.error(e)
                self.conn.rollback()
                count = 0
        return count

    def execute(self, sql, params=None):
        max_attempts = 3
        attempt = 0
        while attempt < max_attempts:
            try:
                self.conn = self.__get_connection()
                self.cursor = self.conn.cursor()
                if params is None:
                    result = self.cursor.execute(sql)
                else:
                    result = self.cursor.execute(sql, params)
                self.conn.commit()
                self.close()
                return result
            except Exception as e:
                attempt += 1
                logging.warning(f"retry: {attempt}")
                logging.exception(e)

    def truncate(self, table):
        sql = f"truncate table {table}"
        self.execute(sql)

    def executemany(self, sql, data):
        max_attempts = 3
        attempt = 0
        while attempt < max_attempts:
            try:
                self.conn = self.__get_connection()
                self.cursor = self.conn.cursor()
                result = self.cursor.executemany(sql, data)
                self.conn.commit()
                self.close()
                return result
            except Exception as e:
                attempt += 1
                logging.warning(f"retry: {attempt}")
                logging.exception(e)


config_filepath = os.path.dirname(os.path.dirname(__file__)) + "/config/dev.ini"
config = Config(config_filepath).get_content("MYSQL")
conn = DBUtil(
    host=config["HOST"],
    port=int(config["PORT"]),
    database=config["DATABASE"],
    user=config["USER"],
    password=config["PASSWORD"],
    charset=config["CHARSET"],
)


def get_connection():
    return conn


if __name__ == "__main__":
    db = get_connection()
    # 使用 cursor() 方法创建一个游标对象 cursor
    cursor = db.cursor

    # 使用 execute()  方法执行 SQL 查询
    cursor.execute("SELECT VERSION()")

    # 使用 fetchone() 方法获取单条数据.
    data = cursor.fetchone()
    print("Database version : %s " % data)

    # 关闭数据库连接
    db.close()

```