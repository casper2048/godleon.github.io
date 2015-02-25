---
layout: post
title:  "[OpenStack] 安裝 Juno @ Ubuntu 14.04 (1) - 基本環境設定"
date:   2015-02-10 16:00:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Ubuntu, Linux]
---

硬體設定說明
================

### 簡介

這次 OpenStack 的安裝過程中，在網路的部分選擇使用 **Neutron** 來管理，不使用 **nova-network**，因此需要額外一個 Network Node 來處理網路的工作

此外，還要加上單獨的 Block Storage Node & Object Storage Node，因此一共會有五個 node (Contoller + Network + Compute + Block Storage + Object Storage)。


### 硬體需求

這邊預計用以下硬體來進行安裝，一共有五台機器：

1. Controller Node
> 2 CPU + 4GB RAM + 16GB HD + 1 NIC

2. Network Node
> 1 CPU + 1GB RAM + 16GB HD + 3 NIC

3. Compute Node
> 4 CPU + 8GB RAM + 32GB HD + 3 NIC

4. Block Storage Node
> 1 CPU + 2GB RAM + 2 x 64GB HD + 2 NIC

5. Object Storage Node
> 1 CPU + 2GB RAM + 32GB HD + 2 NIC

### Service Layout

由於 OpenStack 的 service 眾多且互相緊密關聯，因此每個 node 要裝甚麼 service 可是要先搞清楚，否則之後裝起來無法正常運作時，會不容易找到問題所在。

以下是預計要安裝的 service layout：

