---
layout: post
#标题配置
title:  Sqlalchemy的安装使用
#时间配置
date:   2017-07-12 11:00:00 +0800
#大类配置
categories: Database
#小类配置
tag: Sqlalchemy
---

* content
{:toc}

# 1. Ubuntu下Mysql的安装
1. sudo apt-get install mysql-server （需要对root用户的密码进行设置）
2. sudo apt-get isntall mysql-client
3. sudo apt-get install libmysqlclient-dev\

检查是否成功安装<br/>
sudo netstat -tap | grep mysql （mysql的socket为Listen）

简单数据库命令<br/>
mysql -u root -p<br/>
show databases;<br/>
use database_name;<br/>
show tables;<br/>

简单测试数据生成<br/>
1. create database mytest;
2. create table products(id int(11) not null auto_increment, name varchar(50) not null, count int(11) not null default '0', primary key (id) );
3. insert products(name, count) values('Test1', 20);
4. insert products(name, count) values('Test2', 30);
5. select * from products;

[详细参考][1]

[1]: http://www.linuxidc.com/Linux/2016-07/133128.htm

# 2. Ubuntu下MySQL驱动的安装和sqlalchemy框架安装
1. sudo apt-get install python-mysqldb（python2.X需要安装该驱动，python3.x需要pymysql）
2. pip install SQLAlchemy

   查看是否安装好<br/>
   ```buildoutcfg
   python
   import sqlalchemy
   sqlalchemy.__version__
   ```
3. sudo apt-get install mysql-workbench(可视化工具)

[sqlalchemy操作数据库简单实例一][2]
[sqlalchemy操作数据库简单实例二][3]

[2]: http://blog.csdn.net/will130/article/details/48442699
[3]: http://blog.csdn.net/fgf00/article/details/52949973

# 3. 简单测试SQL and Sqlalchemy
# 3.1 Test SQL
参考代码（确保数据库mytest已存在）
```buildoutcfg
#!/usr/bin/python
# encoding: utf-8
import MySQLdb
# 打开数据库连接
conn = MySQLdb.connect(host="localhost", user="root", passwd="password", db="mytest")
# 使用cursor()方法获取操作游标
cursor = conn.cursor()

cursor.execute("DROP TABLE IF EXISTS user")
# 创建表
sql = """CREATE TABLE user (
         id varchar(20) primary key,
         name varchar(20) )"""
cursor.execute(sql)
conn.close()
```
# 3.2 Test SQLAlchemy
参考代码（确保user表已存在）
```buildoutcfg
#!/usr/bin python
#encoding:utf8
from sqlalchemy import Column, String, create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

# 数据表定义基类
Base = declarative_base()

class User(Base):
    # 表的名字:
    __tablename__ = 'user'

    # 表的结构:
    id = Column(String(20), primary_key=True)
    name = Column(String(20))

# 初始化数据库连接
engine = create_engine('mysql+mysqlconnector://root:password@localhost:3306/mytest', echo=True)
# 创建DBSession类型:
DBSession = sessionmaker(bind=engine)

# 创建session对象:
session = DBSession()
# 创建新User对象
new_user = User(id='8', name='Bob')
# 添加到session:
session.add(new_user)
# 提交即保存到数据库:
session.commit()
# 关闭session:
session.close()
```
输出log,create_engine为True时
```buildoutcfg
2017-07-12 17:36:47,002 INFO sqlalchemy.engine.base.Engine SHOW VARIABLES LIKE 'sql_mode'
2017-07-12 17:36:47,002 INFO sqlalchemy.engine.base.Engine {}
2017-07-12 17:36:47,006 INFO sqlalchemy.engine.base.Engine SELECT DATABASE()
2017-07-12 17:36:47,006 INFO sqlalchemy.engine.base.Engine {}
2017-07-12 17:36:47,010 INFO sqlalchemy.engine.base.Engine SELECT CAST('test plain returns' AS CHAR(60)) AS anon_1
2017-07-12 17:36:47,011 INFO sqlalchemy.engine.base.Engine {}
2017-07-12 17:36:47,013 INFO sqlalchemy.engine.base.Engine SELECT CAST('test unicode returns' AS CHAR(60)) AS anon_1
2017-07-12 17:36:47,014 INFO sqlalchemy.engine.base.Engine {}
2017-07-12 17:36:47,019 INFO sqlalchemy.engine.base.Engine BEGIN (implicit)
2017-07-12 17:36:47,022 INFO sqlalchemy.engine.base.Engine INSERT INTO user (id, name) VALUES (%(id)s, %(name)s)
2017-07-12 17:36:47,022 INFO sqlalchemy.engine.base.Engine {'id': '8', 'name': 'Bob'}
2017-07-12 17:36:47,024 INFO sqlalchemy.engine.base.Engine COMMIT

```