---
layout: post
title:  "[OpenStack] 安裝 Juno @ Ubuntu 14.04 (6) - 設定 Dashboard Service(Horizon)"
description: "安裝 dashboard service(Horizon)，讓管理者可以透過 Web GUI 的方式管理 OpenStack resources & services"
date:   2015-02-25 10:25:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Ubuntu, Linux]
---

Horizon 是什麼?
===============

首先來看看官網對 Horizon 的說明：

> The OpenStack dashboard, also known as Horizon, is a Web interface that enables cloud administrators and users to manage various OpenStack resources and services.

> The dashboard enables web-based interactions with the OpenStack Compute cloud controller through the OpenStack APIs.

> Horizon enables you to customize the brand of the dashboard.

> Horizon provides a set of core classes and reusable templates and tools.

簡單來說，Horizon 就是提供使用者可以透過圖形介面(網頁)簡單的管理雲端資源，加入 third-party 的特殊管理功能，還可以自由客製化成任何想要的樣子。

當然透過 Horizon 操作只是其中一個管理 & 使用 OpenStack 資源的方式，也可以透過直接下指令，甚至可以自行開發程式來呼叫相關 API 達成所需功能。

畢竟，OpenStack 上所有的 service 都是以 REST API-based 來提供服務的。

--------------------------------

安裝 & 設定 @ Controller Node
=============================

#### 安裝 Horizon 套件

以下將會把 Horizon 相關套件都安裝在 `Controller Node` 上

``` bash
root@controller:~# apt-get -y install openstack-dashboard apache2 libapache2-mod-wsgi memcached python-memcache
```

#### Django 設定檔調整

修改檔案 `/etc/openstack-dashboard/local_settings.py` 內容如下：

``` bash
OPENSTACK_HOST = "controller"

# session 設定 (預設使用 memcached 做 session 管理用)
CACHES = {
   'default': {
       'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
       'LOCATION': '127.0.0.1:11211',
   }
}

# 時區設定
TIME_ZONE = "Asia/Taipei"
```

#### 重新啟動服務

重新啟動 apache & memcached 兩個服務：

``` bash
root@controller:~# service apache2 restart
root@controller:~# service memcached restart
```

--------------------------------

登入 Horizon
============

安裝完畢後，我們可以透過 [http://controller/horizon](http://controller/horizon) or [http://10.0.0.11/horizon](http://10.0.0.11/horizon) 連到 Horizon

#### 登入畫面

![Horizon Portal](https://lh4.googleusercontent.com/-nStStTkZ2yc/VO0xGTMNW8I/AAAAAAAAJ_4/g50S_tupjQY/w441-h611-no/openstack_juno_horizon_portal.png)

#### 系統概觀

![Horizon Overview](https://lh4.googleusercontent.com/--2SgbqBczyw/VO0xGKFstSI/AAAAAAAAJ_4/Uh0oJz_E3dY/w958-h559-no/openstack_juno_horizon_overview.png)

#### 虛擬機器管理程式

![Horizon VMs](https://lh4.googleusercontent.com/-caSCyjmURzs/VO0xHLjFUaI/AAAAAAAAJ_4/_fKLwrEFnRQ/w958-h608-no/openstack_juno_horizon_vms.png)

#### 映像檔

![Horizon Images](https://lh6.googleusercontent.com/-U90lKftm-7E/VO0xFZDUJRI/AAAAAAAAJ_4/tPbgot--m-8/w915-h508-no/openstack_juno_horizon_images.png)

#### 所有執行實例

![Horizon VM Instances](https://lh4.googleusercontent.com/-V50bVpipq4s/VO0xFvCwmCI/AAAAAAAAJ_4/SgBE9EeJGlU/w958-h375-no/openstack_juno_horizon_instances.png)

#### 網路

![Horizon Networks](https://lh6.googleusercontent.com/-DQaGdrPpUys/VO0xFfEPr4I/AAAAAAAAJ_4/y6gUlmn1u-Q/w913-h534-no/openstack_juno_horizon_networks.png)

#### 路由器

![Horizon Routers](https://lh3.googleusercontent.com/-lWS9JTKiCes/VO0xG2NcLOI/AAAAAAAAJ_4/Z-smjHIm2HY/w903-h407-no/openstack_juno_horizon_routers.png)

------------------------------

參考資料
========

- [OpenStack Installation Guide for Ubuntu 14.04 - juno][1]

[1]:  http://docs.openstack.org/juno/install-guide/install/apt/content/