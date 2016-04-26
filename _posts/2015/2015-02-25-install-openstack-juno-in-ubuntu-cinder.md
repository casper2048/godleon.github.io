---
layout: post
title:  "[OpenStack] 安裝 Juno @ Ubuntu 14.04 (7) - 設定 Block Storage Service(Cinder)"
description: "安裝 dashboard service(Horizon)，讓管理者可以透過 Web GUI 的方式管理 OpenStack resources & services"
date:   2015-02-25 16:00:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Ubuntu, Linux]
---

Block Storage Service(Cinder) 概觀
==================================

OpenStack Block Storage(Cinder) 的工作包含 volume、volume 快照、volume 類型的管理，主要用來提供給 VM 區塊等級(Block Level)的永久性儲存空間，提供快照、資料回復...等功能。還可以整合其他商業化的企業儲存平台，例如

實際提供儲存空間的 storage node，可以是本地端的區塊級(block-level)儲存設備，也可以透過 SAN/NAS 的方式由外部的設備提供，因此可以整合其他商業化的企業儲存平台，例如 NetApp、Nexenta ... 等不同廠商的解決方案。

-------------------------------------

Cinder 服務元件架構
====================

在安裝之前，可以先用以下這張圖來了解一下 Cinder 是由那些 component 組合而成的(截錄自[工研院-OpenStack Cinder Tutorial][1] 一文)：

