---
layout: post
title:  "[SQLAlchemy] SQL Expression Language Tutorial"
date:   2015-01-10 20:00:00
comments: true
categories: [SQLAlchemy]
tags: [SQLAlchemy, Python]
---

前言
====

要學習 Python 操作資料庫，當然以 SQLAlchemy 作為首選了....相較於 MS.NET，SQLAlchemy 就類似於 ADO.NET(database toolkit) & Entity Framework(object-relational mapping system) 所提供的功能。

![SQLAlchemy layer diagram](http://aosabook.org/images/sqlalchemy/layers.png)

它包含了 `Core` & `ORM` 兩個部分，其中 Core 主要是處理資料庫的表格建立、查詢、更新、刪除等工作，若是要處理較為複雜的表格關聯或是 class 的關聯，則就交由 ORM 來處理。

最底層直接與資料庫操作的部分並沒有包含在 SQLAlchemy 內，而是與其他實作 Python Database API(DBAPI) 的套件進行整合，SQLAlchemy 所提供的是統一對資料庫的操作方式 & ORM 的相關功能，針對比較常見的資料庫支援，可點選下列連結：

- [Microsoft SQL Server](http://docs.sqlalchemy.org/en/rel_0_9/dialects/mssql.html)

- [MySQL](http://docs.sqlalchemy.org/en/rel_0_9/dialects/mysql.html)

- [PostgreSQL](http://docs.sqlalchemy.org/en/latest/dialects/postgresql.html)


套件安裝
=======

Python 中可用來連接 SQL Server 的套件也不少，除了下面會介紹到的 [pymssql][1] 外，還有 [PyODBC][2]、[AdoDBAPI][3] ... 等等，我這邊選用 [pymssql][1] 作為連接 SQL Server 的套件。

因為在過程中所要連結的資料庫為 Microsoft SQL Server，所以要安裝 [pymssql][1] & [SQLAlchemy][4] 兩個套件。

目前環境所使用的套件版本分別是：

- pymssql 2.1.1 

- SQLAlchemy 0.9.8


版本檢查
=======

透過以下檢查，可以得知我們所使用的版本是 0.9.8

``` python
In[2]: import sqlalchemy
In[3]: sqlalchemy.__version__
Out[3]: '0.9.8'
```

連接 SQL Server 資料庫
=====================

要連接資料庫，必須要透過 create_engine 方法產生 Engine 物件來進行連結，而 Engine 的角色則是作為 SQLAlchemy 與 DBAPI 之間的橋梁，也因此開發者就不需要直接呼叫 DBAPI，而都是透過 Engine 來達成。

``` python
# -*- coding: utf-8 -*-

# 引用 create_engine
from sqlalchemy import create_engine

# 建立 engine 並連線至 SQL Server
db_usr = 'YOUR_DB_USER_ACCOUNT'
db_pwd = 'YOUR_DB_USER_PASSWORD'
db_server = 'YOUR_DB_SERVER'
db_name = 'YOUR_DB_NAME'
engine = create_engine("mssql+pymssql://%s:%s@%s/%s?charset=utf8"
                       % (db_usr, db_pwd, db_server, db_name), echo=True)
conn = engine.connect()
conn.close()            
```


定義並建立 Table
===============

若要建立 Table，則必須引用 MetaData / Table / Column ... 等 class 來處理，以下為範例：

``` python
from sqlalchemy import Table, Column, Sequence, String, Integer, Unicode, MetaData, ForeignKey

# 使用 MetaData 作為 catalog
metadata = MetaData()

# 定義 Table 內容
users = Table('users', metadata,
               Column('id', Integer, Sequence('user_id_seq'), primary_key=True),
               Column('name', Unicode(64)),     # 若要存中文難字則要使用 Unicode 而非 String
               Column('fullname', Unicode(128)), )

addresses = Table('addresses', metadata,
                   Column('id', Integer, primary_key=True),
                   Column('user_id', None, ForeignKey('users.id')),
                   Column('email_address', String, nullable=False), )

# 透過 engine 在資料庫中建立 Table
metadata.create_all(engine)
```

在 SQLAlchemy 中，資料庫的特性都被定義成一個個的類別，並會以物件的形式存在；而 MetaData 就是個用來存放並關聯不同資料庫特性物件的容器。

> 以上的範例就是定義了 Table & Column & Type 相關的物件，並將其存放於 MetaData 中，並透過 MetaData 在資料庫中產生實際的 Table。

以上關於 Table 的建立只是一個簡單的範例，若想要詳細了解關於如何在 MetaData 中定義資料庫相關物件，可參考 [Describing Databases with MetaData](http://docs.sqlalchemy.org/en/rel_0_9/core/metadata.html) 這一篇文章。


Insert
======

首先先看看以下這張圖，介紹 SQLAlchemy 與 DBAPI 是如何互動的：

![Engine, Connection, ResultProxy API](http://aosabook.org/images/sqlalchemy/engine.png)

從上圖可以看出，其實之前提到的 Engine 並非實際執行 SQL 命令的物件，而是必須由 Engine 產生一個 Connection 物件並連接到資料庫後，由 Connection 來執行 SQL 命令。

### 新增單筆資料

以下為 insert 的範例：

``` python
# 建立資料庫連線
conn = engine.connect()

# insert 功能
ins = users.insert().values(name='阿堃', fullname='堃喆 unicode 測試')
# 顯示經過 SQLAlchemy compile 後的 insert 語法
print(str(ins))
# 顯示每個 column 所綁定的 value
print(ins.compile().params)
# 顯示存入資料的 primary key
print(result.inserted_primary_key)

# 另一種 insert 的方式
ins = users.insert()
conn.execute(ins, id=2, name='麥可', fullname='麥可菲爾普斯')

# 關閉資料庫連線
conn.close()
```

### 新增多筆資料

``` python
conn = engine.connect()

# 同時新增多筆資料
conn.execute(addresses.insert(), [
    {'user_id': 1, 'email_address': 'user1@test.com'},
    {'user_id': 1, 'email_address': 'user2@test.com'},
    {'user_id': 2, 'email_address': 'user3@test.com'},
    {'user_id': 2, 'email_address': 'user4@test.com'},
    {'user_id': 1, 'email_address': 'user5@test.com'},
])

conn.close()
```


Select
======

SQLAlchemy 會將我們所定義的物件 & 相關的操作 compile 成 SQL 語法後執行，以下我們用範例來說明 select 的用法：

``` python
# -*- coding: utf-8 -*-

from sqlalchemy import create_engine, Table, Column, Sequence, String, Integer, Unicode, MetaData, ForeignKey
from sqlalchemy.sql import select

# 建立 engine 並連線至 SQL Server
db_usr = 'YOUR_DB_USER_ACCOUNT'
db_pwd = 'YOUR_DB_USER_PASSWORD'
db_server = 'YOUR_DB_SERVER'
db_name = 'YOUR_DB_NAME'
engine = create_engine("mssql+pymssql://%s:%s@%s/%s?charset=utf8"
                       % (db_usr, db_pwd, db_server, db_name), echo=True)


metadata = MetaData()

users = Table('users', metadata,
               Column('id', Integer, Sequence('user_id_seq'), primary_key=True),
               Column('name', Unicode(64)),
               Column('fullname', Unicode(128)), )

addresses = Table('addresses', metadata,
                   Column('id', Integer, primary_key=True),
                   Column('user_id', None, ForeignKey('users.id')),
                   Column('email_address', String, nullable=False), )

conn = engine.connect()

# 顯示 SQLAlchemy compile 後的 select 語法
s = select([users])
print(s)

# 查詢資料，顯示所有資料
result = conn.execute(s)
for row in result:
    print(row)
result.close()  # 關閉 ResultProxy

# 查詢資料，顯示第一筆資料
result = conn.execute(s)
row = result.fetchone()
print('name: %s, fullname: %s' % (row['name'], row['fullname']))

# 查詢資料，顯示第二筆資料
row = result.fetchone()
print('name: %s, fullname: %s' % (row[1], row[2]))
result.close()

# 查詢特定欄位資料
s = select([users.c.name, users.c.fullname])
for row in conn.execute(s):
    print(row)

# 查詢兩個 Table join 後的資料
print(str(users.c.id == addresses.c.user_id))   # users.id = addresses.user_id
s = select([users, addresses]).where(users.c.id == addresses.c.user_id)
for row in conn.execute(s):
    print(row)

conn.close()
```

### Operators 

從上面範例中的 where() 方法可以得知，在 where() 方法內的參數會被 compile 成標準的 SQL 語法後傳到資料庫進行處理，以下介紹幾個不同的寫法：

``` python
# users.id = :id_1
print(users.c.id == 7)

# users.id != :id_1
print(users.c.id != 7)

# users.name < :name_1
print(users.c.name < 'fred')

# users.id + addresses.id
print(users.c.id + addresses.c.id)  # 兩個 column 的資料型態皆為 Integer

# users.name || users.fullname
print(users.c.name + users.c.fullname)
```

此外，透過 [and_()](http://docs.sqlalchemy.org/en/rel_0_9/core/sqlelement.html#sqlalchemy.sql.expression.and_)、[or_()](http://docs.sqlalchemy.org/en/rel_0_9/core/sqlelement.html#sqlalchemy.sql.expression.or_)、[not_()](http://docs.sqlalchemy.org/en/rel_0_9/core/sqlelement.html#sqlalchemy.sql.expression.not_) 還可以將多個條件組合起來：

``` python
# users.name LIKE :name_1 AND users.id = addresses.user_id
#   AND (addresses.email_address = :email_address_1 OR addresses.email_address = :email_address_2)
#   AND users.id <= :id_1
print(and_(
        users.c.name.like('%堃%'),
        users.c.id == addresses.c.user_id,
      or_(
        addresses.c.email_address == 'user1@gmail.com',
        addresses.c.email_address == 'user2@gmail.com'
      ),
      not_(users.c.id > 5)))

# 也可以用 & | ~，結果與上面相同
print(users.c.name.like('%堃') & (users.c.id == addresses.c.user_id) &
      ((addresses.c.email_address == 'user1@gmail.com') | (addresses.c.email_address == 'user2@gmail.com'))
        & ~(users.c.id > 5))
```

完整的 SQL 查詢字串範例：

``` python
# SELECT users.fullname || :fullname_1 || addresses.email_address AS title
# FROM users, addresses
# WHERE users.id = addresses.user_id AND users.name BETWEEN :name_1 AND :name_2
#   AND (addresses.email_address LIKE :email_address_1 OR addresses.email_address LIKE :email_address_2)
s = select([(users.c.fullname + ", " + addresses.c.email_address).label('title')])\
        .where(users.c.id == addresses.c.user_id)\
        .where(users.c.name.like('%堃%'))\
        .where(or_(addresses.c.email_address.like('%user1%'), addresses.c.email_address.like('%user2%')))
print(str(s))
print(conn.execute(s).fetchall())
```

使用 Text 方式指定 SQL 語法
==========================

看到上面的語法，說真的很不 friendly 啦.....那就改用純 text 試試看：

``` python
s = text("SELECT users.fullname + ', ' + addresses.email_address AS title "
         "FROM users, addresses "
         "WHERE users.id = addresses.user_id "
         "  AND users.name LIKE :usr "
         "  AND (addresses.email_address LIKE :e1 OR addresses.email_address LIKE :e2)")
# 指定 usr, e1, e2 三個參數
print(conn.execute(s, usr='%麥%', e1='%test.com%', e2='user1%').fetchall())
```

看了上面這段程式，也許有人會有疑惑，用 `text()` 來下語法就好啦! 為什麼還要跟前面一樣搞得這麼複雜?

主要還是因為，在不同資料庫間，語法也是有些許的不同，以上面的例子來說，連接字串的時候在 SQL Server & MySQL 中使用的是 `+`，但到了 PostgreSQL 就改用 `||` 啦!

所以為了讓程式可以跨資料庫執行，透過 `select()`、`where()`、`and_()`、`or_()`、`not_()` 來組成 SQL 指令，會是比較不容易遇到平台限制的做法。

> 【註】所以 text() 使用越多，跨資料庫的相容性就越低......


提高 SQL 語法的相容性
====================

為了提高 SQL 語法的相容性，除了儘量少用 text() 之外，就是多用 SQLAlchemy 所提供的方法囉，調整原本的 SQL 語法如下：

### 使用 literal_column

``` python
s = select([literal_column('users.fullname') + ', ' + literal_column('addresses.email_address').label('title')]) \
    .where(
        and_(
            literal_column('users.id') == literal_column('addresses.user_id'),
            text("users.name like :usr"),
            text("(addresses.email_address like :e1 OR addresses.email_address like :e2)")
        )
    ) \
    .select_from(table('users')) \
    .select_from(table('addresses'))
```

### 使用 group_by & order_by

若要加上 group_by or order_by，可以參考以下寫法：

``` python
# SELECT addresses.user_id, count(addresses.id) AS num_addresses
# FROM addresses 
# Group By user_id
# ORDER BY num_addresses
stmt = select([addresses.c.user_id,
               func.count(addresses.c.id).label('num_addresses')])\
    .group_by(addresses.c.user_id)\
    .order_by("num_addresses")
```


使用 Aliases
============

當每個 Table 的名稱都很長時，適時的使用 alias 的功能，可以讓 SQL 語法看起來更為簡潔：

``` python
# SELECT users.id, users.name, users.fullname 
# FROM users, addresses AS a1, addresses AS addresses_1 
# WHERE users.id = a1.user_id AND users.id = addresses_1.user_id 
# 	AND a1.email_address = 'user1@test.com' 
#		AND addresses_1.email_address = 'user5@test.com'
a1 = addresses.alias('a1')
a2 = addresses.alias()
s = select([users]).where(and_(users.c.id == a1.c.user_id,
                               users.c.id == a2.c.user_id,
                               a1.c.email_address == 'user1@test.com',
                               a2.c.email_address == 'user5@test.com'))
```


從現有資料庫中取得 Table 定義
===========================

執行上面的程式時，都必須將 table 的定義先放在前面，不然關於欄位資訊、table 之間的關聯都無法正常運作，以上這是當資料庫原本沒有 table 資訊存在時，如此寫法可以新增 table 也可以將其資訊拿來使用。

但如果資料庫已經有 table 資訊存了呢? 可以使用以下方式直接將其從資料庫取出並使用：

``` python
# -*- coding: utf-8 -*-
from sqlalchemy import create_engine, Table, MetaData

# 建立 engine 並連線至 SQL Server
db_usr = 'YOUR_DB_USER_ACCOUNT'
db_pwd = 'YOUR_DB_USER_PASSWORD'
db_server = 'YOUR_DB_SERVER'
db_name = 'YOUR_DB_NAME'
engine = create_engine("mssql+pymssql://%s:%s@%s/%s?charset=utf8"
                       % (db_usr, db_pwd, db_server, db_name), echo=True)

meta = MetaData()

users = Table('users', meta, autoload=True, autoload_with=engine)
addresses = Table('addresses', meta, autoload=True, autoload_with=engine)
```

如此一來就不需要辛苦將所有 table 都定義在程式之前了!


其他用法
=======

以下介紹更多一些 SQLAlchemy Core 的相關用法：

### bindparam

使用 `bindparam` 用來將參數綁定到 SQL 語法中：

``` python
# SELECT users.id, users.name, users.fullname 
# FROM users 
# WHERE users.name = 'godleon'
2015-01-10 07:27:05,959 INFO sqlalchemy.engine.base.Engine {'username': 'godleon'}
s = users.select(users.c.name == bindparam('username'))
print(conn.execute(s, username='godleon').fetchall())
```

### Like()

``` python
# SELECT users.id, users.name, users.fullname 
# FROM users 
# WHERE users.name LIKE ('%odl%')
s = users.select(users.c.name.like(text("'%'") + bindparam('username', type_=String) + text("'%'")))
print(conn.execute(s, username='odl').fetchall())
```

### Left Outer Join

``` python
# SELECT users.id, users.name, users.fullname, addresses.id, addresses.user_id, addresses.email_address 
# FROM users LEFT OUTER JOIN addresses ON users.id = addresses.user_id 
# WHERE users.name LIKE (%(name)s + '%') OR addresses.email_address LIKE 'godleon%' 
# ORDER BY addresses.id
s = select([users, addresses])\
    .where(or_(
        users.c.name.like(bindparam('name', type_=String) + text("'%'")),
        addresses.c.email_address.like(bindparam('name', type_=String) + text("'%'"))
            )
    ).select_from(users.outerjoin(addresses)).order_by(addresses.c.id)
    print(conn.execute(s, name='godleon').fetchall())
```

### Max() & Scalar()

``` python
# SELECT max(addresses.email_address) AS [maxEMail] 
# FROM addresses
s = select([func.max(addresses.c.email_address, type_=String).label('maxEMail')])
print(conn.execute(s).scalar())
```

### row_number() over

``` python
# SELECT users.id, row_number() OVER (ORDER BY users.name) AS anon_1 
# FROM users
s = select([users.c.id, func.row_number().over(order_by=users.c.name)])
print(conn.execute(s).fetchall())
```

### Union()

``` python
# SELECT addresses.id, addresses.user_id, addresses.email_address 
# FROM addresses 
# WHERE addresses.email_address = 'godleon@gmail.com'
# UNION 
# SELECT addresses.id, addresses.user_id, addresses.email_address 
# FROM addresses 
# WHERE addresses.email_address = 'user1@test.com' 
# ORDER BY addresses.email_address
u = union(addresses.select().where(addresses.c.email_address == 'godleon@gmail.com'),
          addresses.select().where(addresses.c.email_address == 'user1@test.com'))\
    .order_by(addresses.c.email_address)
print(conn.execute(u).fetchall())
```

### Except()

``` python
# SELECT addresses.id, addresses.user_id, addresses.email_address 
# FROM addresses 
# WHERE addresses.email_address LIKE '%@test.com' 
# EXCEPT 
# SELECT addresses.id, addresses.user_id, addresses.email_address 
# FROM addresses 
# WHERE addresses.email_address LIKE '%user4%'
u = except_(addresses.select().where(addresses.c.email_address.like('%@test.com')),
            addresses.select().where(addresses.c.email_address.like('%user4%')))

print(conn.execute(u).fetchall())
```

### Sub Query

``` python
# SELECT users.name 
# FROM users 
# WHERE users.id = (SELECT users.id 
#										FROM users 
#										WHERE users.name = 'godleon')
stmt = select([users.c.id]).where(users.c.name == 'godleon').correlate(None)
s = select([users.c.name]).where(users.c.id == stmt)
print(conn.execute(s).fetchall())
```

### Join、Group By

``` python
# SELECT users.name, count(addresses.id) AS count_1
# FROM users JOIN addresses ON users.id = addresses.user_id
# GROUP BY users.name
stmt = select([users.c.name, func.count(addresses.c.id)])\
    .select_from(users.join(addresses))\
    .group_by(users.c.name)
print(conn.execute(stmt).fetchall())
```

### Distinct

``` python
# SELECT DISTINCT users.name
# FROM users, addresses
# WHERE (addresses.email_address LIKE '%%' + users.name + '%%')
stmt = select([users.c.name])\
    .where(addresses.c.email_address.contains(users.c.name))\
    .distinct()
print(conn.execute(stmt).fetchall())
```


參考資料
========

- [The Architecture of Open Source Applications (Volume 2): SQLAlchemy](http://aosabook.org/en/sqlalchemy.html)

- [SQL Expression Language Tutorial — SQLAlchemy 1.0 Documentation](http://docs.sqlalchemy.org/en/latest/core/tutorial.html)

- [SQLAlchemy 1.0 Documentation](http://docs.sqlalchemy.org/en/latest/)

- [SQLAlchemy 0.9 Documentation](http://docs.sqlalchemy.org/en/rel_0_9/)



[1]:https://pypi.python.org/pypi/pymssql
[2]:https://code.google.com/p/pyodbc/
[3]:http://adodbapi.sourceforge.net/
[4]:https://pypi.python.org/pypi/SQLAlchemy
