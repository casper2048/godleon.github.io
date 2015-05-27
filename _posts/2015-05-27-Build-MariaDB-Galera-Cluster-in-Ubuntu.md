---
layout: post
title:  "[Cluster] 在 Ubuntu 上安裝 MariaDB Galera Cluster"
description: "在這邊文章中，介紹如何在 Ubuntu 14.04(trusty64) 上安裝 MariaDB Galera Cluster"
date: 2015-05-27 11:30:00
published: true
comments: true
categories: [cluster]
tags: [Cluster, Database, Linux, Ubuntu]
---

1、前言
=======

因為工作上的需要，必須要建置一套 MySQL cluster，讓資料庫服務可以達成 HA(High Availability)。

找了一下目前比較通用的作法，目前有以下兩種可以選擇：

1. [MariaDB Galera Cluster](What is MariaDB Galera Cluster? - MariaDB Knowledge Base)

2. [Percona XtraDB Cluster](https://www.percona.com/software/percona-xtradb-cluster)

以下為 MariaDB Galera Cluster 的實作介紹

---------------------------------------------------------

2、測試環境說明
===============

OS：Ubuntu 14.04 trusty64

Cluster nodes：
- node01: 192.168.122.101 (first node)
- node02: 192.168.122.102
- node03: 192.168.122.103

MariaDB version：10.0

<font color='red'>**【註】**</font>MariaDB 版本建議使用 10.0，之前用 ansible 自動化安裝 5.5 版本很多次，三不五時會遇到 package dependency 的問題。(明明用 apt 安裝不應該會有這問題，所以這是很弔詭的事情。)

---------------------------------------------------------


3、安裝 & 設定 MariaDB Galera Cluster
==========================================

### 安裝 cluster 相關套件

以下動作要在 cluster 中的每一個 node 上面執行：

#### (1) 安裝基本套件

``` bash
$ sudo apt-get update
$ sudo apt-get -y install software-properties-common python-software-properties
```

#### (2) 更新 APT repository for MariaDB(5.5) Galera Cluster

``` bash
$ sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xcbcb082a1bb943db
$ sudo add-apt-repository 'deb http://nyc2.mirrors.digitalocean.com/mariadb/repo/10.0/ubuntu trusty main'
$ sudo apt-get update
```

#### (3) 安裝 MariaDB Galera Cluster 相關套件

``` bash
$ sudo apt-get -y install python-mysqldb galera mariadb-galera-server
```

#### (4) 建立同步資料庫用的使用者帳號

登入每一台 MariaDB 中，建立使用者並設定權限：

``` bash
MariaDB [(none)]> GRANT ALL PRIVILEGES ON *.* TO '<user_name>'@'%' IDENTIFIED BY '<user_password>';
```


### 設定 cluster

以下動作要在 cluster 中的每一個 node 上面執行：

#### (1) 修改 MariaDB 設定

善用 **!includedir /etc/mysql/conf.d/** 的功能，我們可以不用直接修改 **/etc/mysql/my.cnf**，只要把設定放在 <font color='red'>**/etc/mysql/conf.d**</font> 目錄中，原有的設定會 overwrite，沒有的設定則會 append。

因此我們新增設定檔 **/etc/mysql/conf.d/cluster.cnf**，並設定內容如下：

```
[mysqld]
# 開放外部存取資料庫服務(預設僅 localhost 可存取)
bind-address = 0.0.0.0

# 因為 Galera 使用 RBR format 建立 writesets，因此 binlog_format 必須設定為 row-level
binlog_format = ROW

# Galera cluster 僅能使用 InnoDB
default_storage_engine = innodb

# auto-increment 必須設定為 interleaved lock mode(2)，設定其他 mode(0 or 1) 可能會在 insert 資料的時候造成 deadlock
innodb_autoinc_lock_mode = 2

# 為提升 performance，將 log buffer flush 的時間改為每秒，而非每次 commit 時
innodb_flush_logs_at_trx_commit=0

# 預設為 128MB，保留 5% 給 Galera cluster 使用(若設定太大，記憶體不足會導致 mysql 服務無法啟動)
innodb_buffer_pool_size = 122M

# 關閉 query cache 功能
query_cache_type = 0

# ========= 以下為 Galera WriteSet Replication(wsrep) API 設定 =================

# Galera Provider 設定
wsrep_provider = /usr/lib/galera/libgalera_smm.so

# Galera Cluster 設定
wsrep_cluster_name="OpenStack_DB_Cluster"
wsrep_cluster_address="gcomm://192.168.122.101,192.168.122.102,192.168.122.103"

# Galera 同步設定(使用者帳號必須設定好權限)
wsrep_sst_method=rsync      # 若改用 Percona xtrabackup，則要將值改為 'xtrabackup' (速度較快)
wsrep_sst_auth=<user_name>:<user_password>

# Galera Node 設定(每個 node 不一樣)
wsrep_node_address="192.168.122.101"    # 置換成每個 node 各自的 ip
wsrep_node_name="node01"                # 置換成每個 node 的 hostname
```

#### (2) 複製 /etc/mysql/debian.cnf 

為了確保 **debian-sys-maint** 的密碼是相同的，我們必須將第一個 node 的 <font color='red'>**/etc/mysql/debian.cnf**</font> 複製到其他 node。

``` bash
$ scp /etc/mysql/debian.cnf root@<IP_of_other_node>:/etc/mysql
```


### 啟動 cluster

啟動 cluster 是有其順序性的! 請按照以下步驟：

#### (1) 第一個 node 啟動 cluster 服務

第一個 node 是哪一個? 就是在 **cluster.cnf** 中 <font color='blue'>**wsrep_cluster_address**</font> 參數的第一個 ip 的 node。

執行以下命令啟動 cluster 服務：(多加上 <font color='blue'>**--wsrep-new-cluster**</font>)

``` bash
$ sudo service mysql stop
$ sudo service mysql start --wsrep-new-cluster
```

#### (2) 其餘 node 啟動 DB 服務

``` bash
$ sudo service mysql restart
```


### 確認 cluster 服務已經啟動

隨便選一台機器，登入 MariaDB(以 root 身分)，並下達以下命令：

``` bash
MariaDB [(none)]> show status like 'wsrep%';
+------------------------------+----------------------------------------------------------------+
| Variable_name                | Value                                                          |
+------------------------------+----------------------------------------------------------------+
| wsrep_local_state_comment    | Synced                                                         |
| wsrep_incoming_addresses     | 192.168.122.101:3306,192.168.122.102:3306,192.168.122.103:3306 |
| wsrep_cluster_size           | 3                                                              |
| wsrep_cluster_status         | Primary                                                        |
| wsrep_connected              | ON                                                             |
| wsrep_ready                  | ON                                                             |
| ...........                  | ..............                                                 |
+------------------------------+----------------------------------------------------------------+
57 rows in set (0.00 sec)
```

從上面可以看到目前 cluster 已經可以正常運作了!

之後在任何 node 所有的操作(包含對 database / table / data 的新增修改刪除)，都會同步的 cluster 中的所有 node 上。


---------------------------------------------------------

4、效能測試
===========

以下效能測試看看就好.........

首先要找台電腦安裝 **sysbench**：

``` bash
$ sudo apt-get -y install sysbench
```

接著在任一台機器上建立資料庫 sbtest：

``` bash
MariaDB [(none)]> create database sbtest;
```

回到 client 端，執行以下指令安裝 tables & data：

``` bash
$ sysbench --debug=off --test=oltp --db-driver=mysql --mysql-table-engine=innodb --oltp-table-size=1000000 --mysql-user=<user_name> --mysql-password=<user_password> --mysql-host=192.168.122.101 prepare
```

進行測試：

```
$ sysbench --debug=off --num-threads=16 --test=oltp --db-driver=mysql --mysql-table-engine=innodb --oltp-table-size=1000000 --mysql-user=<user_name> --mysql-password=<user_password> --mysql-host=192.168.122.101 run

sysbench 0.4.12:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 16

Doing OLTP test.
Running mixed OLTP test
Using Special distribution (12 iterations,  1 pct of values are returned in 75 pct cases)
Using "BEGIN" for starting transactions
Using auto_inc on the id column
Maximum number of requests for OLTP test is limited to 10000
Threads started!
Done.

OLTP test statistics:
    queries performed:
        read:                            140000
        write:                           50000
        other:                           20000
        total:                           210000
    transactions:                        10000  (148.14 per sec.)
    deadlocks:                           0      (0.00 per sec.)
    read/write requests:                 190000 (2814.62 per sec.)
    other operations:                    20000  (296.28 per sec.)

Test execution summary:
    total time:                          67.5046s
    total number of events:              10000
    total time taken by event execution: 1079.4991
    per-request statistics:
         min:                                 37.68ms
         avg:                                107.95ms
         max:                                709.25ms
         approx.  95 percentile:             179.54ms

Threads fairness:
    events (avg/stddev):           625.0000/3.87
    execution time (avg/stddev):   67.4687/0.02
```


---------------------------------------------------------

5、結論
====

其實從測試中可以發現，要連結資料庫還是要指定某一台才行，雖然資料修改後 db cluster 會自動同步，但若是連線的 db 掛點了，還是會造成服務停擺。

有鑑於此，之後會希望可以有個 virtual ip 代表整個 db cluster，只要不是全部 db 都掛掉的狀況，透過 virtual ip 都還是可以連上。

要達到以上目的，有兩種方法：

1. 使用 corosync + pacemaker，建立 virtual ip

2. 使用 HAProxy，提供 Load Balance 服務

關於第一個方法，在[之前的文章](https://godleon.github.io/blog/2015/05/20/Build-2-nodes-Linux-HA-Cluster-By-Corosync-Pacemaker/)有介紹過；之後若是有機會，將會以第二種方法為主。


---------------------------------------------------------

6、參考資料
========

- [Galera Cluster Documentation — Galera Cluster Documentation](http://galeracluster.com/documentation-webpages/index.html)

- [MySQL wsrep Options — Galera Cluster Documentation](http://galeracluster.com/documentation-webpages/mysqlwsrepoptions.html)

- [How To Configure a Galera Cluster with MariaDB on Ubuntu 12.04 Servers | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-configure-a-galera-cluster-with-mariadb-on-ubuntu-12-04-servers)

- [How to deploy a MariaDB Galera Cluster on Ubuntu](http://davidburgosonline.com/ddbb/2014/deploy-mariadb-galera-cluster-ubuntu/)

- [How to setup load balancer HAProxy with MariaDb Cluster](http://davidburgosonline.com/ddbb/2014/setup-load-balancer-haproxy-front-mariadb-cluster/)

- [apt - How do I resolve unmet dependencies after adding a PPA? - Ask Ubuntu](http://askubuntu.com/questions/140246/how-do-i-resolve-unmet-dependencies-after-adding-a-ppa)

- [实例解读MySQL debian-sys-maint的安全设置 - 51CTO.COM](http://netsecurity.51cto.com/art/201107/272722.htm)

- [Active/Passive failover cluster with Pacemaker on a MySQL-Galera cluster with HAProxy (LSB agent) - Sébastien Han](http://www.sebastien-han.fr/blog/2012/04/15/active-passive-failover-cluster-on-a-mysql-galera-cluster-with-haproxy-lsb-agent/)

- [[SOLVED] MySQL Crash with Fatal error: cannot allocate memory for the buffer pool | Web Traffic Exchange](http://www.webtrafficexchange.com/solved-mysql-crash-fatal-error-cannot-allocate-memory-buffer-pool)

- [MySQL benchmark tool - sysbench @ 不大會寫程式 :: 隨意窩 Xuite日誌](http://blog.xuite.net/misgarlic/weblogic/56170203-MySQL+benchmark+tool+-+sysbench)