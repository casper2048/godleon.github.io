---
layout: post
title:  "[Django] Basic URL dispatch"
date:   2015-02-02 19:30:00
comments: true
categories: [django]
tags: [django, python]
---


專案目錄架構
===========

在說明 url dispatch 之前，先看一下目前專案目錄 & 檔案架構：

``` bash
├── db.sqlite3
├── hello_django
│   ├── __init__.py
│   ├── __pycache__
│   │   ├── __init__.cpython-34.pyc
│   │   ├── settings.cpython-34.pyc
│   │   ├── urls.cpython-34.pyc
│   │   ├── views.cpython-34.pyc
│   │   └── wsgi.cpython-34.pyc
│   ├── settings.py
│   ├── urls.py
│   ├── views.py
│   └── wsgi.py
└── manage.py
```

--------------------

自訂第一個 URL
=============

首先檢視設定檔 `hello_django/settings.py`，裡面與 url dispatch 相關的設定為 `ROOT_URLCONF`，其中的設定為：

``` bash
ROOT_URLCONF = 'hello_django.urls'
```

表示目前整個專案的 routing 設定是由 `hello_django/urls.py` 這個檔案中的設定所開始，預設的設定如下：

``` python
from django.conf.urls import patterns, include, url
from django.contrib import admin

urlpatterns = patterns('',
    url(r'^admin/', include(admin.site.urls)),
)
```

可以看出目前 Django 中只有針對 admin site 相關的 routing 設定，而 routing 的路徑是用正規表示式的方式設定。

接著以下先來建立第一個 view 並設定 routing 相關資訊，首先建立檔案 `hello_django/views.py`：

``` python
#filename: hello_django/views.py

from django.http import HttpResponse

def hello(request):
    return HttpResponse('Hello, UTF-8 測試，堃喆!')
```

接著作 routing 相關設定，修改 `hello_django/urls.py`：

``` python
#filename: hello_django/urls.py

from django.conf.urls import patterns, include, url
from django.contrib import admin
from hello_django import views

urlpatterns = patterns('',
    url(r'^admin/', include(admin.site.urls)),
    url(r'^hello$', views.hello),
)
```

