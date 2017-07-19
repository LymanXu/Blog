---
layout: post
#标题配置
title:  Python中Mysql数据库适配器mysqldb的使用
#时间配置
date:   2017-07-17 11:00:00 +0800
#大类配置
categories: Database
#小类配置
tag: Mysqldb
---

* content
{:toc}

# 1. 数据库连接的相关异常
**数据库密码错误权限问题**<br/>
Exception: OperationalError<br/>

数据类型<br/>
exception = {args:(tuple), message:string}

其中权限异常情况下 e 中args的值：(1045, "Access denied for user 'root'@'localhost' (using password: YES)")

**数据库不存在**<br/>
Exception: OperationalError<br/>
其中数据库不存在异常情况下 e 中args的值：(1049, "Unknown database 'mytest1'")

**代码记录**<br/>
```buildoutcfg
#!usr/bin/python

import MySQLdb as db

conn = None
try:
    conn = db.connect(host='localhost', user='root', passwd='password', db='mytest1')

except db.OperationalError as e:
    if 1045 == e.args[0]:
        print "\nAccess denied"
    elif 1049 == e.args[0]:
        conn = db.connect(host='localhost', user='root', passwd='password')
        conn.query('CREATE DATABASE mytest1')
        conn.commit()
        conn.close()
        conn = db.connect(host='localhost', user='root', passwd='password', db='mytest1')
    else:
        print e
except Exception as e:
    print e

finally:
    if conn:
        conn.close()
        print '\n close connection'

```

注：<br/>
lambda起到对一个简单逻辑的函数快速定义使用的作用<br/>
f = lambda x,y,z:x+y+z<br/>
f(1,2,3) = 4

ljust()返回一个源字符串的左对齐，并使用空格填充至指定的长度（也可以指定填充的字符），如果指定的长度小于原字符长度则直接返回原字符串<br/>
str.ljust(width[, fillchar])

map(func, seq1[, seq2,…]) <br/>
第一个参数接受一个函数名，后面的参数接受一个或多个可迭代的序列，返回的是一个集合。 <br/>
Python函数编程中的map()函数是将func作用于seq中的每一个元素，并将所有的调用的结果作为一个list返回。

# 2. 基本增删改查例子
```buildoutcfg
#!usr/bin/python

from random import randrange as rand

DBNAME = 'mytest1'
DB_USRR = 'root'
NAMELEN = 16
TRY_TIME = 3
SQL_db = None

RDBMSs = {'s': 'sqlite', 'm': 'mysql'}
FIELDS = ('id', 'name', 'projid')

tformat = lambda s: str(s).title().ljust(NAMELEN)
cformat = lambda s: str(s).ljust(NAMELEN)

def setup():
    return RDBMSs[raw_input('''
    Choose a database system:
    
    (S): sqlite
    (M): mysql
    Enter : '''
                            ).strip().lower()[0]]


def connect(db, DBNAME):
    global SQL_db
    conn = None

    if db == 'sqlite':
        print 'Database system is sqlite'

    elif db == 'mysql':
        import MySQLdb as SQL_db

        try:
            conn = SQL_db.connect(host='localhost', user='root', passwd='password', db='mytest1')

        except SQL_db.OperationalError as e:
            if 1045 == e.args[0]:
                print "\nAccess denied"
            elif 1049 == e.args[0]:
                conn = SQL_db.connect(host='localhost', user='root', passwd='password')
                conn.query('CREATE DATABASE mytest1')
                conn.commit()
                conn.close()
                conn = SQL_db.connect(host='localhost', user='root', passwd='password', db='mytest1')
            else:
                print e
        except Exception as e:
            print e

    else:
        return None

    print 'SQL.db:', SQL_db
    return conn


def create(cur, try_time=0):
    print 'SQL.db:', SQL_db
    if try_time < TRY_TIME:
        try:
            cur.execute('''
            CREATE TABLE user (
            id INT NOT NULL AUTO_INCREMENT,
            name VARCHAR(16) NULL,
            projid INT NULL,
            PRIMARY KEY (id))''')
        except SQL_db.OperationalError as e:
            try_time = try_time + 1
            drop(cur)
            create(cur, try_time)
    else:
        print 'try time > 3'

drop = lambda cur: cur.execute("DROP TABLE user")

NAMES = (
    (1, 'aaron', 1000),(2, 'blen', 1000),(3, 'canelk', 2000),(4, 'deandl', 2000)
)

def randName():
    pick = set(NAMES)
    while pick:
        yield pick.pop()

def insert(cur):
    cur.executemany("INSERT INTO user VALUES(%s,%s,%s)",
                    [(id, name, projid) for id, name, projid in randName()])

getRC = lambda cur: cur.rowcount if hasattr(cur, 'rowcount') else -1

def delete(cur, id):
    cur.execute('DELETE FROM user WHERE id=%d' % id)
    return getRC(cur)

def update(cur, id):
    to = rand(1,5)
    cur.execute('UPDATE user SET projid=%d WHERE id=%d' % (to, id))

def select(cur):
    cur.execute('SELECT * FROM user')
    print ('\n%s' % ''.join(map(cformat, FIELDS)))
    for data in cur.fetchall():
        print (''.join(map(tformat, data)))

def main():
    db = setup()
    conn = connect(db, DBNAME)
    cur = conn.cursor()
    create(cur)
    insert(cur)
    select(cur)

    update(cur, 1)
    select(cur)

    delete(cur, 1)
    select(cur)

    drop(cur)
    cur.close()
    conn.commit()
    conn.close()

if __name__ == '__main__':
    main()

```
结果：
```buildoutcfg
/usr/bin/python2.7 /home/lyman/PycharmProjects/LearnDatabase/src/use_mysqldb_1.py

    Choose a database system:
    
    (S): sqlite
    (M): mysql
    Enter : m
SQL.db: <module 'MySQLdb' from '/usr/lib/python2.7/dist-packages/MySQLdb/__init__.pyc'>
SQL.db: <module 'MySQLdb' from '/usr/lib/python2.7/dist-packages/MySQLdb/__init__.pyc'>

id              name            projid          
1               Aaron           1000            
2               Blen            1000            
3               Canelk          2000            
4               Deandl          2000            

id              name            projid          
1               Aaron           2               
2               Blen            1000            
3               Canelk          2000            
4               Deandl          2000            

id              name            projid          
2               Blen            1000            
3               Canelk          2000            
4               Deandl          2000            

Process finished with exit code 0

```
注：<br/>
对于main（）函数中的drop()注释掉后

1. 将main()中的commit()注释掉后，Python Control中的输出结果一样，不过数据库中user表中的记录为空（mytest1,user数据库－表存在）
2. 去掉注释使用commit()后，Python Control输出结果一样，数据库中user表中的数据存在

说明在没有commit()之前时，sql的相关记录应该存在内存中，通过commit刷新后持久化到数据库中（需要深入了解）