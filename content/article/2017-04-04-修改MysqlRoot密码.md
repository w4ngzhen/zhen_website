---
layout: post
title: 修改MySql Root密码（包含忘记密码的方式）
date: 2017-04-04
tags: 
- Mysql
categories: 
- 技术
---
曾几何时，我也是记得MySQL root密码的人，想要修改root密码还不是轻而易举的事？下面前三种修改改方式都是在记得密码的情况下进行修改，如果你忘记了原本的root，请直接跳至 **终极**

<!-- more -->

**第一种：**
在MySQL中修改：mysql> set password for root@localhost = password(‘新密码’);
当然，你也可以在root账户下去修改其他账户的密码，只需要将root换为其他账户即可
（注意：后面的localhost是指只能在本地登陆的账户，在修改其他账户密码时一定要对应其可登录范围修改@后面的字段属性）

**第二种：**
直接进入mysql数据库中，修改user表中的root的password。
mysql> use mysql;
mysql> update user set password = password(‘新密码’) where user = ‘root’ and host = ‘localhost’;
(注意：这个host后面的东西的意义和上面一样)
mysql> flush privileges; （记得刷新权限）

**第三种：**
不要忘了mysqladmin
```mysqladmin -u root -p 123456 password 123``` 就行了

然而重点来了，在以上的几种方法，都是针对于我们还记得root用户密码。可是一开始就忘了root密码了怎么办？
**终极：**
1．首先确认服务器出于安全的状态，也就是没有人能够任意地连接MySQL数据库。

2．修改MySQL的登录设置：
```# vi /etc/my.cnf```

在[mysqld]的段中加上一句：skip-grant-tables （这一句话表示，绕过所有的用户权限）
例如：
```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
skip-grant-tables
```
保存并且退出vi。

3．重新启动mysqld
```service mysqld restart```
好了，在此基础上，你就可以直接mysql进入数据库了

4．登录并修改MySQL的root密码
```
mysql
mysql> USE mysql ;
mysql> UPDATE user SET Password = password ( ‘新密码’ ) WHERE User = ‘root’ ;
mysql> flush privileges ;
mysql> quit
Bye
```
5．将MySQL的登录设置修改回来
```# vi /etc/my.cnf```

将刚才在[mysqld]的段中加上的skip-grant-tables删除
保存并且退出vi。

6．重新启动mysqld
```service mysqld restart```

好了，重新使用新密码的root账户吧