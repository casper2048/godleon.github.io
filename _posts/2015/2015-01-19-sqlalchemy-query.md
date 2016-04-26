---
layout: post
title:  "[SQLAlchemy] Query"
date:   2015-01-19 12:27:00
comments: true
categories: [SQLAlchemy]
tags: [SQLAlchemy, Python]
---

這篇文章主要是介紹一下在 SQLAlchemy 中查詢的撰寫方式，從 Model 定義先開始，接著新增資料，再來才是查詢。

Model 定義
==========

首先是 Model 的定義，建立檔案 `model_define.py`：

``` python
# -*- coding: utf-8 -*-
from sqlalchemy import Column, String, Integer, ForeignKey, Float
from sqlalchemy.orm import relationship, backref
from sqlalchemy.ext.declarative import declarative_base
# 在另外一個檔案中定義的 engine，用以連接 SQL Server 資料庫
from conn_db import engine

# 宣告 declarative base class
Base = declarative_base()


class User(Base):
    __tablename__ = 'user'

    id = Column(Integer, primary_key=True)
    name = Column(String(256))


class ShoppingCart(Base):
    __tablename__ = 'shopping_cart'

    id = Column(Integer, primary_key=True)
    owner_id = Column(Integer, ForeignKey(User.id))

    owner = relationship(User, backref=backref('shopping_carts', uselist=True))
    products = relationship('Product', secondary='shopping_cart_product_link')

    def __repr__(self):
        return '( {0}:{1.owner.name}:{1.products!r} )'.format(ShoppingCart, self)


class Product(Base):
    __tablename__ = 'product'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    price = Column(Float)

    shopping_carts = relationship('ShoppingCart', secondary='shopping_cart_product_link')

    def __repr__(self):
        return '( {0}:{1.name!r}:{1.price!r} )'.format(Product, self)


class ShoppingCartProductLink(Base):
    __tablename__ = 'shopping_cart_product_link'
    shopping_cart_id = Column(Integer, ForeignKey('shopping_cart.id'), primary_key=True)
    product_id = Column(Integer, ForeignKey('product.id'), primary_key=True)


Base.metadata.create_all(engine)
```


新增資料
=======

接著新增資料到上述定義的 table 中：

``` python
# -*- coding: utf-8 -*-
from conn_db import dbSession
from tutorial_9.model_define import Product, ShoppingCart, User

cpu = Product(name='CPU', price=300.00)
motherboard = Product(name='Motherboard', price=150.00)
coffee_machine = Product(name='Coffee Machine', price=30.00)
john = User(name='John')
dbSession.add(cpu)
dbSession.add(motherboard)
dbSession.add(coffee_machine)
dbSession.add(john)
dbSession.commit()

john_shopping_cart_computer = ShoppingCart(owner=john)
john_shopping_cart_kitchen = ShoppingCart(owner=john)
john_shopping_cart_computer.products.append(cpu)
john_shopping_cart_computer.products.append(motherboard)
john_shopping_cart_kitchen.products.append(coffee_machine)
dbSession.add(john_shopping_cart_computer)
dbSession.add(john_shopping_cart_kitchen)
dbSession.commit()
```


資料查詢
========

### 查詢價格在 100 元以上的商品

#### 使用 Express Language 查詢
``` python
product_higher_than_one_hundred = select([Product.id]).where(Product.price > 100.00)
q = dbSession.query(Product).filter(Product.id.in_(product_higher_than_one_hundred)).all()
pprint(q)
```

#### 使用 ORM 查詢
``` python
orm_q = dbSession.query(Product).filter(Product.price > 100.00).all()
pprint(orm_q)
```


### 查詢購物車中沒有 100 元以下的商品的購物車清單

#### 使用 Express Language 查詢
``` python
products_lower_than_one_hundred = select([Product.id]).where(Product.price < 100.00)
shopping_carts_with_no_products_lower_than_one_hundred = select([ShoppingCart.id])\
    .where(not_(ShoppingCart.products.any(Product.id.in_(products_lower_than_one_hundred))))
q = dbSession.query(ShoppingCart)\
    .filter(ShoppingCart.id.in_(shopping_carts_with_no_products_lower_than_one_hundred)).all()
pprint(q)
```

#### 使用 ORM 查詢
``` python
orm_q = dbSession.query(ShoppingCart)\
    .filter(and_(ShoppingCartProductLink.shopping_cart_id == ShoppingCart.id,
                 ShoppingCartProductLink.product_id == Product.id))\
    .filter(Product.price > 100)\
    .all()
pprint(orm_q)
```


### 查詢購物車內總價大於 200 元的購物車資訊

``` python
total = dbSession.query(ShoppingCart.id, func.sum(Product.price))\
    .filter(and_(ShoppingCart.id == ShoppingCartProductLink.shopping_cart_id,
                 ShoppingCartProductLink.product_id == Product.id))\
    .group_by(ShoppingCart.id)\
    .having(func.sum(Product.price) > 200).all()
pprint(total)
```


參考資料
=======

- [Python SQLAlchemy Tutorial](http://www.pythoncentral.io/series/python-sqlalchemy-database-tutorial/)

- [SQLAlchemy Lastest Documentation](http://docs.sqlalchemy.org/en/latest/)
