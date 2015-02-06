---
layout: post
title:  "[Django] 在 Django 中使用 SQLAlchemy"
date:   2015-02-05 17:30:00
published: true
comments: true
categories: [django]
tags: [Django, Python, SQLAlchemy]
---

前言
====

在開發 web 的時候，如果有存取到資料庫的部分，若有使用 ORM(object relational mapping) 的相關 library，就會覺得使用類似存取物件的方式來存取資料庫，真的是一件很方便的事情。

在 Django 中有提供 [Django ORM helper](https://docs.djangoproject.com/en/1.7/topics/db/) 給開發者來使用，讓開發者可以很方便的進行資料庫存取；但唯一的問題就是這個功能僅能在 Django 中使用，當一脫離 Django 的開發範圍時，就沒辦法繼續使用了...

於是另外找到了 [SQLAlchemy](http://www.sqlalchemy.org/)，這個屬於外部的工具，與 Django 並沒有直接關連，但可以用來取代 Django ORM 來處理 Django 開發中的 model 部分的工作。

------------------------

SQLAlchemy 架構簡介
====================

為了讓 SQLAlchemy 可以用在 Django 專案中，先必須了解 SQLAlchemy 的運作方式，以下用一張圖來說明：

![SQLAlchemy layer diagram](http://aosabook.org/images/sqlalchemy/layers.png)

從上圖可以看出，SQLAlchemy 是依賴實作 DBAPI 的 library 在運作的，將其抽象化後提供給開發者使用，雖然這樣的做法會使得效能變得比較不好，但卻可提供開發者使用相同的方式來存取各種不同的關聯式資料庫，因此兩相權衡之下，若是專案整體架構下，DB performace 並不是太大的 issue 的話，使用 SQLAlchemy 作為資料庫的操作工具是個相當好的選擇。

------------------------

初始化專案
==========

這邊新產生一個專案，名稱為 **PropertyRental**，並新增一個 app，名稱為 **property_manage**：

``` bash
$ django-admin startproject Scholarship
$ cd Scholarship
$ python3 manage.py startapp SchoolManage
```

以下是目前專案結構：

``` bash
```

------------------------

套件安裝
=========

資料庫的部分，我們所使用的是 [MariaDB](https://mariadb.org/)，因此我們選擇用來連接資料庫的套件是 [PyMySQL](https://github.com/PyMySQL/PyMySQL)(當然也有其他不同的選擇)

以下透過 pip3 安裝 PyMySQL & SQLAlchemy 兩個套件

``` bash
$ sudo pip3 install pymysql
$ sudo pip3 install sqlalchemy
```

確認一下有沒有裝好：

``` bash
$ python3
Python 3.4.0 (default, Apr 11 2014, 13:05:11)
[GCC 4.8.2] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import sqlalchemy
>>> sqlalchemy.__version__
'0.9.8'
```

上面顯示了目前系統中 SQLAlchemy 的版本號是 0.9.8，表示套件有正確安裝。

------------------------

Model 定義
==========

Model 的部分要改成 SQLAlchemy 有以下幾個步驟：

### 1、與資料庫連線

首先在專案根目錄下新增資料庫連線檔 `.\dbConn.py`：

``` python
# -*- coding: utf-8 -*-
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

dbEngine = create_engine('mysql+pymysql://<YOUR_LOGIN_NAME>:<YOUR_PASSWORD>@<YOUR_DB_SERVER_IP>/<YOUR_DB_NAME>?charset=utf8', pool_recycle=3600)
dbSession = sessionmaker(bind=dbEngine)()
```

### 2、設定 Model

編輯 `.\SchoolManage\models.py` 

``` python
# -*- coding: utf-8 -*-
from sqlalchemy import Column, Integer, Unicode, ForeignKey, String
from sqlalchemy.orm import relationship, backref
from sqlalchemy.ext.declarative import declarative_base

from dbConn import dbEngine


Base = declarative_base()

class SchoolType(Base):
    __tablename__ = 'school_type'
    __table_args__ = {
        'mysql_engine': 'InnoDB',
        'mysql_charset': 'utf8'
    }

    id = Column(Integer, primary_key=True)
    type_name = Column(Unicode(128), nullable=False)
    remark = Column(Unicode(256))
    print_sort = Column(Integer, server_default='0', nullable=False)

class School(Base):
    __tablename__ = 'school'
    __table_args__ = {
        'mysql_engine': 'InnoDB',
        'mysql_charset': 'utf8'
    }

    id = Column(Integer, primary_key=True)
    apply_year = Column(Integer, default='func.year(now())', nullable=False)
    sch_code = Column(String(32), nullable=False)
    sch_name = Column(Unicode(128), nullable=False)
    school_type_id = Column(Integer, ForeignKey('school_type.id'))
    print_sort = Column(Integer, server_default='9999', nullable=False)

    school_type = relationship('SchoolType', backref=backref('schools', uselist=True, passive_updates=False))

class Academy(Base):
    __tablename__ = 'academy'
    __table_args__ = {
        'mysql_engine': 'InnoDB',
        'mysql_charset': 'utf8'
    }

    id = Column(Integer, primary_key=True)
    apply_year = Column(Integer, default='func.year(now())', nullable=False)
    acad_code = Column(String(32), nullable=False)
    acad_name = Column(Unicode(256), nullable=False)
    school_id = Column(Integer, ForeignKey('school.id'))

    school = relationship('School', backref=backref('academies', uselist=True, passive_updates=False))

Base.metadata.create_all(dbEngine)
```

------------------------

撰寫 View Function
==================

設定好 model 之後，再來就是撰寫 view function 了，接著建立 `.\SchoolManage\views.py`

``` python
from django.shortcuts import render
from SchoolManage.models import School
from dbConn import dbSession

def index(request):
    all_schools = dbSession.query(School).all()
    context = {'all_schools': all_schools}
    return render(request, 'SchoolManage/index.html', context)
```

------------------------

URL Routing 設定
================

這個部分包含了專案 & App 兩個部份的 url routing 設定，說明如下：

### 1、設定 App url routing rule

接著設定 url roting 的規則，將剛剛撰寫的 view function 設定為 app 首頁，建立檔案 `.\SchoolManage\urls.py`：

``` python
from django.conf.urls import patterns, url
from SchoolManage import views

urlpatterns = patterns('', url(r'^$', views.index, name='index'),)
```

### 2、設定專案 url routing rule

再來調整專案的 routing 規則，把 app 的 routing 規則改由 app 底下的 urls.py 來設定，調整 `.\Scholarship\urls.py` 如下：

``` python
from django.conf.urls import patterns, include, url

urlpatterns = patterns('',
    url(r'^SchoolManage/', include('SchoolManage.urls', namespace='SchoolManage')),
)
```

------------------------

調整專案設定
============

最後調整專案設定，修改 `.\Scholarship\settings.py`，在 `INSTALLED_APPS` 設定中加入 `SchoolManage`。


------------------------

檢視專案
========

大功告成，最後執行 `python3 manage.py runserver 0.0.0.0:8000` 瀏覽 [http://YOUR_SERVER_IP:8000/SchoolManage/](http://YOUR_SERVER_IP:8000/SchoolManage/) 即可建立 table 並瀏覽資料!

不過一開始沒有資料的話，可以連線進 MariaDB 建立幾筆資料後，重新整理網頁即可看到相關資料。

------------------------

參考資料
========

- [Replacing Django's ORM with SQLAlchemy - Irrational Exuberance](http://lethain.com/replacing-django-s-orm-with-sqlalchemy/)

- [Column Insert/Update Defaults — SQLAlchemy 1.0 Documentation](http://docs.sqlalchemy.org/en/latest/core/defaults.html)


[1]: https://docs.djangoproject.com/en/1.7/topics/