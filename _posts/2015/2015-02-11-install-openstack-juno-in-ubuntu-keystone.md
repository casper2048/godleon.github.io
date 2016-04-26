---
layout: post
title:  "[OpenStack] 安裝 Juno @ Ubuntu 14.04 (2) - 設定 Identity Service(Keystone)"
date:   2015-02-11 10:00:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Ubuntu, Linux]
---

Identity Service 概觀
=====================

Identity Service 在整個 OpenStack 架構中提供了兩項功能：

1. 認證 &授權
2. 提供可用服務的 API 服務端點目錄資訊

Identity Service 提供了 Role-based 的管理概念，並提供傳統的 UserName/Password & Token 的認證方式。

以下的來自官網的圖，描述了使用者啟用一個 instance 時，在 Identity Service(Keystone) 中的完整作業流程：
![Keystone 作業流程](http://docs.openstack.org/juno/install-guide/install/apt/content/figures/2/figures/SCH_5002_V00_NUAC-Keystone.png)

### 重要名詞說明

Identity Service 有些觀念要先清楚，列出比較容易搞混的：

#### Tenant

tenant 是 identity 服務的操作者用來將特定的 resource 或是 identity objects 區隔。

每一個 tenant 可能會對應到一個客戶，或是一個帳號，也有可能是一個專案。

#### Role

role 包含了指定功能的使用權限，管理者可以根據不同的 role 給定不同的權限，再將 role 指定給 user，每個 user 可以同時被指定為多個 role 藉以授予系統存取權限。

### 重要觀念說明

1. 每個 user 會隸屬於一個或多個 tenant，假設將 tenant 視為不同的專案，表示每個 user 可以屬於多個不同的專案。

2. 在每個不同的 tenant 裡，user 可以各自屬於不同的 role (例如：Leon 在 A 專案有管理者權限，但在 B 專案卻只有 view 的權限)

所以透過 user / role / tenant 的結合，整個權限的管理是可以做到非常彈性的!

`其實還有 domain 的概念，但這個部分等到之後進階探討時在來說吧!`

---------------------------

安裝 Identity Service
=====================

了解 Keystone 的功能後，接著要進行安裝設定工作，以下工作皆在 controller node 下完成：

``` bash
# 登入 MariaDB
root@controller:~# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 29
Server version: 5.5.41-MariaDB-1ubuntu0.14.04.1 (Ubuntu)

Copyright (c) 2000, 2014, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

# 建立資料庫 keystone
MariaDB [(none)]> CREATE DATABASE keystone;
Query OK, 1 row affected (0.00 sec)

# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
Query OK, 0 rows affected (0.00 sec)

# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
Query OK, 0 rows affected (0.00 sec)

# 將權限設定寫入資料庫
MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> exit;
Bye
```

接著安裝 keystone 相關套件：

``` bash
root@controller:~# apt-get -y install keystone python-keystoneclient
```

安裝完 keystone 套件後，接著要先產生一個 admin token 作為初始設定之用，透過以下指令產生：

``` bash
root@controller:~# openssl rand -hex 10
ad484997c32ff1b38836
```

有了 token 之後，進行 `/etc/keystone/keystone.conf` 的修改：

``` bash
[DEFAULT]
admin_token=ad484997c32ff1b38836
verbose=True

[database]
connection=mysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
provider=keystone.token.providers.uuid.Provider
driver=keystone.token.persistence.backends.sql.Token

[revoke]
driver=keystone.contrib.revoke.backends.sql.Revoke
```

執行以下指令，佈署資料庫並重新啟動 keystone 服務：

``` bash
# 佈署資料庫
root@controller:~# su -s /bin/sh -c "keystone-manage db_sync" keystone

# 重新啟動 keystone 服務
root@controller:~# service keystone restart
```

最後進行微調，將不需要的 sqlite 資料庫刪除，並設定 crontab job：

``` bash
# 使用 MariaDB，移除不需要的 SQLite 資料庫
root@controller:~# rm -f /var/lib/keystone/keystone.db

# 自動移除已經過期的 token (預設不會移除)
root@controller:~# (crontab -l -u keystone 2>&1 | grep -q token_flush) || echo '@hourly /usr/bin/keystone-manage token_flush >/var/log/keystone/keystone-tokenflush.log 2>&1' >> /var/spool/cron/crontabs/keystone
```

---------------------------

建立 user / role / tenant
==========================

在建立 user / role / tenant 之前，必須先設定好 **OS_SERVICE_TOKEN** & **OS_SERVICE_ENDPOINT** 兩個環境變數，目的是為了以管理者的身分啟用並註冊 Identity Service：

``` bash
root@controller:~# export OS_SERVICE_TOKEN=ad484997c32ff1b38836
root@controller:~# export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
```

### 建立 tenant

``` bash
# 建立 admin tenant
root@controller:~# keystone tenant-create --name admin --description "Admin Tenant"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |           Admin Tenant           |
|   enabled   |               True               |
|      id     | 095227f7415a4bdc9a4f48f620208fd7 |
|     name    |              admin               |
+-------------+----------------------------------+

# 建立 demo tenant
root@controller:~# keystone tenant-create --name demo --description "Demo Tenant"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |           Demo Tenant            |
|   enabled   |               True               |
|      id     | 824e59f8952849648d19b99275e3a913 |
|     name    |               demo               |
+-------------+----------------------------------+

```

### 建立 user

``` bash
# 建立 admin user
root@controller:~# keystone user-create --name admin --pass ADMIN_PASS --email ADMIN_EMAIL_ADDRESS
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |        godleon@gmail.com         |
| enabled  |               True               |
|    id    | 42eba4d5f7d340979565d4f190ecd110 |
|   name   |              admin               |
| username |              admin               |
+----------+----------------------------------+

# 在 demo tenant 下建立 demo user
keystone user-create --name demo --tenant demo --pass DEMO_PASS --email DEMO_EMAIL_ADDRESS
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |        godleon@gmail.com         |
| enabled  |               True               |
|    id    | 31cce302b6f449a8b81615aa6c2bd309 |
|   name   |               demo               |
| tenantId | 824e59f8952849648d19b99275e3a913 |
| username |               demo               |
+----------+----------------------------------+
```

### 建立 admin role

``` bash
root@controller:~# keystone role-create --name admin
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|    id    | fcfa0370495e42289bc54aeff8bc453c |
|   name   |              admin               |
+----------+----------------------------------+
```

### 將 user & tenant 與 role 進行連結

``` bash
root@controller:~# keystone user-role-add --user admin --tenant admin --role admin
```

### 幫其他 openstack service 建立 tenant

其他 openstack service 同樣也需要 tenant / user / role 的相關權限來與其他 service 進行互動，每個 service 都會有指定的 user 身分，並歸屬於 **service tenant** 底下，搭配著 **admin role** 的權限來運作，以下先建立 **service tenant**：

``` bash
root@controller:~# keystone tenant-create --name service --description "Service Tenant"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |          Service Tenant          |
|   enabled   |               True               |
|      id     | 8cc687f0f03a49e0ae9f7c45a0c6c62e |
|     name    |             service              |
+-------------+----------------------------------+
```

---------------------------

定義 services & API 服務端點
============================

之前提過 Identity Service 其中一個重要功能是「**提供可用服務的 API 服務端點目錄資訊**」，因此安裝好的服務都必須向 Identity Service 註冊，就連 Identity Service 自己也不例外。

``` bash
# 註冊 keystone service
root@controller:~# keystone service-create --name keystone --type identity --description "OpenStack Identity"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |        OpenStack Identity        |
|   enabled   |               True               |
|      id     | 94078b94a100432d87e42c87a14c4405 |
|     name    |             keystone             |
|     type    |             identity             |
+-------------+----------------------------------+

# 設定 Identity Service 的 API 服務端點
root@controller:~# keystone endpoint-create --service-id $(keystone service-list | awk '/ identity / {print $2}') --publicurl http://controller:5000/v2.0 --internalurl http://controller:5000/v2.0 --adminurl http://controller:35357/v2.0 --region regionOne
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |   http://controller:35357/v2.0   |
|      id     | ef22c8101fbc49aabd4bfe65a5241925 |
| internalurl |   http://controller:5000/v2.0    |
|  publicurl  |   http://controller:5000/v2.0    |
|    region   |            regionOne             |
|  service_id | 94078b94a100432d87e42c87a14c4405 |
+-------------+----------------------------------+
```

---------------------------

驗證 Identity Service 是否安裝成功
==================================

最後我們要驗證之前所安裝的 Keystone 是否成功，可以執行以下步驟進行檢查：

### 將 `OS_SERVICE_TOKEN ` & `OS_SERVICE_ENDPOINT` 兩個環境變數取消設定：

``` bash
root@controller:~# unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
```

### 以 tenant(admin) & user(admin) 的身分取得 token

``` bash
root@controller:~# keystone --os-tenant-name admin --os-username admin --os-password ADMIN_PASS --os-auth-url http://controller:35357/v2.0 token-get
+-----------+----------------------------------+
|  Property |              Value               |
+-----------+----------------------------------+
|  expires  |       2015-02-11T02:29:16Z       |
|     id    | 7be61ad61718468a966f375db501f4ea |
| tenant_id | 095227f7415a4bdc9a4f48f620208fd7 |
|  user_id  | 42eba4d5f7d340979565d4f190ecd110 |
+-----------+----------------------------------+
```

### 以 tenant(admin) & user(admin) 的身分查詢 tenant 清單

``` bash
root@controller:~# keystone --os-tenant-name admin --os-username admin --os-password ADMIN_PASS --os-auth-url http://controller:35357/v2.0 tenant-list
+----------------------------------+---------+---------+
|                id                |   name  | enabled |
+----------------------------------+---------+---------+
| 095227f7415a4bdc9a4f48f620208fd7 |  admin  |   True  |
| 824e59f8952849648d19b99275e3a913 |   demo  |   True  |
| 8cc687f0f03a49e0ae9f7c45a0c6c62e | service |   True  |
+----------------------------------+---------+---------+
```

### 以 tenant(admin) & user(admin) 的身分查詢 user 清單

``` bash
root@controller:~# keystone --os-tenant-name admin --os-username admin --os-password ADMIN_PASS --os-auth-url http://controller:35357/v2.0 user-list
+----------------------------------+-------+---------+-------------------+
|                id                |  name | enabled |       email       |
+----------------------------------+-------+---------+-------------------+
| 42eba4d5f7d340979565d4f190ecd110 | admin |   True  | godleon@gmail.com |
| 31cce302b6f449a8b81615aa6c2bd309 |  demo |   True  | godleon@gmail.com |
+----------------------------------+-------+---------+-------------------+
```

### 以 tenant(admin) & user(admin) 的身分查詢 role 清單

``` bash
root@controller:~# keystone --os-tenant-name admin --os-username admin --os-password ADMIN_PASS --os-auth-url http://controller:35357/v2.0 role-list
+----------------------------------+----------+
|                id                |   name   |
+----------------------------------+----------+
| 9fe2ff9ee4384b1894a90878d3e92bab | _member_ |
| fcfa0370495e42289bc54aeff8bc453c |  admin   |
+----------------------------------+----------+
```

### 以 tenant(demo) & user(demo) 的身分取得 token

``` bash
root@controller:~# keystone --os-tenant-name demo --os-username demo --os-password DEMO_PASS --os-auth-url http://controller:35357/v2.0 token-get
+-----------+----------------------------------+
|  Property |              Value               |
+-----------+----------------------------------+
|  expires  |       2015-02-11T02:30:39Z       |
|     id    | bf02b9f158ad416e86f6cc7d82bccc42 |
| tenant_id | 824e59f8952849648d19b99275e3a913 |
|  user_id  | 31cce302b6f449a8b81615aa6c2bd309 |
+-----------+----------------------------------+
```

### 以 tenant(demo) & user(demo) 的身分取得使用者清單(顯示權限不足)

``` bash
root@controller:~# keystone --os-tenant-name demo --os-username demo --os-password DEMO_PASS --os-auth-url http://controller:35357/v2.0 user-list
You are not authorized to perform the requested action: admin_required (HTTP 403)
```

如果經過以上驗證，都會有類似結果顯示，表示 Keystone 安裝成功囉!

---------------------------

建立快速切換使用者身分的 script
===============================

由於我們在前面建立了 admin & demo 兩個 user，每個 user 所使用的環境變數皆不同，所以為了可以快速切換，我們建立以下兩個 script 作為後續使用：

`~/openstack/admin-openrc.sh`

``` bash
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:35357/v2.0
```

`~/openstack/demo-openrc.sh`

``` bash
export OS_TENANT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=DEMO_PASS
export OS_AUTH_URL=http://controller:5000/v2.0
```

---------------------------

參考資料
========

- [OpenStack Installation Guide for Ubuntu 14.04 - juno][1]

[1]:  http://docs.openstack.org/juno/install-guide/install/apt/content/