---
layout: post
title:  "[SQLAlchemy] ORM Tutorial"
date:   2015-01-19 12:00:00
comments: true
categories: [SQLAlchemy]
tags: [SQLAlchemy, Python]
---

在 [Python Central](http://www.pythoncentral.io/) 上找到一系列很不錯的 SQLAlchemy ORM 的教學文章，透過這一系列的文章可以很有條理的學習 SQLAlchemy：

1. [Introductory Tutorial of Python's SQLAlchemy](http://www.pythoncentral.io/introductory-tutorial-python-sqlalchemy/)

2. [How to Install SQLAlchemy on Windows, Mac and Linux](http://www.pythoncentral.io/how-to-install-sqlalchemy/)

3. [Python's SQLAlchemy vs Other ORMs](http://www.pythoncentral.io/sqlalchemy-vs-orms/)

4. [Overview of SQLAlchemy's Expression Language and ORM Queries](http://www.pythoncentral.io/overview-sqlalchemys-expression-language-orm-queries/)

5. [SQLAlchemy – Some Commonly Asked Questions](http://www.pythoncentral.io/sqlalchemy-faqs/)

6. [SQLAlchemy ORM Examples](http://www.pythoncentral.io/sqlalchemy-orm-examples/)

7. [SQLAlchemy Association Tables](http://www.pythoncentral.io/sqlalchemy-association-tables/)

8. [Understanding Python SQLAlchemy’s Session](http://www.pythoncentral.io/understanding-python-sqlalchemy-session/)

9. [SQLAlchemy Expression Language, Advanced Usage](http://www.pythoncentral.io/sqlalchemy-expression-language-advanced/)

10. [SQLAlchemy Expression Language, More Advanced Usage](http://www.pythoncentral.io/sqlalchemy-expression-language-advanced-usage/)

11. [Migrate SQLAlchemy Databases with Alembic](http://www.pythoncentral.io/migrate-sqlalchemy-databases-alembic/)


Model 定義重點
==============

原本光閱讀 SQLAlchemy 官網的文件資料，看得真的有點糊裡糊塗的；但仔細閱讀了上面的 tutorial 之後，有把我一些疑問釐清了，像是：

1. 資料庫若是已經定義好 table schema，可以使用 autoload 的方式將相關資訊轉成 class。

2. 若是對自動轉出來的 class 覺得不足或是想要調整，可以乾脆自己定義，只要讓程式可以正確 mapping 回資料庫即可。

3. 關於**關聯**的部分相當重要，程式中可不可以透過物件連結的方式將不同 table 的資料優雅的串起來，就靠這個了，所以 [relation](http://docs.sqlalchemy.org/en/latest/orm/relationships.html) & [backref](http://docs.sqlalchemy.org/en/latest/orm/backref.html) 這兩個部份真的要仔細研究一下。


Model 定義範例
==============

### One to One

首先是最簡單的一對一關係：

``` python
class User(Base):
    __tablename__ = 'user'

    id = Column(Integer, primary_key=True)
    name = Column(String(256))


class Address(Base):
    __tablename__ = 'address'
    id = Column(Integer, primary_key=True)
    address_content = Column(String(256))
    user_id = Column(Integer, ForeignKey('user.id'))
    
    ''' 一對一關係，uselist 參數設定為 False '''
    可透過 (User Object).address.address_content 取得使用者的地址 */
    user = relationship('User', backref=backref('address', uselist=False))
```


### One to Many

再來是最常看見的一對多關係：

``` python
class User(Base):
    __tablename__ = 'user'
    
    id = Column(Integer, primary_key=True)
    name = Column(String(256))


class Post(Base):
    __tablename__ = 'post'
    
    id = Column(Integer, primary_key=True)
    # 參考到 user.id
    owner_id = Column(Integer, ForeignKey('user.id'))
    
    ''' 可透過 (User Object).posts 取得 Post 資料
   	也可透過 (Post Object).owner 取得 User 資料
    有 passive_updates 參數設定為 False，(User Object).id 變更時，
    會連同將參考對應的 (Post Object).owner_id 一併變更 '''
    owner = relationship(User, backref=backref('posts', uselist=True, passive_updates=False))
```

以上的 model 定義等同於以下的方式定義：

``` python
class User(Base):
    __tablename__ = 'user'
    id = Column(Integer, primary_key=True)
    name = Column(String(256))
    posts = relationship("Post", back_populates="owner")
 

class Post(Base):
    __tablename__ = 'post'
    id = Column(Integer, primary_key=True)
    owner_id = Column(Integer, ForeignKey('user.id'))
    owner = relationship(User, back_populates="posts")
```

### Many to Many

而較為複雜的多對多定義，可參考如下：

``` python
class Department(Base):
    __tablename__ = 'department'

    id = Column(Integer, primary_key=True)
    name = Column(String)

    ''' 透過 secondary 參數指定 many-to-many 關係所使用的 table 名稱 '''
    employees = relationship('Employee', secondary='department_employee_link')


class Employee(Base):
    __tablename__ = 'employee'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    hired_on = Column(DateTime, default=func.now())

    ''' 透過 secondary 參數指定 many-to-many 關係所使用的 table 名稱 '''
    departments = relationship(Department, secondary='department_employee_link')


class DepartmentEmployeeLink(Base):
    __tablename__ = 'department_employee_link'

    ''' 指定參考到 department.id '''
    department_id = Column(Integer, ForeignKey('department.id'), primary_key=True)
    ''' 指定參考到 employee.id '''
    employee_id = Column(Integer, ForeignKey('employee.id'), primary_key=True)
    extra_data = Column(String(256))
    ''' 可透過 (Department Object).employee_assoc 取得此部門在 department_employee_link 中的所有資料 '''
    department = relationship(Department, backref=backref("employee_assoc"))
    ''' 可透過 (Employee Object).department_assoc 取得此員工在 department_employee_link 中的所有資料 '''
    employee = relationship(Employee, backref=backref("department_assoc"))
```


Session Object Status
=====================

當程式開始與資料庫往來時，Session 就扮演著中間者的角色，不僅可用來查詢資料庫中的資料，也用來維護使用者在程式中所使用的所有相關物件，並在必要時將物件的最新資訊更新至資料庫。

透過 Session，使用者可以知道目前資料的狀態是處於準備被新增、或是準備被修改，甚至準備被刪除；當然，若是因為某些條件發生而取消所有的資料更新，也可以透過 rollback 的方式回復。

每個 session object 都會有四種不同的狀態，分別是 `Transient`, `Pending`, `Persistent`, `Detached`，以下用個簡單範例說明四個狀態的變化：

``` python
# -*- coding: utf-8 -*-
from sqlalchemy import inspect
from conn_db import dbSession
from tutorial_7.model_define import Employee

usr_leon = Employee(name='Leon')
ins = inspect(usr_leon)
print('Transient(%s); Pending(%s); Persistent(%s); Detached(%s)'
      % (ins.transient, ins.pending, ins.persistent, ins.detached))
# Transient(True); Pending(False); Persistent(False); Detached(False)

dbSession.add(usr_leon)
print('Transient(%s); Pending(%s); Persistent(%s); Detached(%s)'
      % (ins.transient, ins.pending, ins.persistent, ins.detached))
# Transient(False); Pending(True); Persistent(False); Detached(False)

dbSession.commit()
print('Transient(%s); Pending(%s); Persistent(%s); Detached(%s)'
      % (ins.transient, ins.pending, ins.persistent, ins.detached))
# Transient(False); Pending(False); Persistent(True); Detached(False)

dbSession.close()
print('Transient(%s); Pending(%s); Persistent(%s); Detached(%s)'
      % (ins.transient, ins.pending, ins.persistent, ins.detached))
# Transient(False); Pending(False); Persistent(False); Detached(True)
```


其他相關資料
===========

- 若 Table 欄位參考到自己本身的另一個欄位，例如：Employee Table 中上司下屬的階層關係，可使用 `sqlamp` plugin 來處理。

- Database Migration Tool - [SQL Migration Tool - Alembic](https://alembic.readthedocs.org/en/latest/)


參考資料
=======

- [Python SQLAlchemy Tutorial | Python Central](http://www.pythoncentral.io/series/python-sqlalchemy-database-tutorial/)

- [Introductory Tutorial of Python's SQLAlchemy | Python Central](http://www.pythoncentral.io/introductory-tutorial-python-sqlalchemy/)


[1]:http://docs.sqlalchemy.org/en/latest/core/metadata.html#sqlalchemy.schema.MetaData
[2]:http://docs.sqlalchemy.org/en/latest/orm/extensions/declarative/api.html#sqlalchemy.ext.declarative.declarative_base
[3]:http://docs.sqlalchemy.org/en/latest/orm/session_basics.html
