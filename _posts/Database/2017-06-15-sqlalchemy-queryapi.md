---
layout: post
#标题配置
title:  sqlalchemy中的QueryAPI使用
#时间配置
date:   2017-06-15 15:53:00 +0800
#大类配置
categories: Database
#小类配置
tag: sqlalchemy
---

* content
{:toc}

# 1.join(*props, **kwargs)
## 1.1 简单关系 joins
Consider a mapping between two classes User and Address, with a relationship User.addresses representing a collection of Address objectsassociated with each User. The most common usage of join() is to create a JOIN along this relationship, using the User.addresses attribute as an indicator for how this should occur:

    q = session.query(User).join(User.addresses)
Where above, the call to join() along User.addresses will result in SQL equivalent to:

    SELECT user.* FROM user JOIN address ON user.id = address.user_id

## 1.2 Joins to a Target Entity or Selectable
第二种join的形式，允许任何mapped entity作为target，在如此join中attempt to create a JOIN along the natural foreign key relationship between two entities

    q = session.query(User).join(Address)
如果上面的两个entities没有foreign keys或者有多个foreign keys将会出现error,join() is called upon to create the “on clause” automatically for us.

# 2.union(*q)
多个query之间的联合
    
    q1 = sess.query(SomeClass).filter(SomeClass.foo=='bar')
    q2 = sess.query(SomeClass).filter(SomeClass.bar=='foo')
    
    q3 = q1.union(q2)
通过嵌套的方式进行多个query之间的联合

    x.union(y).union(z).all()
等价于
    
    SELECT * FROM (SELECT * FROM (SELECT * FROM X UNION
                    SELECT * FROM y) UNION SELECT * FROM Z)
**对比join.all**
    
    x.union(y, z).all()
    
    SELECT * FROM (SELECT * FROM X UNION SELECT * FROM y UNION
                    SELECT * FROM Z)

Note that many database backends do not allow ORDER BY to be rendered on a query called within UNION, EXCEPT, etc. To disable all ORDER BY clauses including those configured on mappers, issue query.order_by(None) - the resulting Query object will not render ORDER BY within its SELECT statement.

