---
layout: post
title:  "[Cluster] 在 Ubuntu 上安裝 Percona XtraDB Cluster"
description: "在這邊文章中，介紹如何在 Ubuntu 14.04(trusty64) 上安裝 Percona XtraDB Cluster"
date: 2015-05-28 13:20:00
published: true
comments: true
categories: [cluster]
tags: [Cluster, Database, Linux, Ubuntu]
---


前言
====

其實研究這個 topic 是為了之後要把這個功能設定到 OpenStack 的環境之中而準備的~ XD

在 OpenStack 中資料庫是非常重要的一個服務，因此資料庫要有 HA 的保護自然也是不在話下。

### Percona XtraDB Cluster HA 運作原理

要建置 Percona XtraDB Cluster 的環境，必須至少要有 3 nodes。

在 cluster 中，所有的 nodes 會自動同步資料庫中的資料，只要在 cluster 中的 node 沒有全部掛掉，就還是可以提供資料庫服務。

當 cluster 中有 node 掛掉時，隨著時間的變動，資料也會有所異動，當掛掉的 node 回復正常重新加入 cluster 後，有兩種方法可以進行資料的同步：

### 1、State Snapshot Transfer: (SST) 

SST 的功能是將 working node 的資料完整的複製並傳給新加入的 node，目前有三種傳送方式可以選擇：

1. **mysqldump**

2. **rsync**

3. **xtrabackup**

前兩個方法的缺點是會在整個資料傳送過程中進行 lock，這段時間中整個 cluster 是屬於 read-only 的狀態。

而 xtrabackup 則只會在傳送 .fm 檔案時需要作 lock，其他時間則不用，因此相對於前兩項是較好的選擇。


### 2、Incremental State Transfer (IST)

上面的 SST 是傳遞完整資料用，那 IST 則是傳遞部分資料用。
 
當 cluster node 某個 node 可能因為各種不同因素(e.g. 重開機、系統更新)而短暫的離開 cluster，而當此 node 重新加入 cluster 後，就可以透過 IST 很快的將變動的部分進行資料同步。

但 IST 可傳遞的資料並非沒大小限制，若資料量超過設定值，則還是會使用 SST 來進行資料同步。


-------------------------------------


示範環境
========

**OS**：Ubnutu 14.04 trusty64


**Cluster Nodes**：
- node01：192.168.122.101/24
- node02：192.168.122.102/24
- node03：192.168.122.103/24

**Percona XtraDB Cluster**：5.6

<font color='red'>**【註】**</font>每個 node 的記憶體至少需要 1.5 GB 以上，低於 1 GB 在 apt 安裝套件時會失敗，低於 1.5 GB 在啟動 mysql 服務加入 cluster 的時候會失敗。


-------------------------------------


前置作業
========

為了有 add-apt-repository 指令可用，必須安裝以下套件：

1. python-software-properties

2. software-properties-common

``` bash
$ sudo apt-get install python-software-properties software-properties-common
```


-------------------------------------


安裝 Percona XtraDB Cluster
===========================

#### 1. 新增套件庫

``` bash
$ sudo apt-key adv --recv-keys --keyserver keys.gnupg.net 1C4CBDCDCD2EFD2A
$ sudo add-apt-repository -y 'deb http://repo.percona.com/apt trusty main'
$ sudo add-apt-repository -y 'deb-src http://repo.percona.com/apt trusty main'
$ sudo apt-get update
```

#### 2. 設定 APT pin

新增檔案 **/etc/apt/preferences.d/00percona.pref**，內容如下：

```
Package: *
Pin: release o=Percona Development Team
Pin-Priority: 1001
```

#### 3. 安裝 percona xtradb cluster 套件

``` bash
$ sudo apt-get -y install percona-xtradb-cluster-full-56 python-mysqldb
```


-------------------------------------


設定 Percona XtraDB Cluster
===========================

#### 1. 設定 Galera WriteSet Replication 功能所使用的 DB User

以下請自行替換：

- User Name = <wsrep_sst_user>

- User Password = <wsrep_sst_password>

