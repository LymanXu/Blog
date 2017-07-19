---
layout: post
#标题配置
title:  Python中SQLAlchemy的使用
#时间配置
date:   2017-07-19 11:00:00 +0800
#大类配置
categories: Database
#小类配置
tag: Mysqldb
---

* content
{:toc}

# 1. 相关异常
实验记录
```buildoutcfg
>>> from sqlalchemy import create_engine,exc
>>> addr = 'mysql+mysqlconnector://root:password@localhost:3306/mytest1'
>>> eng = create_engine(addr)
>>> print eng
Engine(mysql+mysqlconnector://root:***@localhost:3306/mytest1)

>>> from os.path import dirname
>>> eng2 = create_engine(dirname(addr)+'/mytest2')
>>> eng2
Engine(mysql+mysqlconnector://root:***@localhost:3306/mytest2)

>>> x = eng.connect()
>>> print x
<sqlalchemy.engine.base.Connection object at 0x7f3261b4d0d0>


>>> x2 = eng2.connect()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 2091, in connect
    return self._connection_cls(self, **kwargs)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 90, in __init__
    if connection is not None else engine.raw_connection()
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 2177, in raw_connection
    self.pool.unique_connection, _connection)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 2151, in _wrap_pool_connect
    e, dialect, self)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 1465, in _handle_dbapi_exception_noconnection
    exc_info
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/util/compat.py", line 203, in raise_from_cause
    reraise(type(exception), exception, tb=exc_tb, cause=cause)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 2147, in _wrap_pool_connect
    return fn()
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/pool.py", line 328, in unique_connection
    return _ConnectionFairy._checkout(self)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/pool.py", line 766, in _checkout
    fairy = _ConnectionRecord.checkout(pool)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/pool.py", line 516, in checkout
    rec = pool._do_get()
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/pool.py", line 1138, in _do_get
    self._dec_overflow()
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/util/langhelpers.py", line 66, in __exit__
    compat.reraise(exc_type, exc_value, exc_tb)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/pool.py", line 1135, in _do_get
    return self._create_connection()
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/pool.py", line 333, in _create_connection
    return _ConnectionRecord(self)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/pool.py", line 461, in __init__
    self.__connect(first_connect_check=True)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/pool.py", line 651, in __connect
    connection = pool._invoke_creator(self)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/engine/strategies.py", line 105, in connect
    return dialect.connect(*cargs, **cparams)
  File "/usr/local/lib/python2.7/dist-packages/sqlalchemy/engine/default.py", line 393, in connect
    return self.dbapi.connect(*cargs, **cparams)
  File "/usr/lib/python2.7/dist-packages/mysql/connector/__init__.py", line 162, in connect
    return MySQLConnection(*args, **kwargs)
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 129, in __init__
    self.connect(**kwargs)
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 454, in connect
    self._open_connection()
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 421, in _open_connection
    self._ssl)
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 204, in _do_auth
    self._auth_switch_request(username, password)
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 240, in _auth_switch_request
    raise errors.get_exception(packet)
sqlalchemy.exc.ProgrammingError: (mysql.connector.errors.ProgrammingError) 1049 (42000): Unknown database 'mytest2'

>>> addr
'mysql+mysqlconnector://root:password@localhost:3306/mytest1'
>>> eng3 = create_engine(dirname(addr))
>>> print eng3
Engine(mysql+mysqlconnector://root:***@localhost:3306)
```
在本地Mysql中存在数据库mytest1, 不存在数据库mytest2,可以看到在create_engine()时均正常，在connet()时连接数据库mytest2报错　exc.ProgrammingError:Unknown database 'mytest2'

**create_engine() and connect()**<br/>
create_engine()返回一个 Engine的实例，Engine代表和数据库交流的核心接口，适配底层数据库的方言

The first time a method like Engine.execute() or Engine.connect() is called,(第一次调用execute()或connect()时和数据库建立真正连接)
the Engine establishes a real DBAPI connection to the database, which is then used to emit the SQL. 