接著瀏覽 [http://your_server_ip:8000/hello](http://your_server_ip:8000/hello) 就可以正確顯示 view function 所回應的訊息，而 UTF-8 的字元也可以正確顯示喔!

帶有參數的 URL
=============

Restful URL 當然不只如此了，以下再用幾個範例作更進一步介紹，這次把參數的部分一起帶進來，相關設定如下：

``` python
#filename: hello_django/views.py

from django.http import HttpResponse

def hello(request):
    return HttpResponse('Hello, UTF-8 測試，堃喆!')

'''處理沒有帶任何參數時的 request'''
def special_case_2003(request):
    return HttpResponse('this is page of special_case_2003')

'''處理帶有一個參數的 request'''
def year_archive(request, article_year):
    return HttpResponse('this is page of year {0} archive'.format(article_year))

'''處理帶有兩個參數的 request'''
def month_archive(request, article_year, article_month):
    return HttpResponse('this is page of year {0} month {1} archive'.format(article_year, article_month))

'''處理帶有三個參數的 request'''
def article_detail(request, article_year, article_month, article_no):
    return HttpResponse('this is page of {0}/{1} article({2}) detail'.format(article_year, article_month, article_no))
```

``` python
#filename: hello_django/urls.py
from django.conf.urls import patterns, include, url
from django.contrib import admin
from hello_django import views


urlpatterns = patterns('',
    # Examples:
    # url(r'^$', 'hello_django.views.home', name='home'),
    # url(r'^blog/', include('blog.urls')),

    url(r'^admin/', include(admin.site.urls)),
    url(r'^hello$', views.hello),
    url(r'^articles/2003/$', views.special_case_2003), # CASE 1
    url(r'^articles/(\d{4})/$', views.year_archive), # CASE 2
    url(r'^articles/(\d{4})/(\d{1,2})/$', views.month_archive), # CASE 3
    url(r'^articles/(\d{4})/(\d{1,2})/(\d+)/$', views.article_detail), # CASE 4
)
```

在 MVC 架構中，routing 部分的規則是`先符合先使用`，因此在整個 routing 過程的比對，會從第一條規則開始比對下來，若是有第一個符合的，就直接拿來使用。

### CASE 1

僅針對只有一個參數且為 2003 的 request 進行 routing

若網址為 `http://your_server_ip:8000/articles/2003/`，呼叫 **special_case_2003(request)** function 處理。

### CASE 2

傳遞 1 個參數至 view function

若網址為 `http://your_server_ip:8000/articles/2015/`，呼叫 **year_archive(request, 2015)** function 處理。

### CASE 3

傳遞 2 個參數至 view function

若網址為 `http://your_server_ip:8000/articles/2003/5/`，呼叫 **month_archive(request, 2003, 5)** function 處理。

若網址為 `http://your_server_ip:8000/articles/2014/11/`，呼叫 **month_archive(request, 2014, 11)** function 處理。

### CASE 4

傳遞 3 個參數至 view function

若網址為 `http://your_server_ip:8000/articles/2011/4/23`，呼叫 **month_archive(request, 2011, 4, 23)** function 處理。

--------------------

Named groups
============

按照上面的範例可以看出，Django 在比對到 routing rule 後，會將參數「**按照順序**」分別丟進對應 function，因此第一個參數是 article_year，第二個是 article_month，第三個是 article_no。

其實這順序是可以調整的，只要在 routing rules 中定義好參數名稱即可，以下修改 `hello_django/urls.py` 的內容：

``` python
#filename: hello_django/urls.py

import ...

urlpatterns = patterns('',
    # Examples:
    # url(r'^$', 'hello_django.views.home', name='home'),
    # url(r'^blog/', include('blog.urls')),

    url(r'^admin/', include(admin.site.urls)),
    url(r'^hello$', views.hello),
    url(r'^articles/2003/$', views.special_case_2003),
    url(r'^articles/(?P<article_year>\d{4})/$', views.year_archive),
    url(r'^articles/(?P<article_month>\d{1,2})/(?P<article_year>\d{4})/$', views.month_archive),
    url(r'^articles/(?P<article_no>\d+)/(?P<article_month>\d{1,2})/(?P<article_year>\d{4})/$', views.article_detail),
)
```

按照上面的設定，則會變成：

- `http://your_server_ip:8000/articles/2011`
呼叫 **year_archive(request, artile_year=2011)**

- `http://your_server_ip:8000/articles/4/2011`
呼叫 **month_archive(request, artile_year=2011, article_month=4)**

- `http://your_server_ip:8000/articles/50/1/2015`
呼叫 **article_detail(request, artile_year=2015, article_month=1, article_no=50)**

--------------------

URL dispatch 的遊戲規則
=======================

1. url dispatch 的工作是由 Django 中的 URLconf 來負責處理。

2. URL 中的所有參數都會被視為是 string

3. 比對每一條 routing rule 時，會以第一個符合作為比對結果，藉以提升比對的效率。

4. 指定的 view function 可用 string 來替代，但由於 Django 之後的版本會廢除掉此功能，所以這邊就不多著墨了

--------------------

App 的 routing 處理
===================

Django Project 是由很多 app 所組成的，上面看到的都是 main site 的設定，如果新增一個 app，那 url routing 又該怎麼作呢?

首先先見立一個新的 app，名稱為 `first_app`：

``` bash
$ python3 manage.py startapp first_app
```

接著整個 project 的目錄結構如下：

``` python
.
├── db.sqlite3
├── first_app
│   ├── admin.py
│   ├── __init__.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── hello_django
│   ├── __init__.py
│   ├── __pycache__
│   │   ├── __init__.cpython-34.pyc
│   │   ├── settings.cpython-34.pyc
│   │   ├── urls.cpython-34.pyc
│   │   ├── views.cpython-34.pyc
│   │   └── wsgi.cpython-34.pyc
│   ├── settings.py
│   ├── urls.py
│   ├── views.py
│   └── wsgi.py
└── manage.py
```

首先先建立 first_app 的 view function：

``` python
#filename: hello_django\first_app\views.py

from django.http import HttpResponse

def index(request):
    return HttpResponse('first_app - List')

def detail(request, object_id):
    return HttpResponse('first_app - Detail')
```

接著建立 first_app 專用的 url routing rule：

``` python
#filename: hello_django\first_app\urls.py

from django.conf.urls import patterns, url
from first_app import views

urlpatterns = patterns('', url(r'^$', views.index),
                        url(r'^(\d+)/$', views.detail),
)
```

最後在 main site 的 urls.py 裡面加入 first_app 的 url routing 設定就完工啦!

``` python
#filename: hello_django\urls.py

from django.conf.urls import patterns, include, url
from django.contrib import admin
from hello_django import views

urlpatterns = patterns('',
    # Examples:
    # url(r'^$', 'hello_django.views.home', name='home'),
    # url(r'^blog/', include('blog.urls')),

    url(r'^admin/', include(admin.site.urls)),
    url(r'^first_app/', include('first_app.urls', namespace='first_app')), # 加入這一行
    url(r'^hello$', views.hello),
    url(r'^articles/2003/$', views.special_case_2003),
    url(r'^articles/(?P<article_year>\d{4})/$', views.year_archive),
    url(r'^articles/(?P<article_month>\d{1,2})/(?P<article_year>\d{4})/$', views.month_archive),
    url(r'^articles/(?P<article_no>\d+)/(?P<article_month>\d{1,2})/(?P<article_year>\d{4})/$', views.article_detail),
)
```

所以，測試結果如下：

- `http://your_server_ip:8000/first_app`：由 **first_app.views.index** 處理

- `http://your_server_ip:8000/first_app/123`：由 **first_app.views.detail** 處理


最後 project 的專案結構如下：

```
.
├── db.sqlite3
├── first_app
│   ├── admin.py
│   ├── __init__.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── __pycache__
│   │   ├── __init__.cpython-34.pyc
│   │   ├── urls.cpython-34.pyc
│   │   └── views.cpython-34.pyc
│   ├── tests.py
│   ├── urls.py
│   └── views.py
├── hello_django
│   ├── __init__.py
│   ├── __pycache__
│   │   ├── __init__.cpython-34.pyc
│   │   ├── settings.cpython-34.pyc
│   │   ├── urls.cpython-34.pyc
│   │   ├── views.cpython-34.pyc
│   │   └── wsgi.cpython-34.pyc
│   ├── settings.py
│   ├── urls.py
│   ├── views.py
│   └── wsgi.py
└── manage.py
```

從以上可以看出我們是透過 django.conf.urls.include 的方式將 first_app 的 url routing rules 引用過來。

用這方式的好處在於，若是網站中很多 app 時，在 main site 的 urls.py 中不需要定義所有 app 的 routing rules，而可以分散到各個 app 中去處理，相對於來說會較好維護。 

--------------------

加入額外參數
===========

若在 url routing rule 中要加入額外參數也是可以，只是 view function 要隨著改變，以下用個範例說明：

``` python
#filename: hello_django\first_app\urls.py

from django.conf.urls import patterns, url
from first_app import views

urlpatterns = patterns('', url(r'^$', views.index),
                        url(r'^(\d+)/$', views.detail, {'foo': 'bar'}),
)
```

以上多加了一個 foo 參數，因此要調整 view function 如下：

``` python
#filename: hello_django\first_app\views.py

from django.http import HttpResponse

def index(request):
    return HttpResponse('first_app - List')

# 多加一個 foo 參數
def detail(request, object_id, foo='bar'):
    return HttpResponse('first_app - Detail')
```

如此一來，`http://your_server_ip:8000/first_app/123` 才可以順利執行!

此外，還可以從 main site 的 urls.py 中指定給 app 的額外參數，以下為範例：

``` python
#filename: hello_django\urls.py

from django.conf.urls import patterns, include, url
from django.contrib import admin
from hello_django import views

urlpatterns = patterns('',
    url(r'^admin/', include(admin.site.urls)),
    url(r'^first_app/', include('first_app.urls', namespace='first_app'), {'mainpara': 'main'}),
)
```
以上範例會送一個 `mainpara = 'main'` 的參數給 first_app。

-----------------------

URL Namming & Namespace
=======================

在設定 url routing rule 時，還可以進行 naming & namespace 的設定，這與 reversing url 有關，這部分較為複雜，等之後有提到 template 的部分再順便一起提吧!