``` bash
mysql> CREATE USER '<wsrep_sst_user>'@'localhost' IDENTIFIED BY '<wsrep_sst_password>';
mysql> GRANT RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO ’sstuser’@’localhost’;
mysql> FLUSH PRIVILEGES;
```

#### 2. 加入 WriteSet Replication 相關設定

在每個 node 新增檔案 <font color='blue'>**/etc/mysql/conf.d/cluster.cnf**</font>，內容如下：

``` bash
[mysqld]
query_cache_size=0
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
query_cache_type=0
bind-address=0.0.0.0

# Galera Provider Configuration
wsrep_provider=/usr/lib/galera3/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="Openstack_DB_Cluster"	# 可自行定義 cluster name
wsrep_cluster_address="gcomm://192.168.122.101,192.168.122.102,192.168.122.103"		# 所有 node 的集合

# Galera Synchronization Congifuration
wsrep_sst_method=xtrabackup-v2
wsrep_sst_auth=<wsrep_sst_user>:<wsrep_sst_password>	# 輸入前面的 wsrep sst user & passsword

# Galera Node Configuration
wsrep_node_address="192.168.122.101"	# 改成每個 node ip address
wsrep_node_name="node01"	# 改成每個 node 的 hostname
```


-------------------------------------

啟動 Cluster
============

要能夠正確啟動 cluster 是有其順序性的，首先要在第一個 node 上執行以下指令：

``` bash
$ sudo service mysql restart-bootstrap
```

等待第一個 node 將 cluster 初始化完成之後，在其他的 nodes 執行以下指令：

``` bash
$ sudo service mysql restart
```

### 確認 cluster 狀態

最後確認一下 cluster 狀態，有看到以下的資訊表示 cluster 已經成功啟動囉!

``` bash
mysql> show status like 'wsrep%';
+------------------------------+----------------------------------------------------------------+
| Variable_name                | Value                                                          |
+------------------------------+----------------------------------------------------------------+
| ......................       | .....................................                          |
| wsrep_protocol_version       | 7                                                              |
| wsrep_local_state            | 4                                                              |
| wsrep_local_state_comment    | Synced                                                         |
| wsrep_incoming_addresses     | 192.168.122.101:3306,192.168.122.102:3306,192.168.122.103:3306 |
| wsrep_evs_state              | OPERATIONAL                                                    |
| wsrep_cluster_conf_id        | 2                                                              |
| wsrep_cluster_size           | 3                                                              |
| wsrep_cluster_state_uuid     | b8365002-04ec-11e5-b736-56bc0626b3ac                           |
| wsrep_cluster_status         | Primary                                                        |
| wsrep_connected              | ON                                                             |
| .................            | ......................................                         |
| wsrep_provider_name          | Galera                                                         |
| wsrep_provider_vendor        | Codership Oy <info@codership.com>                              |
| wsrep_ready                  | ON                                                             |
+------------------------------+----------------------------------------------------------------+
56 rows in set (0.00 sec)
```


-------------------------------------


參考資料
========

- [Percona XtraDB Cluster - A High Scalability Solution for MySQL Clustering and High Availability](https://www.percona.com/software/percona-xtradb-cluster)

- [Percona apt Repository — Percona XtraDB Cluster Documentation](https://www.percona.com/doc/percona-xtradb-cluster/5.6/installation/apt_repo.html)

- [Percona XtraDB Cluster 5.6 Documentation — Percona XtraDB Cluster Documentation](https://www.percona.com/doc/percona-xtradb-cluster/5.6/index.html)

- [How to set up Percona XtraDB Cluster on Ubuntu | Red Crackle](http://redcrackle.com/blog/how-set-percona-xtradb-cluster-ubuntu)

### WSREP(WriteSet Replication) 相關設定說明

- [Index of wsrep system variables — Percona XtraDB Cluster Documentation](https://www.percona.com/doc/percona-xtradb-cluster/5.6/wsrep-system-index.html)

- [Index of wsrep status variables — Percona XtraDB Cluster Documentation](https://www.percona.com/doc/percona-xtradb-cluster/5.6/wsrep-status-index.html)