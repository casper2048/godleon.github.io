---
layout: post
title:  "[OpenStack] 安裝 Juno @ Ubuntu 14.04 (4) - 設定 Compute Service(Nova)"
date:   2015-02-11 19:40:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Ubuntu, Linux]
---

Compute Service 概觀
=========================

Compute Service 在整個 IaaS 的架構中是屬於最主要的部份，同時會向 Identity Service 進行認證授權、向 Image Service 要求 image、將資料提供給 Dashboard .... 等等。

而整個 Compute Service (NOVA) 包含了以下幾個重要部分：

### API

#### nova-api (service)

用來處理 & 回應終端使用者的 Compute API call，除了支援 OpenStack API 之外，還支援了 Amazon EC2 以及管理用 API。

像是啟動 instance 就是由 nova-api service 所執行。

#### nova-api-metadata (service)

通常會用在多個 compute node 並安裝 nova-network 時使用，用來回應來自 instance 對 metadata 的要求。


### Compute core

這元件包含了幾個重要的 daemon，來共同組成 Compute Core 以提供服務：

#### nova-compute (service)

主要目的是與 hypervisor API 溝通，已建立 / 中止 VM instance。例如：

- XenAPI for XenServer/XCP

- libvirt for KVM or QEMU

- VMwareAPI for VMware

簡單來說，這個 daemon 僅接受來自 Message Queue 的訊息並執行相關命令而已，例如啟動一個 KVM instance、更新 instance 在資料庫中的狀態。

#### nova-scheduler (service)

取得 VM instance 的需求後，根據制定的規則 & 目前情況，決定要讓 VM 在哪一台實體主機啟動。

#### nova-conductor (module)

作為 nova-compute 與 DB 之間的橋樑，目的是為了減少 nova-compute 直接存取 DB 的行為。

> 不要將 nova-conductor 與 nova-compute service 安裝在同一個 node 上。

### Networking for VMs

這個部分的功能已經被移進獨立的 Network Service (Neutron) 了，但還是可以簡單說明一下：

#### nova-network (daemon)

同樣也是一支 daemon，接收來自 Message Queue 的網路相關需求並處理，例如：設定 bridge 介面 or iptables 規則 .... 等等。


### Console interface

#### nova-novncproxy (daemon)

作為透過 VNC 連線時存取 instance VM 的 proxy 之用，支援 browser-based novnc client。

#### nova-xvpnvncproxy (daemon)

作為透過 VNC 連線時存取 instance VM 的 proxy 之用，支援 Java client 的連線。

#### nova-spicehtml5proxy (daemon)