![Cinder Interaction](https://lh3.googleusercontent.com/-1O0C3CEdZA8/VDI21yfyTRI/AAAAAAAAJr0/bp9lRkWeeVo/w944-h690-no/Cinder_interaction.png)

從上圖可看出，Cinder 共包含了以下重要的部分：

### cinder-api

用來接受來自外部對於 volume 空間的請求後，透過 message queue 將請求轉給 cinder-scheduler 後再轉給 cinder-volume 進行後續處理。

### cinder-scheduler daemon

類似 nova-schedular 的角色，接收到來自 message queue 的命令後，會從多個(如果有)提供 block storage 服務的 node 挑選出一個最合適的 node 來建立 volume。

### cinder-volume

cinder-volume 的工作大概有幾項：

1. 接收來自 cinder-scheduler 的命令，直接存取 Block Storage Service(可能由本地 or 外部設備提供)，建立新的 volume。
2. 接收來自 message queue 的訊息，進行 volume 空間的讀寫。
3. 透過不同的 driver，還可以使用多種不同的 storage provider 所提供的設備。

### message queue

負責將訊息在不同的 Block Storage 程序間進行派送。

-------------------------------------

安裝 & 設定 @ Controller Node
=============================

在 Cinder 服務中，所有的控制動作都是由 Controller Node 發起的，因此首先要在 **Controller Node** 上進行管理相關套件的安裝 & 設定

### 安裝前的準備工作 @ Controller Node

#### 1、建立 Cinder 資料庫 & 設定權限

``` bash
# 登入 MariaDB
root@controller:~# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 11960
Server version: 5.5.41-MariaDB-1ubuntu0.14.04.1 (Ubuntu)

Copyright (c) 2000, 2014, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

# 建立 Database
MariaDB [(none)]> CREATE DATABASE cinder;
Query OK, 1 row affected (0.00 sec)

# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
Query OK, 0 rows affected (0.00 sec)

# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
Query OK, 0 rows affected (0.00 sec)

# 將權限寫入資料庫
MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit;
Bye
```

#### 2、切換為 admin user 權限

``` bash
root@controller:~# source ~/openstack/admin-openrc.sh
```

#### 3、為 Cinder 建立 user / role / tenant 相關權限

##### 3.1 建立 user cinder

``` bash
root@controller:~# keystone user-create --name cinder --pass CINDER_PASS
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |                                  |
| enabled  |               True               |
|    id    | 28c1d442022e47fdaeb681e9201c0114 |
|   name   |              cinder              |
| username |              cinder              |
+----------+----------------------------------+
```

##### 3.2 將 user(cinder)指定屬於 tenant(service)，並設定 role(admin) 權限

``` bash
root@controller:~# keystone user-role-add --user cinder --tenant service --role admin
```

##### 3.3 在 Keystone 註冊 Cinder service

``` bash
# API version 1
root@controller:~# keystone service-create --name cinder --type volume --description "OpenStack Block Storage"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |     OpenStack Block Storage      |
|   enabled   |               True               |
|      id     | 28e5cd684d8c4e00ab829921011ce22f |
|     name    |              cinder              |
|     type    |              volume              |
+-------------+----------------------------------+

# API version 2
root@controller:~# keystone service-create --name cinderv2 --type volumev2 --description "OpenStack Block Storage"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |     OpenStack Block Storage      |
|   enabled   |               True               |
|      id     | b4d65521f031466689abb4379d090659 |
|     name    |             cinderv2             |
|     type    |             volumev2             |
+-------------+----------------------------------+
```

##### 3.4 在 Keystone 註冊 Cinder API endpoints

``` bash
# API version 1 endpoint
root@controller:~# keystone endpoint-create --service-id $(keystone service-list | awk '/ volume / {print $2}') --publicurl http://controller:8776/v1/%\(tenant_id\)s --internalurl http://controller:8776/v1/%\(tenant_id\)s --adminurl http://controller:8776/v1/%\(tenant_id\)s --region regionOne
+-------------+-----------------------------------------+
|   Property  |                  Value                  |
+-------------+-----------------------------------------+
|   adminurl  | http://controller:8776/v1/%(tenant_id)s |
|      id     |     5d0f4159b7624b888c89e5ce8ccbd51a    |
| internalurl | http://controller:8776/v1/%(tenant_id)s |
|  publicurl  | http://controller:8776/v1/%(tenant_id)s |
|    region   |                regionOne                |
|  service_id |     28e5cd684d8c4e00ab829921011ce22f    |
+-------------+-----------------------------------------+

# API version 2 endpoint
root@controller:~# keystone endpoint-create --service-id $(keystone service-list | awk '/ volumev2 / {print $2}') --publicurl http://controller:8776/v2/%\(tenant_id\)s --internalurl http://controller:8776/v2/%\(tenant_id\)s --adminurl http://controller:8776/v2/%\(tenant_id\)s --region regionOne
+-------------+-----------------------------------------+
|   Property  |                  Value                  |
+-------------+-----------------------------------------+
|   adminurl  | http://controller:8776/v2/%(tenant_id)s |
|      id     |     69fd9becf6bf487c8b181737dd573c0b    |
| internalurl | http://controller:8776/v2/%(tenant_id)s |
|  publicurl  | http://controller:8776/v2/%(tenant_id)s |
|    region   |                regionOne                |
|  service_id |     b4d65521f031466689abb4379d090659    |
+-------------+-----------------------------------------+
```

### 安裝 & 設定 Cinder 管理套件

#### 1、安裝套件

``` bash
root@controller:~# apt-get -y install cinder-api cinder-scheduler python-cinderclient
```

#### 2、修改設定檔 `/etc/cinder/cinder.conf` 如下：

``` ini
[DEFAULT]
......
# 指定使用 keystone 進行認證
auth_strategy = keystone
# message broker 設定
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = RABBIT_PASS

# 指定 controller node ip
my_ip = 10.0.0.11

# Keystone service 相關設定
[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = cinder
admin_password = CINDER_PASS

[database]
connection = mysql://cinder:CINDER_DBPASS@controller/cinder
```

#### 3、佈署 cinder 資料庫

``` bash
root@controller:~# su -s /bin/sh -c "cinder-manage db sync" cinder
```

### 重新啟動 Cinder 相關管理服務

``` bash
root@controller:~# service cinder-scheduler restart
root@controller:~# service cinder-api restart
```

### 移除不需要的 sqlite 資料庫

``` bash
root@controller:~# rm -f /var/lib/cinder/cinder.sqlite
```

-------------------------------------

安裝 & 設定 @ Storage Node
===========================

### Block Storage Node 1 的 Disk 配置狀況

``` bash
root@block1:~# fdisk -l

Disk /dev/sda: 68.7 GB, 68719476736 bytes
255 heads, 63 sectors/track, 8354 cylinders, total 134217728 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000abe

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048    31457279    15727616   83  Linux
/dev/sda2        31459326    33552383     1046529    5  Extended
/dev/sda5        31459328    33552383     1046528   82  Linux swap / Solaris

Disk /dev/sdb: 68.7 GB, 68719476736 bytes
171 heads, 8 sectors/track, 98112 cylinders, total 134217728 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x315eec44

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   134217727    67107840   83  Linux
```

以下主要以 /dev/sdb1 作為存放 volume 的空間之用。

### LVM 設定

#### 1、安裝 LVM 套件

``` bash
root@block1:~# apt-get -y install lvm2
```

#### 2、建立 Physical Volume

``` bash
root@block1:~# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created
```

#### 3、建立 Volume Group

``` bash
root@block1:~# vgcreate cinder-volumes /dev/sdb1
  Volume group "cinder-volumes" successfully created
```

#### 4、調整 LVM 設定，僅使用 /dev/sdb

修改 LVM 設定檔 `/etc/lvm/lvm.conf`，並調整 filter 設定如下：

``` bash
filter = [ "a/sdb/", "r/.*/"]
```

以上設定表示限定只使用 /dev/sdb 這顆硬碟。


### 安裝 & 設定 Cinder 相關套件

#### 安裝套件

``` bash
root@block1:~# apt-get -y install cinder-volume python-mysqldb
```

#### 修改設定

修改設定檔 `/etc/cinder/cinder.conf`，調整內容如下：

``` ini
[DEFAULT]
...
verbose = True
# 使用 Keystone 進行認證
auth_strategy = keystone
# message broker 相關設定
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = RABBIT_PASS
# block storage node 的 ip address
my_ip = 10.0.0.41
# image service host
glance_host = controller

# Keystone 相關設定
[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = cinder
admin_password = CINDER_PASS

# 資料庫相關設定
[database]
connection = mysql://cinder:CINDER_DBPASS@controller/cinder
```

### 重新啟動相關服務

``` bash
root@block1:~# service tgt restart
root@block1:~# service cinder-volume restart
```

### 移除不需要的 sqlite 資料庫

``` bash
root@block1:~# rm -f /var/lib/cinder/cinder.sqlite
```

-------------------------------------

Cinder 功能驗證
===============

### 建立 Volume

以 admin 身分執行：

``` bash
# 切換成 admin
root@controller:~# source ~/openstack/admin-openrc.sh

# 檢視 cinder 相關服務清單
root@controller:~# cinder service-list
+------------------+------------+------+---------+-------+----------------------------+-----------------+
|      Binary      |    Host    | Zone |  Status | State |         Updated_at         | Disabled Reason |
+------------------+------------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler | controller | nova | enabled |   up  | 2015-02-25T09:29:19.000000 |       None      |
|  cinder-volume   |   block1   | nova | enabled |   up  | 2015-02-25T09:29:24.000000 |       None      |
+------------------+------------+------+---------+-------+----------------------------+-----------------+
```

以 demo 身分執行：

``` bash
# 切換成 demo user
root@controller:~# source ~/openstack/demo-openrc.sh

# 建立 1GB volume (名稱為 demo-volume1)
root@controller:~# cinder create --display-name demo-volume1 1
+---------------------+--------------------------------------+
|       Property      |                Value                 |
+---------------------+--------------------------------------+
|     attachments     |                  []                  |
|  availability_zone  |                 nova                 |
|       bootable      |                false                 |
|      created_at     |      2015-02-25T09:30:03.875463      |
| display_description |                 None                 |
|     display_name    |             demo-volume1             |
|      encrypted      |                False                 |
|          id         | 3631bc9d-fd4b-4612-869f-f0604dbd4436 |
|       metadata      |                  {}                  |
|         size        |                  1                   |
|     snapshot_id     |                 None                 |
|     source_volid    |                 None                 |
|        status       |               creating               |
|     volume_type     |                 None                 |
+---------------------+--------------------------------------+

# 檢視 cinder volume 清單
root@controller:~# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 3631bc9d-fd4b-4612-869f-f0604dbd4436 | available | demo-volume1 |  1   |     None    |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```

看到以上類似的內容，表示 Cinder 服務就安裝成功囉!

### 附加 Volume 到 VM instance 上

在[之前的文章](http://godleon.github.io/blog/2015/02/14/launch-an-instance-in-openstack/)中，有曾經使用 demo 的身分啟動一個 VM instance，名稱為 **demo-instance1**，接著我們將剛剛產生的 volume 附加到這個 VM 上：

``` bash
# 切換成 demo user
root@controller:~# source ~/openstack/demo-openrc.sh

# 顯示目前的 VM instance 清單
root@controller:~# nova list
+--------------------------------------+----------------+--------+------------+-------------+-----------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks              |
+--------------------------------------+----------------+--------+------------+-------------+-----------------------+
| 9dce2984-7c49-43db-a733-f64cbb125992 | demo-instance1 | ACTIVE | -          | Running     | demo-net=192.168.50.2 |
+--------------------------------------+----------------+--------+------------+-------------+-----------------------+

# 顯示目前的 volume 清單
root@controller:~# nova volume-list
+--------------------------------------+-----------+--------------+------+-------------+-------------+
| ID                                   | Status    | Display Name | Size | Volume Type | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+-------------+
| 3631bc9d-fd4b-4612-869f-f0604dbd4436 | available | demo-volume1 | 1    | None        |             |
+--------------------------------------+-----------+--------------+------+-------------+-------------+

# 將 volume(demo-volume1) 附加到 VM instance() 上
root@controller:~# nova volume-attach demo-instance1 3631bc9d-fd4b-4612-869f-f0604dbd4436
+----------+--------------------------------------+
| Property | Value                                |
+----------+--------------------------------------+
| device   | /dev/vdb                             |
| id       | 3631bc9d-fd4b-4612-869f-f0604dbd4436 |
| serverId | 9dce2984-7c49-43db-a733-f64cbb125992 |
| volumeId | 3631bc9d-fd4b-4612-869f-f0604dbd4436 |
+----------+--------------------------------------+
```

登入到 Horizon，看到以下畫面，就可以看到類似以下畫面：

![Attach volume to VM instance](https://lh3.googleusercontent.com/-CT92TVVcrnA/VO2Z4b9juGI/AAAAAAAAKAY/GywP2DtygHA/w957-h526-no/openstack_attach_volume_to_instance.png)

-------------------------------------

參考資料
========

- [OpenStack Installation Guide for Ubuntu 14.04 - juno][3]

[1]: https://tw.pycon.org/2013/site_media/media/proposal_files/cinder_2013.pdf
[2]: http://blog.nuface.tw/?p=1267
[3]: http://docs.openstack.org/juno/install-guide/install/apt/content/