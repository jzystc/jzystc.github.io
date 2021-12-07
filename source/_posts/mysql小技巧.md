---
title: mysql 小技巧
date: 2021-12-03 17:12:00
categories:
    - mysql
tags:
---

# mysql 小技巧

## varchar类型的时间如何进行比较

使用str_to_date(str,format)函数, 该函数返回一个datetime值

假设有一张user表:

| id   | username | login_time |
| ---- | -------- | ---------- |
| 1    | aaa      | 2021-12-03 |
| 2    | bbb      | 2020-11-22 |
| 3    | ccc      | 2019-10-01 |
| 4    | ddd      | 2018-04-28 |

```mysql
# 方法一
select user,login_time from user where str_to_date(login_time,'%Y-%m-%d') between '2020-01-01' and '2020-12-31';
# 方法二
select user,login_time from user where str_to_date(login_time,'%Y-%m-%d') between str_to_date('2020-01-01','%Y-%m-%d') and str_to_date('2020-12-31','%Y-%m-%d');
```