作為透過 [SPICE](http://www.spice-space.org/) 通訊協定連線存取 instance 的 proxy 之用，支援 browser-based HTML5 client。

#### nova-consoleauth (daemon)

用來驗證由 console proxy (上面的 **nova-novncproxy** & **nova-xvpnvncproxy**) 所提供的使用者 token。

#### nova-cert (daemon)

與 x509 憑證處理相關。


### Image management (在 AWS EC2 情境)

#### nova-objectstore (daemon)

提供一個等同於 AWS S3 的介面給 OpenStack Image Service 進行註冊，主要用於一定要支援 [euca2ools](https://www.eucalyptus.com/download/euca2ools) 的前提下。

nova-objectstore 的用途主要將來自 euca2ools 的 S3 request 轉換為 Image Service request。

#### euca2ools (client)

用來管理 AWS 雲端資源的工具，但由於功能強大，可以用在 OpenStack 中相容 AWS 的服務中。


### Command-line clients and other interfaces

#### nova client

允許使用者以 tenant 管理者 or 一般使用者的身分執行命令。


### Other components

#### Message Queue

任何實作 AMQP 的 message queue 服務，用來作為所有 daemon 通訊的中介，透過這個服務，可以降低 OpenStack 眾多服務之間溝通的複雜度，這邊我們所使用的 [RabbitMQ](http://www.rabbitmq.com/)。

#### SQL Database

儲存在 OpenStack 上運行的所有 VM instance 的狀態、網路設定、相關專案，任何 SQLAlchemy 支援的 DB 都可以使用，我們這邊所使用的是 MariaDB。

-----------------------------

安裝前的準備工作
================

以下動作皆在 controller node 上完成：

### 建立資料庫

``` bash
# 登入資料庫
root@controller:~# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 47
Server version: 5.5.41-MariaDB-1ubuntu0.14.04.1 (Ubuntu)

Copyright (c) 2000, 2014, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

# 建立 nova 資料庫
MariaDB [(none)]> CREATE DATABASE nova;
Query OK, 1 row affected (0.00 sec)

# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
Query OK, 0 rows affected (0.00 sec)

# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
Query OK, 0 rows affected (0.00 sec)

# 將權限設定寫入資料庫
MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit
Bye
```

### 切換為使用者 admin

``` bash
root@controller:~# source ~/openstack/admin-openrc.sh
```

### 建立 user(nova)

1、建立 user(nova)

``` bash
# 建立 user(nova)
root@controller:~# keystone user-create --name nova --pass NOVA_PASS
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |                                  |
| enabled  |               True               |
|    id    | 797bb152eeb4429aa8824b5d3e192600 |
|   name   |               nova               |
| username |               nova               |
+----------+----------------------------------+
```

2、將 user(nova)指定屬於 tenant(service)，並設定 role(admin) 權限

``` bash
root@controller:~# keystone user-role-add --user nova --tenant service --role admin
```

3、在 Keystone 註冊 Nova service

``` bash
root@controller:~# keystone service-create --name nova --type compute --description "OpenStack Compute"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |        OpenStack Compute         |
|   enabled   |               True               |
|      id     | 00a8f09816dc4306bc122ee14c67fec8 |
|     name    |               nova               |
|     type    |             compute              |
+-------------+----------------------------------+
```

4、在 Keystone 註冊 Nova API endpoints

``` bash
root@controller:~# keystone endpoint-create --service-id $(keystone service-list | awk '/ compute / {print $2}') --publicurl http://controller:8774/v2/%\(tenant_id\)s --internalurl http://controller:8774/v2/%\(tenant_id\)s --adminurl http://controller:8774/v2/%\(tenant_id\)s --region regionOne
+-------------+-----------------------------------------+
|   Property  |                  Value                  |
+-------------+-----------------------------------------+
|   adminurl  | http://controller:8774/v2/%(tenant_id)s |
|      id     |     337d2fec65a4461fab3246a789a4c353    |
| internalurl | http://controller:8774/v2/%(tenant_id)s |
|  publicurl  | http://controller:8774/v2/%(tenant_id)s |
|    region   |                regionOne                |
|  service_id |     00a8f09816dc4306bc122ee14c67fec8    |
+-------------+-----------------------------------------+
```

-----------------------------

安裝 Compute Service
====================

Compute Service 其實是由好幾個 service 一同合作，讓使用者可以啟用 VM instance。

安裝的時候可以將這些 service 安裝在不同的機器上，也可以都裝在相同的機器上；在這邊我們將大部分的 service 安裝在 controller node 上，而主要啟動 VM instance 的服務則安裝在 compute node 上。

### 在 Controller Node 上安裝相關套件

1、安裝套件

``` bash
root@controller:~# apt-get -y install nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-novaclient
```

2、編輯 `/etc/nova/nova.conf`，並將以下部分進行修改：

``` bash
[database]
connection = mysql://nova:NOVA_DBPASS@controller/nova	# nova 資料庫連線設定

[DEFAULT]
verbose = True	# 顯示 debug 訊息
rpc_backend = rabbit	# Message Broker 相關設定
rabbit_host = controller
rabbit_password = RABBIT_PASS
auth_strategy = keystone	# 指定透過 keystone 進行認證授權
my_ip = 10.0.0.11	# 設定管理介面 IP
vncserver_listen = 0.0.0.0	# 設定 vnc server listen source
vncserver_proxyclient_address = 10.0.0.11	# 設定 vnc proxy IP

# keystone 相關設定
[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = nova
admin_password = NOVA_PASS

# 設定提供 image service 的 host 位置
[glance]
host = controller
```

3、將設定資料佈署到資料庫中

``` bash
root@controller:~# su -s /bin/sh -c "nova-manage db sync" nova
```

4、完成設定，重啟相關服務

``` bash
root@controller:~# service nova-api restart
root@controller:~# service nova-cert restart
root@controller:~# service nova-consoleauth restart
root@controller:~# service nova-scheduler restart
root@controller:~# service nova-conductor restart
root@controller:~# service nova-novncproxy restart
```

5、移除不需要的 nova.sqlite 檔案

``` bash
root@controller:~# rm -f /var/lib/nova/nova.sqlite
```

### 在 Compute Node 上安裝相關套件

1、安裝 Nova 相關套件

``` bash
root@compute1:~# apt-get -y install nova-compute sysfsutils
```

2、編輯 `/etc/nova/nova.conf`，並將以下部分進行修改：

``` bash
[DEFAULT]
verbose = True	# 顯示 Debug 資訊
rpc_backend = rabbit	# Message Broker 設定
rabbit_host = controller
rabbit_password = RABBIT_PASS
auth_strategy = keystone	# 指定 keystone 作為認證授權之用
my_ip = 10.0.0.31	# 設定管理介面 IP
vnc_enabled = True	# 設定啟用 vnc
vncserver_listen = 0.0.0.0	# 設定 vnc server listen source
vncserver_proxyclient_address = 10.0.0.31	# 設定 vnc proxy ip
novncproxy_base_url = http://controller:6080/vnc_auto.html	# 設定 vnc url

# keystone 相關設定
[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = nova
admin_password = NOVA_PASS

# 設定提供 image service 的 host 位置
[glance]
host = controller
```

3、檢查 Compute Node 是否支援硬體加速

``` bash
root@compute1:~# egrep -c '(vmx|svm)' /proc/cpuinfo
4
```

若是結果為 0，表示不支援硬體加速；若是結果大於零 0，表示支援硬體加速。

編輯 `/etc/nova/nova-compute.conf`：

``` bash
[libvirt]
virt_type=kvm
```

若是不支援硬體加速，則改成如下：

``` bash
[libvirt]
virt_type=qemu
```

4、重新啟動 Compute Service

``` bash
root@compute1:~# service nova-compute restart
```

5、移除不需要的 nova.sqlite 檔案

``` bash
root@controller:~# rm -f /var/lib/nova/nova.sqlite
```


---------------------------

驗證安裝是否成功
================

上述套件在 controller node & compute node 都安裝設定完成後，最後要來驗證是否安裝成功。

1、回到 controller node，切換至 user admin

``` bash
root@controller:~# source ~/openstack/admin-openrc.sh
```

2、列出 nova service list

``` bash
root@controller:~# nova service-list
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host       | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-cert        | controller | internal | enabled | up    | 2015-02-11T08:58:09.000000 | -               |
| 2  | nova-consoleauth | controller | internal | enabled | up    | 2015-02-11T08:58:14.000000 | -               |
| 3  | nova-scheduler   | controller | internal | enabled | up    | 2015-02-11T08:58:08.000000 | -               |
| 5  | nova-conductor   | controller | internal | enabled | up    | 2015-02-11T08:58:14.000000 | -               |
| 6  | nova-compute     | compute1   | nova     | enabled | up    | 2015-02-11T08:58:17.000000 | -               |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
# 從上面可看出 nova-compute 位於 compute1 上，其他 service 位於 controller 上
```

3、列出可用的 image list，驗證 Identity Service(Keystone) & Image Service(Glance) 正確互通

``` bash
root@controller:~# nova image-list
+--------------------------------------+---------------------+--------+--------+
| ID                                   | Name                | Status | Server |
+--------------------------------------+---------------------+--------+--------+
| e37ba2d2-b9c1-42fb-8e81-5336efb0c164 | cirros-0.3.3-x86_64 | ACTIVE |        |
+--------------------------------------+---------------------+--------+--------+
```


---------------------------

參考資料
========

- [OpenStack Installation Guide for Ubuntu 14.04 - juno][1]

[1]:  http://docs.openstack.org/juno/install-guide/install/apt/content/