[**SQLAlchemy官方参考文档**](http://docs.sqlalchemy.org/en/latest/orm/tutorial.html)

# 2. 基本操作代码
```buildoutcfg
#!usr/bin/python

from os.path import dirname
from random import randrange as rand

from sqlalchemy import Column, Integer, String, create_engine, exc, orm
from sqlalchemy.ext.declarative import declarative_base

NUMBER_LENGHT = 20
Base = declarative_base()
DBNAME = 'mytest1'

FIELDS = ('id', 'name', 'projid')

tformat = lambda s: str(s).title().ljust(8)
cformat = lambda s: str(s).ljust(8)

NAMES = (
    (1, 'a', 1000), (2, 'b', 1000), (3, 'c', 2000), (4, 'd', 2000)
)


def rand_name():
    pick = set(NAMES)
    while pick:
        yield pick.pop() # 使用yield返回一个可迭代对象


class User(Base):
    __tablename__ = 'user'
    id = Column(Integer, primary_key=True)
    name = Column(String(NUMBER_LENGHT))
    projid = Column(Integer)

    def __str__(self):
        return ''.join(map(cformat, (self.id, self.name,self.projid)))


class SQLAlchemyTest(object):
    def __init__(self, dsn):
        try:
            eng = create_engine(dsn)
        except ImportError:
            raise RuntimeError()
        try:
            eng.connect()
        except exc.ProgrammingError: #　当不存在目标数据库时抛出ＰｒｏｇｒａｍｍｉｎｇＥｒｒｏｒ
            eng = create_engine(dirname(dsn))　＃ 对engine进行实例化时，可以精确到某个数据库，也可不确定具体数据库，执行新建数据库
            eng.execute('CREATE DATABASE %s' % DBNAME)　# 第一次调用execute或connect时才会和数据库建立真正的连接
            eng = create_engine(dsn)
        except Exception as e:
            print e

        Session = orm.sessionmaker(bind=eng)
        self.ses = Session()
        self.user = User.__tablename__
        self.eng = eng

    def insert(self):

        self.ses.add_all(
            User(id=aId, name=aName, projid=aProjid)
            for aId, aName, aProjid in rand_name()
        )

        self.ses.commit()

    def delete(self):

        rm = rand(1, 5) * 1000

        users = self.ses.query(User).filter_by(projid=rm).all()
        for i, user in enumerate(users):
            self.ses.delete(user)
        self.ses.commit()

    def update(self):

        fr = rand(1, 5) * 1000

        to = rand(1, 5)
        users = self.ses.query(User).filter_by(projid=fr).all()

        for i, user in enumerate(users):
            user.projid = to

        self.ses.commit()

    def select(self):

        print ('\n%s' % ''.join(map(tformat, FIELDS)))　# map([1],[2])对【２】中的每个值都执行【１】定义的函数操作，所以【２】必须为可迭代类型

        users = self.ses.query(User).all()
        for aUser in users:
            print aUser　# 需要对User中的__str__()进行复写，不然只输出对象地址


def main():
    dsn = 'mysql+mysqlconnector://root:password@localhost:3306/mytest1'

    try:
        orm = SQLAlchemyTest(dsn)
    except Exception as e:
        print e

    print 'create user table, drop old one'
    Base.metadata.create_all(orm.eng)

    orm.insert()
    orm.select()

    orm.update()
    orm.select()

    orm.delete()
    orm.select()

    orm.ses.close()


if __name__ == '__main__':
    main()

```

Python Console:
```buildoutcfg
create user table, drop old one

Id      Name    Projid  
1       a       1000    
2       b       1000    
3       c       2000    
4       d       2000    

Id      Name    Projid  
1       a       1000    
2       b       1000    
3       c       2000    
4       d       2000    

Id      Name    Projid  
3       c       2000    
4       d       2000    

```
