---
layout: post
title:  "[OpenStack] 安裝 Juno @ Ubuntu 14.04 (3) - 設定 Image Service(Glance)"
date:   2015-02-11 12:30:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Ubuntu, Linux]
---

Image Service 概觀
================

在 OpenStack 中，Image Service 的用途在於讓使用者可以透過 REST API 尋找/註冊/取得虛擬機器的映像檔來使用，而映像檔可以存放於多種不同的位置，提供這樣的服務的專案稱為 ***Glance***。

Glance 是用來管理虛擬磁碟的 image 之用，除了可以讓使用者新增 image 之外，也可以從正在運作的 server 上取得 snapshop 來作為 image 的備份或者是其他虛擬磁碟的 image。

Glance 包含了以下四個主要部分：

- **glance-api**
	- 接受來自其他服務的 API call

- **glance-registry**
	- 管理 image 的 metadata 之用(例如：image 的大小 & 類型)。

- **Database**
	- 儲存 image metadata 之用，可選擇 MySQL or SQLite。

- **存放 image 的 storage repository**
	- 存放 image 的位置有很多種不同的選擇，例如：一般的檔案系統、Object Storage、RADOS Block device、甚至是 Amazon S3 也可以。(但某些 repository 僅支援唯讀模式)
	
![OpenStack conceptual architecture](http://docs.openstack.org/juno/install-guide/install/apt/content/figures/1/a/common/figures/openstack_havana_conceptual_arch.png)

以上是 OpenStack 的概念架構圖，從圖中可以看出 Glance 的定位：

1. 可以將 image 存於 Swift 中
2. 提供 image 給 Nova 作為執行 VM 之用
3. 使用者可以透過 Horizon 呼叫 Glance API 來管理 image
4. 在使用 Glance API 之前，都需要通過 Keystone 的認證

---------------------------

安裝前準備工作
==============

### 建立 Glance 專用資料庫

``` bash
# 登入 MariaDB
root@controller:~# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 40
Server version: 5.5.41-MariaDB-1ubuntu0.14.04.1 (Ubuntu)

Copyright (c) 2000, 2014, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
# 建立 glance 資料庫
MariaDB [(none)]> CREATE DATABASE glance;
Query OK, 1 row affected (0.00 sec)
# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
Query OK, 0 rows affected (0.00 sec)
# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
Query OK, 0 rows affected (0.00 sec)
# 啟用權限
MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit;
Bye
```

### 切換為 admin user 權限

``` bash
root@controller:~# source ~/openstack/admin-openrc.sh
```

### 為 Glance 建立 user / role / tenant 相關權限

1、建立 user glance

``` bash
root@controller:~# keystone user-create --name glance --pass GLANCE_PASS
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |                                  |
| enabled  |               True               |
|    id    | 72e54c7716ec4bce9fd75707dec30436 |
|   name   |              glance              |
| username |              glance              |
+----------+----------------------------------+
```

2、將 user(glance)指定屬於 tenant(service)，並設定 role(admin) 權限

``` bash
root@controller:~# keystone user-role-add --user glance --tenant service --role admin
```

3、在 Keystone 註冊 Glance service

``` bash
root@controller:~# keystone service-create --name glance --type image --description "OpenStack Image Service"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |     OpenStack Image Service      |
|   enabled   |               True               |
|      id     | ac6fd120d5c045db9cd44e9ce5a2a936 |
|     name    |              glance              |
|     type    |              image               |
+-------------+----------------------------------+
```

4、在 Keystone 註冊 Glance API endpoints

``` bash
root@controller:~# keystone endpoint-create --service-id $(keystone service-list | awk '/ image / {print $2}') --publicurl http://controller:9292 --internalurl http://controller:9292 --adminurl http://controller:9292 --region regionOne
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |      http://controller:9292      |
|      id     | 72d7e7adf4ae429bad64ac8a07b58150 |
| internalurl |      http://controller:9292      |
|  publicurl  |      http://controller:9292      |
|    region   |            regionOne             |
|  service_id | ac6fd120d5c045db9cd44e9ce5a2a936 |
+-------------+----------------------------------+
```

---------------------------

安裝 Glance Service
===================

1、安裝 Glance 套件

``` bash
root@controller:~# apt-get -y install glance python-glanceclient
```

2、編輯 `/etc/glance/glance-api.conf`，並將以下部分進行修改：

``` bash
# 資料庫設定
[database]
connection = mysql://glance:GLANCE_DBPASS@controller/glance

# 設定 keystone 認證授權相關設定
[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = glance
admin_password = GLANCE_PASS

[paste_deploy]
flavor = keystone

[glance_store]
filesystem_store_datadir = /var/lib/glance/images/	# images 檔案預設存放位置

[DEFAULT]
verbose = True	# 顯示 debug 相關訊息
default_store = file	# 預設儲存本地檔案系統
notification_driver = noop	# 關閉通知功能(與 Ceilometer 有關)
```

3、編輯 `/etc/glance/glance-registry.conf`，並將以下部分進行修改：

``` bash
[database]
connection = mysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = glance
admin_password = GLANCE_PASS
 
[paste_deploy]
flavor = keystone

[DEFAULT]
verbose = True
notification_driver = noop
```

4、將設定資料佈署到資料庫中

``` bash
root@controller:~# su -s /bin/sh -c "glance-manage db_sync" glance
```

5、重新啟動 Glance service

``` bash
root@controller:~# service glance-registry restart
root@controller:~# service glance-api restart
```

---------------------------

驗證安裝是否成功
================

完成 Glance 安裝之後，接下來要驗證是否有正確安裝完成。

以下將會下載一個 cirros 的映像檔，將其上傳 & 註冊到 Glance，最後再查詢是否有上傳成功!

``` bash
# 建立資料夾並下載 Linux OS image
root@controller:~# mkdir /tmp/images
root@controller:~# cd /tmp/images
root@controller:~# wget http://cdn.download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img

# 切換為 admin user
root@controller:~# source ~/openstack/admin-openrc.sh

# 將檔案上傳至 Glance 並註冊
root@controller:~# glance image-create --name "cirros-0.3.3-x86_64" --file cirros-0.3.3-x86_64-disk.img --disk-format qcow2 --container-format bare --is-public True --progress
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 133eae9fb1c98f45894a4e60d8736619     |
| container_format | bare                                 |
| created_at       | 2015-02-11T03:59:43                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | e37ba2d2-b9c1-42fb-8e81-5336efb0c164 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.3-x86_64                  |
| owner            | 095227f7415a4bdc9a4f48f620208fd7     |
| protected        | False                                |
| size             | 13200896                             |
| status           | active                               |
| updated_at       | 2015-02-11T03:59:43                  |
| virtual_size     | None                                 |
+------------------+--------------------------------------+

# 檢查 image 存放目錄，是否有相對應的檔案
root@controller:~# ls -al /var/lib/glance/images/
total 12900
drwxr-xr-x 2 glance glance     4096 Feb 11 11:59 .
drwxr-xr-x 4 glance glance     4096 Feb 11 11:35 ..
-rw-r----- 1 glance glance 13200896 Feb 11 11:59 e37ba2d2-b9c1-42fb-8e81-5336efb0c164

# 列出 image 清單，確認是否有上傳成功
root@controller:~# glance image-list
+--------------------------------------+---------------------+-------------+------------------+----------+--------+
| ID                                   | Name                | Disk Format | Container Format | Size     | Status |
+--------------------------------------+---------------------+-------------+------------------+----------+--------+
| e37ba2d2-b9c1-42fb-8e81-5336efb0c164 | cirros-0.3.3-x86_64 | qcow2       | bare             | 13200896 | active |
+--------------------------------------+---------------------+-------------+------------------+----------+--------+

# 確認無誤，移除 temp directory
root@controller:~# rm -rf /tmp/images
```

---------------------------

參考資料
========

- [OpenStack Installation Guide for Ubuntu 14.04 - juno][1]

[1]:  http://docs.openstack.org/juno/install-guide/install/apt/content/