![Minimal architecture example with OpenStack Networking (neutron)—Service layout](http://docs.openstack.org/juno/install-guide/install/apt/content/figures/1/a/common/figures/installguidearch-neutron-services.png)

---------------------------

安全設定
========

由於 OpenStack 的服務眾多，因此在安裝過程中所需要設定的密碼種類也是相當驚人的，以下列出所需要的密碼種類：

| Password name | Description |
|---------------|-------------|
| *`DB_ROOT_PASS`* | 資料庫的 root password |
| *`RABBIT_PASS`* | Password of user guest of RabbitMQ |
| *`KEYSTONE_DBPASS`* | Database password of Identity service |
| *`DEMO_PASS`* | Password of user demo |
| *`ADMIN_PASS`* | Password of user admin |
| *`GLANCE_DBPASS`* | Database password for Image Service | 
| *`GLANCE_PASS`* | Password of Image Service user **glance** |
| *`NOVA_DBPASS`* | Database password for Compute service |
| *`NOVA_PASS`* | Password of Compute service user **nova** |
| *`DASH_DBPASS`* | Database password for the dashboard |
| *`CINDER_DBPASS`* | Database password for the Block Storage service |
| *`CINDER_PASS`* | Password of Block Storage service user **cinder** |
| *`NEUTRON_DBPASS`* | Database password for the Networking service |
| *`NEUTRON_PASS`* | Password of Networking service user **neutron** |
| *`HEAT_DBPASS`* | Database password for the Orchestration service |
| *`HEAT_PASS`* | Password of Orchestration service user **heat** |
| *`CEILOMETER_DBPASS`* | Database password for the Telemetry service |
| *`CEILOMETER_PASS`* | Password of Telemetry service user **ceilometer** |
| *`TROVE_DBPASS`* | Database password of Database service |
| *`TROVE_PASS`* | Password of Database Service user **trove** |

---------------------------

網路設定
========

### IP Address 設定

接著是每個  node 的網路設定：

![Minimal architecture example with OpenStack Networking (neutron)—Network layout](http://docs.openstack.org/juno/install-guide/install/apt/content/figures/1/a/common/figures/installguidearch-neutron-networks.png)

上圖是從 OpenStaock 官網的 document 引用過來的，與目前所要進行的環境有所差別，差異如下：

1. Object Storage Node 只會有一個，所以`沒有 Object Storage Node 2`

2. External network 的設定並非 203.0.113.0/24，而是`要改成在自己環境中可以連上 internet 的 ip 設定`

除了上述兩點之外，其他設定皆相同，包含三個網段(Management/Tunnel/Storage Network)，可以接在同一個 switch 上用不同 VLAN 切開，也可以用不同的實體 switch 將其分開。

在我的環境中，我執行在 vSphere ESXi 上，因此只要設定 vSwitch，使用 VLAN 進行區隔，並將 [promiscuous mode](http://cyrilwang.blogspot.tw/2012/08/vmware-promiscuous-mode.html) 開啟即可。


### DNS 設定

進入到每一個 node，修改 `/etc/hosts`，加入以下內容：

``` bash
10.0.0.11	controller
10.0.0.21	network
10.0.0.31	compute1
10.0.0.41	block1
10.0.0.51	object1
```

並確認可以相互 ping 的到。

---------------------------

安裝 NTP service
================

在雲端架構中，每一台機器的時間都需要同步，有兩個部分需要完成：

### 設定 Controller Node 為 NTP server

1、安裝 ntp 套件

``` bash
root@contoller:~# apt-get update; apt-get -y install ntp
```

2、修改 `/etc/ntp.conf`，進行以下設定

``` bash
# 設定 tock.stdtime.gov.tw 為參照的 time server
# 並將其他 server 開頭的設定進行註解
server tock.stdtime.gov.tw iburst

# 修改 restrict 設定
restrict -4 default kod notrap nomodify
restrict -6 default kod notrap nomodify
```

3、重新啟動 ntp servive

``` bash
root@contoller:~# service ntp restart
```

### 設定其他 Node 的 NTP service

設定完 controller node 之後，我們把其他 node 的 ntp 設定指向 controller node，在其他 node 執行以下步驟：

1、安裝 ntp 套件

``` bash
contoller:~# apt-get update; apt-get -y install ntp
```

2、修改 `/etc/ntp.conf`，進行以下設定

``` bash
# 設定 controller 為參照的 time server
# 並將其他 server 開頭的設定進行註解
server controller iburst
```

3、重新啟動 ntp servive

``` bash
root@contoller:~# service ntp restart
```

### 驗證 NTP 是否正確安裝

可在其他 node 上執行下列指令：

``` bash
root@other_nodes:~# ntpq -c peers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 controller      211.22.103.157   3 u   17   64    0    0.000    0.000   0.000
```

若是有類似上面的結果出現，表示設定成功囉!

---------------------------

設定 Repository
=========================

接著為了之後可以透過 apt 來安裝管理套件，以下我們將 openstack 的套件庫加入：

``` bash
root@all_nodes:~# apt-get install ubuntu-cloud-keyring
root@all_nodes:~# echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu trusty-updates/juno main" > /etc/apt/sources.list.d/cloudarchive-juno.list
root@all_nodes:~# apt-get update && apt-get -y dist-upgrade
```

---------------------------

安裝資料庫
==========

在 openstack 中很多資訊會儲存在資料庫做為管理之用，因此在最重要的 controller node 上必須安裝好資料庫

而由於 MySQL 已經被 Oracle 收購，非完全免費的軟體，所以改用 MariaDB 取而代之

1、安裝 MaraiDB

``` bash
root@controller:~# apt-get -y install mariadb-server python-mysqldb
```

2、修改 `/etc/mysql/my.cnf`

``` bash
[mysqld]
# 修改 bind-address 設定
bind-address = 10.0.0.11

# 加入以下 UTF-8 的相關設定
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```

3、重新啟動 db service

``` bash
root@controller:~# service mysql restart
```

---------------------------

安裝 Messaging Server
=====================

openstack 透過 message broker 來進行各個服務之前操作 & 狀態訊息的交換，由於這些訊息與各服務的統籌管理有很大關係，因此都會集中到 controller node 上的 message broker service 來進行。

openstack 支援多種不同的 message broker，例如：[RabbitMQ][2]、[Apache Qpid][3]、[ZeroMQ][4] .... 等等。

這安裝環境中使用 [RabbitMQ][2] 作為 message broker，將 [RabbitMQ][2] 安裝在 controller node 上：

``` bash
root@controller:~# apt-get -y install rabbitmq-server
```

安裝好 RabbitMQ 之後，會設定一位預設的使用者 **guest**，為了之後設定方便，其他的服務在 message broker 的設定都用 guest 這個身分，所以這裡透過以下命令設定 guest 的密碼：

``` bash
# 「RABBIT_PASS」請自行修改為自訂的密碼
root@controller:~# rabbitmqctl change_password guest RABBIT_PASS
```

---------------------------

>

完成以上落落長的設定後，基本的設定就大致完成了，接著後面就是把各個服務(Keystone、Glance、Nova、Neutron ... 等等)一個一個給安裝起來囉!


參考資料
========

- [OpenStack Installation Guide for Ubuntu 14.04 - juno][1]

[1]:  http://docs.openstack.org/juno/install-guide/install/apt/content/
[2]:  http://www.rabbitmq.com/
[3]:  http://qpid.apache.org/
[4]:  http://zeromq.org/