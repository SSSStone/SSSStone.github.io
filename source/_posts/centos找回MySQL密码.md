---
title: centos找回MySQL密码
date: 2017-03-31 19:34:18
tags: 
- MySql
categories: 
- 杂记
- 工具
---

1. 停止mysqld服务

```text
/etc/init.d/mysqld stop
```

<!-- more -->

2. 以不检查权限的方式启动

```text
cd /usr/local/mysql/bin
./mysqld_safe --skip-grant-tables & 或者 mysqld --skip-grant-tables &
```

3. 用空密码方式使用root用户登录MySQL

```text
./mysql -uroot -p
```

4. 修改root用户的密码

```text
update mysql.user set password=PASSWORD('newpass') where User='root';
flushprivileges;
quit
```

5. 重新启动MySQL

`/etc/init.d/mysqld restart`
