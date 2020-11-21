---
title: 新服务器迁移postgres数据库
date: 2020-11-21 09:39:33
tags: 
- 运维
- Postgres
---

课题组要部署一个云服务，之前用的是我个人申请的学生版服务器，价格优惠。快到期需要续费，但是优惠版的服务器不能够继续以优惠价格续费，正常的价格是一千多每年。所以我决定新申请一个优惠版服务器，然后迁移Postgres数据库和部署的web server。

基本步骤是购买新服务器（Ubuntu18.04版本），密码重设，为了方便登录也可以设置ssh免密码登录。

## 安装配置Postgres

我觉得Postgres比MySQL好的一点在于其开源。且能实现数据库的基本功能。

### 服务器安装Postgres

`sudo apt-get install postgresql postgresql-client `

可以在本机安装一个postgres-client，免得在两个服务器之间来回登录。

### 配置postgres数据库账号和远程连接

登录新服务器，设置linux中的postgres用户密码

`sudo passwd postgres`或者`sudo -i -u postgres`，免密码登录。

设置postgres 中postgres用户密码(以postgres用户登录)

`psql`进入数据库clinet软件

`postgres \password`设置数据库管理员postgres的密码

### 远程连接设置

修改Postgres远程连接允许 

`sudo vim /etc/postgres/10/main/postgres.conf`， 

修改`listen_addresses`那一行为`listen_addresses = '*'`（有的公司要求不允许数据库开放外网访问，请谨慎该选项）。

修改远程登录选项

`vim  /etc/postgres/10/main/pg_dba.conf`

添加新行

` host  all  all 0.0.0.0/0 md5` (不要使用trust，除非你想任何人能够访问你的数据库内容)

重启服务以应用刚刚的修改

`sudo service postgresql restart`

[参考](http://lazybios.com/2016/11/how-to-make-postgreSQL-can-be-accessed-from-remote-client/)

## 数据库迁移

Postgres 提供导出数据库数据和结构的程序。有两种方法，一种是在本机安装psql程序，新旧服务器都进行上面的远程连接设置，然后本机连接源服务器数据库dump备份文件，然后本机连接目标服务器数据库导入数据；另外一种是登录源服务器dump备份文件，再导出到目标服务器，在目标服务器执行导入操作。我使用的是比较麻烦的第二种。

### 备份数据库

`pg_dump -h (ip or localhost) -U postgres databasename > databasename.bak`

我是在服务器本地进行的操作，所以可以使用localhost。

### 恢复数据库

恢复数据库指令不会创建新的数据库，所以需要先行在新的数据库中创建数据库 `create database databasename;`

然后执行`psql -h localhost -U postgres -d databasename < databasename.bak`恢复数据库。

[参考](https://juejin.cn/post/6844904002409201671)