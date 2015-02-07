---
layout: post
title:  "[SQLAlchemy] 架構簡介 - Core"
date:   2014-12-06 20:58:00
comments: true
categories: [SQLAlchemy]
tags: [SQLAlchemy, Python]
---

SQLAlchemy 是什麼?
=================

首先引用自[SQLAlchemy 官方網站][2]的介紹：

> SQLAlchemy is the Python SQL toolkit and Object Relational Mapper that gives application developers the full power and flexibility of SQL. 

> It provides a full suite of well known enterprise-level persistence patterns, designed for efficient and high-performing database access, adapted into a simple and Pythonic domain language.

接著再看一段在[維基百科][1]中的介紹：

> SQLAlchemy是Python程式語言下的一款開源軟體。提供了SQL工具包及物件關聯對映（ORM）工具，使用MIT許可證發行。

> SQLAlchemy「採用簡單的Python語言，為高效和高效能的資料庫存取設計，實現了完整的企業級持久模型」。SQLAlchemy的理念是，SQL資料庫的量級和效能重要於物件集合；而物件集合的抽象又重要於表和行。因此，SQLAlchmey採用了類似於Java里Hibernate的資料對映模型。

SQLAlchemy 包含了 Core & ORM 兩大部分，而看到上述的說明其實就很清楚了，若您之前是 .NET Developer，其實就是對應到ADO.NET(Core) & Entity Framework(ORM) 囉。

而這些工具被開發出來的主要用意，就是提供 Database Abstraction 的功能，讓開發者可以透過統一介面，且更容易地使用各種不同的資料庫。


Core & ORM
==========

SQLAlchemy 架構分為 Core & ORM 兩個部分，以下是架構圖：

![SQLAlchemy layer diagram](http://aosabook.org/images/sqlalchemy/layers.png)

其實 SQLAlchemy 的概念很簡單，大概條列如下：

1. SQLAlchemy 並沒有提供直接連接資料庫的功能，而是透過有實作 DBAPI 的第三方套件來實現連接資料庫的功能。

2. SQLAlchemy Core 包裝了第三方套件後並抽象化，提供出一個一致性且更為好用的介面供開發者利用，也做為 ORM 的底層呼叫之用；而在 Core 中，開發者所接觸到的就是跟直接操作資料庫的經驗是差不多的，遇到的就是 Table、Column、select...等等，而透過 select 回傳的資料集稱為 ResultProxy。

3. SQLAlchemy ORM 則是將 Core 再包裝並抽象化，讓開發者可以用操作物件的形式來進行資料庫的操作。


SQLAlchemy 與 DBAPI 的相互關係
=============================

以下先看以下這張圖：

![Engine, Connection, ResultProxy API](http://aosabook.org/images/sqlalchemy/engine.png)

大致上歸納如下：

1. 在 SQLAlchemy 中，所有動作都是由 `create_engine` 方法產生 Engine 物件後，再產生 Dialect / Connection ... 等物件來與外部介面溝通。

2. 其中的 Pool 作為資料庫連線池(connection pool)的管理之用。

3. Dialect 直接與實作 DBAPI 的第三方套件溝通之用。

4. SQLAlchemy Connection 與 Pool 的功能都會直接使用到 DBAPI Connection 功能。

5. 原本在 DBAPI 中的 Cursor 功能，在 SQLAlchemy 則再包裝成 ResultProxy & ExecutionContext 來使用。


The Dialect System
==================

什麼是 Dialect System? 

在說明之前必須先了解到，SQLAlchemy 可用來操作多種資料庫，例如 MS SQL Server、MySQL、PostgreSQL ...等等，雖然目前有很多已經實作了 DBAPI 的套件可用，但還是有某些 API 會因為每個資料庫所提供的特性 or 功能不同而有差異，而 SQLAlchemy 的目標就是要將這個差異抽象化，提供一致的介面，讓開發者不論是操作哪種資料庫，都可以使用相同的方式。

接著以下有兩張圖可以說明，SQLAlchemy Dialect System 在整個架構中所扮演的角色：

![Simple Dialect/ExecutionContext hierarchy](http://aosabook.org/images/sqlalchemy/engine.png)

![Common DBAPI behavior shared among dialect hierarchies](http://aosabook.org/images/sqlalchemy/common-dbapi.png)

以上兩個是 PostgreSQL & MySQL 的範例說明，從圖中可以看出 SQLAlchemy 將各個實作 DBAPI 的第三方套件的差異性給封裝了起來並將其抽象化後，提供了一個一致性的介面給開發者(圖中的 DefaultDialect & Dialect)，讓開發者在面對不同的資料庫時，都可以使用相同的語法。


Schema 定義
===========

在 SQLAlchemy Core 中將 SQL 相關的特性，通通定義成一個個的類別，讓開發者可以透過產生物件的方式來對資料庫進行 SQL 相關操作。

![Basic sqlalchemy.schema objects](http://aosabook.org/images/sqlalchemy/basic-schema.png)

在資料庫定義中，像是 Table、Column、Squence、Index、Constraint ... 等等都被定義成物件，並透過 MetaData 來存放這些物件的集合與關係，用來對應到實際資料庫的狀況。

以下這張圖則是說明 SQLAchemy 是如何將一個個的物件轉換成相關的 DDL 語法：

![The dual lives of Table and Column](http://aosabook.org/images/sqlalchemy/table-column-crossover.png)


SQL 語法的生成
=============

在 SQLAlchemy 中有一個主要的 compile 功能，負責將各個物件的相互關係與動作轉換成實際的 SQL 語法，而與 compile 相關的類別及階層關係如下：

![Compiler hierarchy, including PostgreSQL-specific implementation](http://aosabook.org/images/sqlalchemy/compiler-hierarchy.png)

其中 `TypeCompiler` 的部分則是額外處理每個資料庫相異於一般資料庫的的特殊語法之用。


參考資料
========

- [SQLAlchemy - 維基百科，自由的百科全書][1]

- [Why SQLAlchemy is Awesome // Speaker Deck](https://speakerdeck.com/mitsuhiko/why-sqlalchemy-is-awesome)


[1]:http://zh.wikipedia.org/wiki/SQLAlchemy
[2]:http://www.sqlalchemy.org/
[3]:http://aosabook.org/en/sqlalchemy.html

