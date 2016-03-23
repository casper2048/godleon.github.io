---
layout: post
title:  "[Django] HTTP Request/Response 的加工廠 - Middleware"
date:   2015-02-07 08:25:00
published: true
comments: true
categories: [django]
tags: [Django, Python]
---

Middleware 是什麼?
==================

在 Django 中，middleware 的作用是外掛在 Django 處理 request/reponse 的過程中，針對 request/reponse 的資訊進行額外的加工處理。

每個 middleware 都有自己的專屬功能，以 [SessionMiddleware](https://docs.djangoproject.com/en/1.7/ref/middleware/#django.contrib.sessions.middleware.SessionMiddleware) 為例，就是用來處理 session 之用，若是在專案中要使用到 session 的功能，就必須要啟用 SessionMiddleware。

-----------------------------------

啟用 Middlwware
================

專案初始化時，可以在 settings.py 中找到 middleware 的設定如下：

``` python
MIDDLEWARE_CLASSES = (
	'django.middleware.common.CommonMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.auth.middleware.SessionAuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
)
```

這一段設定表示在這個專案中啟用的 middleware，而這些 middleware 是有順序性的，例如：CommonMiddleware 是所有 middleware 的基礎，因此是一定要有的，所以建議擺在設定的第一行。

而執行的方式可用下圖表示：

![middleware process order](https://docs.djangoproject.com/en/1.7/_images/middleware.svg)

因此我們可以把 middleware 視為 View Wrapper，可以針對 HTTP request/response 傳遞到 view 之前，進行額外的加工處理。

-----------------------------------

Middleware 的運作細節
=====================

每個 middleware 都是一個獨立的 python class，並根據 middleware 的功能定義以下一個或多個 method：

1、**process_request(request)**

在 Django 決定要將 http request 交給哪個 view function 處理之前，就會先執行到 process_request() 這一段程式。

`return None`：則會繼續執行其他 middleware 的 process_request() 方法，接著再繼續執行所有 middleware 的 process_view() 方法。

`return HttpResponse`：則會直接跳過所有 middleware 中關於 request & exception 的處理，而交給處理 response 的 middleware 繼續處理。

2、**process_view(request, view_func, view_args, view_kwargs)**

執行於 Django 呼叫 view function 之前，process_view 必須回應的是 None or [HttpResponse](https://docs.djangoproject.com/en/1.7/ref/request-response/#django.http.HttpResponse)。

`return None`：Django 則會繼續執行其他 middleware 的 process_view() 方法。

`return HttpResponse`：Django 則會跳過執行 view function 以及 exception middleware，直接將結果交給 middleware 中處理 response 的方法來繼續處理。

3、**process_template_response(request, response)**

當 view function 執行完後，緊接著就是執行 process_template_response 程式，顧名思義，若是在 view function 中有使用到 render() 方法，就表示 response 是屬於一個 [TemplateResponse](https://docs.djangoproject.com/en/1.7/ref/template-response/#django.template.response.TemplateResponse)

4、**process_response(request, response)**

process_response() 方法會在 Django 將 response 回傳給 browser 之前執行，可以根據功能需求針對 response 進行額外加工。

與 process_request() & process_view() 不一樣的是，process_response() 永遠都會執行不會被忽略，因此在開發時要注意到這一點。

5、**process_exception(request, exception)**

這個是當在 view function 中產生例外的時候，就會跑到這裡來做額外的處理。

`return None`：交由預設的例外處理程序處理。

`return HttpResponse`：交由接下來的 template response & response middleware 來繼續處理。

6、**__init__** (不一定需要)

一般 middleware 不會需要 initializer，但若是有些需要預先定義的全域變數，也可以在這邊進行設定。

-----------------------------------

停用 Middleware
===============

若想要在程式中動態的停用 middleware 要怎麼辦到呢?

只要在 middleware 的 `__init__(self)` 中呼叫 `django.core.exceptions.MiddlewareNotUsed` 例外即可，例如以下程式碼：

``` python
def __init__(self):
	if some_condition:
		raise django.core.exceptions.MiddlewareNotUsed
```

-----------------------------------

Middleware 範例
===============

後來在[網路上](http://stackoverflow.com/questions/17751163/django-display-time-it-took-to-load-a-page-on-every-page/17777539#17777539)找到一個不錯的範例，這個 middleware 主要是計算產生每一頁所花費的時間：

`.\Scholarship\middlewares.py`

``` python
from functools import reduce
from time import time
from operator import add
import re

from django.db import connection


class StatsMiddleware(object):

    def process_view(self, request, view_func, view_args, view_kwargs):
        '''
        In your base template, put this:
        <div id="stats">
        <!-- STATS: Total: %(total_time).2fs Python: %(python_time).2fs DB: %(db_time).2fs Queries: %(db_queries)d ENDSTATS -->
        </div>
        '''

        n = len(connection.queries)

        # time the view
        start = time()
        response = view_func(request, *view_args, **view_kwargs)
        total_time = time() - start

        # compute the db time for the queries just run
        db_queries = len(connection.queries) - n
        if db_queries:
            db_time = reduce(add, [float(q['time'])
                                   for q in connection.queries[n:]])
        else:
            db_time = 0.0

        python_time = total_time - db_time

        stats = {
            'total_time': total_time,
            'python_time': python_time,
            'db_time': db_time,
            'db_queries': db_queries,
        }

        if response and response.content:
            s = response.content
            regexp = re.compile(b'(?P<cmt><!--\s*STATS:(?P<fmt>.*?)ENDSTATS\s*-->)')
            match = regexp.search(s)
            if match:
                s = (s[:match.start('cmt')] +
                     match.group('fmt') +
                     s[match.end('cmt'):]).decode() % stats
                response.content = s

        return response
```

接著修改 `.\Scholarship\settings.py`，將 `MIDDLEWARE_CLASSES` 變數設定如下：

``` python
MIDDLEWARE_CLASSES = (
	'django.middleware.common.CommonMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    ....(other import)
	'Scholarship.middlewares.StatsMiddleware',
)
```

接著在每一個想要看到產生時間的頁面中，加入以下 html 程式碼即可：

``` html
<div id="stats">
<!-- STATS: Total: %(total_time).2fs Python: %(python_time).2fs DB: %(db_time).2fs Queries: %(db_queries)d ENDSTATS -->
</div>
```

此外，由於 middleware 有執行順序的問題，所以會有以下不同：

1. 將 middleware 定義放在最後面 => 計算 `render template` 的時間

2. 將 middleware 定義放在最前面 => 計算`所有 middleware process request` + `render template` 的時間

-----------------------------------

參考資料
========

- [Middleware | Django documentation](https://docs.djangoproject.com/en/1.7/topics/http/middleware/)

- [i'm 安追: [Mezzanine] 利用 Django 的 Middleware 機制進行多國語系 (I18N) 處理 (使用 JavaScript 的解決方案)](http://www.andretw.com/2014/04/customize-your-mezzanine-with-Django-middleware-and-JavaScript-AngularJS-for-i18n-multilingual.html)

- [python - Django: display time it took to load a page on every page - Stack Overflow](http://stackoverflow.com/questions/17751163/django-display-time-it-took-to-load-a-page-on-every-page/17777539#17777539)