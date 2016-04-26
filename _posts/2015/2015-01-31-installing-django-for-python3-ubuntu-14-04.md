---
layout: post
title:  "[Django] 安裝 Django for Python3"
date:   2015-01-31 15:23:00
comments: true
categories: [django]
tags: [Django, Python]
---

原本是使用 PyCharm 作為練習 Django 之用，但為了瞭解 Django 的細節，我決定還是從安裝、開發、佈署，一步一步直接從 Linux command line 做起好了。

我所使用的平台為 `Ubuntu 14.04 Server LTS`

在目前的環境中，Python 指令預設的版本為 2.7.6，若要使用 python 3，則要用 `python3` 指令

直接安裝 python-django 會變成幫 python 2 安裝 Django，在 Python 3 中會無法 import，所以要改用以下方式安裝：(使用 `root` 身分)

### 1、安裝 pip3
> apt-get -y install python3-pip

### 2、安裝 Django
> pip3 install django

接著可以透過以下方式確認 Django 已經安裝完成：

``` bash
$ python3
Python 3.4.0 (default, Apr 11 2014, 13:05:11)
[GCC 4.8.2] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import django
>>> print(django.get_version())
1.7.4
>>> quit()
```

最後很快的測試一下是否可以啟動 Django server：

``` bash
$ django-admin startproject hello_django
$ cd hello_django/
$ python3 manage.py runserver 0.0.0.0:8000
Performing system checks...

System check identified no issues (0 silenced).

You have unapplied migrations; your app may not work properly until they are applied.
Run 'python manage.py migrate' to apply them.

January 31, 2015 - 07:40:13
Django version 1.7.4, using settings 'hello_django.settings'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.

```

若是可以在 [http://your_server_ip:8000](http://your_server_ip:8000) 看到預設網頁顯示，就表示成功囉!
