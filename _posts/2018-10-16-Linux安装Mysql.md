---
layout: post
title: "linux上安装mysql常见的一些问题"
date: 2018-10-16
description: "在linux(centos6.10)安装mysql5.6"
tag: 数据库
---   

# 关于在linux上安装mysql
## 这里主要说centos6.10 yum安装mysql 5.6，这是我在踩各种坑后整理出来，最简单最有效的安装方法了，在这里记录一下，方面日后用到
----------

### 一、检查系统是否安装其他版本的MYSQL数据：
```python
# yum list installed | grep mysql   检查是否已安装
# rpm -qa | grep mysql*	            检查是否已安装
# yum -y remove 文件名             （有就删）
```
### 二、安装mysql5.6前的工作和一些配置：
```oython
# wget http://repo.mysql.com/mysql-community-release-el6-5.noarch.rpm (下载mysql5.6的)
# mysql5.7的链接：https://dev.mysql.com/downloads/file/?id=470281
# 默认yum只能安装mysql 5.1，我们可以自己配置通过yum安装5.6或者其他版本
# yum repolist all | grep mysql     查看系统里面有没有mysql 的repo
# vim /etc/yum.repos.d/mysql-community.repo
# 在几个版本中只能有一个是enabled=1的，其他的都得enabled=0。（安装哪个版本，那个版本就设enabled=1）
# yum repolist enabled | grep mysql  再看看是否存在 MySQL 的 repo
```
### 二、开始安装mysql5.6：
```
# yum install mysql-community-server -y
```
看到如下画面：

那么就安装成功啦！，接下来开始配置数据库啦
### 三、设置开机启动：
```python
# chkconfig --list | grep mysqld
# 2、3、4、5都是on代表开机自动启动
# chkconfig mysqld on 即可
```
### 四、设置登录密码：
```python
# mysql_secure_installation     设置root密码
```

### 五、设置远程登录（方便我们在windows上连接linux上的mysql）
#### 1、登录mysql
```python
# mysql -uroot -p密码   登录mysql
```
#### 2、建立远程登录用户
```python
mysql> select user,host,password from mysql.user;  
create user 'root'@'%' identified by 'root';
mysql> set password for root=password('root');
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '你设置的密码' WITH GRANT OPTION;
mysql> flush privileges;
```
### 六、设置utf-8编码
```python
mysql> show variables like 'character%';  查看编码方式
# vi /etc/my.cnf 打开mysql的配置文件 添加和修改一些配置

[mysql]
default_character_set = utf8

[mysqld]
character-set-server=utf8
collation-server=utf8_general_ci
performance_schema_max_table_instances=400
table_definition_cache=400
table_open_cache=256

# 修改
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION

# service mysqld restart  重启mysql
# mysql> show variables like 'character%'; 再次查看编码
```
