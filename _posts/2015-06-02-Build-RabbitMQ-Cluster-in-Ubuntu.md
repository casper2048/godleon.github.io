---
layout: post
title:  "[Cluster] 在 Ubuntu 上安裝 RabbitMQ Cluster"
description: "在這邊文章中，介紹如何在 Ubuntu 14.04(trusty64) 上安裝 RabbitMQ Cluster 並設定 Queue HA"
date: 2015-06-02 09:20:00
published: true
comments: true
categories: [amqp]
tags: [Cluster, AMQP, RabbitMQ, Linux, Ubuntu]
---


前言
====

暨前一篇文章完成 DB cluster 之後，在 OpenStack 中還有一個很重要的服務需要保護，就是 Message Queue 的服務啦!

透過 Message Queue 的服務，就可以讓其他耗時的服務更方便的進行狀態的控制 & 管理。

一般安裝 OpenStack，在 Message Queue 服務的部分大多會安裝 [RabbitMQ](https://www.rabbitmq.com/)，因此以下的安裝示範就是以 RabbitMQ Cluster 為主。

此外，關於 cluster 的部分有幾點需要知道的：

1. RabbitMQ Cluster 無法在 WAN 的環境中正常運作，因此若是跨 WAN 的資料中心，就必須使用 RabbitMQ [shovel](https://www.rabbitmq.com/shovel.html) or [federation](https://www.rabbitmq.com/federation.html) 這兩個 plugin 來協助了。

2. 加入 cluster 的 node type 可以是 **disk node** or a **RAM node**，一般都是以 disk node 為主，只有在特殊狀況下要求效能時，就可以考慮 RAM node。

---------------------------------------------


安裝環境說明
============

**OS**：Ubuntu 14.04 Trusy64

**Nodes**：

- master：192.168.122.101

- slave01：192.168.122.102

- slave02：192.168.122.102


---------------------------------------------

前置工作準備
============

在每一個 node 上執行以下工作：

### 1. 安裝管理 APT repository 用套件

``` bash
$ sudo apt-get update
$ sudo apt-get -y install python-software-properties software-properties-common
```

### 2. 加入 APT repository

``` bash
$ wget https://www.rabbitmq.com/rabbitmq-signing-key-public.asc
$ sudo apt-key add rabbitmq-signing-key-public.asc
$ sudo add-apt-repository -y 'deb http://www.rabbitmq.com/debian/ testing main'
$ sudo apt-get update
```

### 3. 設定 DNS

修改 /etc/hosts 檔案，加入以下內容：

```
192.168.122.101 master.example.com master
192.168.122.102 slave01.example.com slave01
192.168.122.103 slave02.example.com slave02

```

並確認相互之間可以透過 domain name 互通。(例如：$ ping master)


---------------------------------------------


安裝 RabbitMQ
=============

### 1. 安裝 RabbitMQ server 套件

``` bash
$ sudo apt-get -y install rabbitmq-server
```

### 2. 啟用 RabbitMQ Management Web Console

``` bash
$ sudo rabbitmq-plugins enable rabbitmq_management
```

### 3. 開啟遠端使用者可登入 web console

``` bash
$ sudo sh -c "echo '[{rabbit, [{loopback_users, []}]}].' > /etc/rabbitmq/rabbitmq.config"
$ sudo service rabbitmq-server restart
```

接著就可以透過 http://<your_rabbitmq_server_ip>:15672 來登入 RabbitMQ management web console 了。(預設帳號密碼為 <font color='blue'>**guest**</font> / <font color='blue'>**guest**</font>)


---------------------------------------------


同步 ERLang Cookie
==================

ERLang Cookie 的用途是讓 nodes 之間判斷能不能相互溝通之用，一般來說大家都是不一樣的，因此每個 node 原本都是獨立運作的，而為了讓 nodes 可以形成一個 cluster，首先必須將 ERLang Cookie 進行同步。

在同步 ERLang Cookie 之前，必須先將 <font color='red'>**slave nodes**</font> 的 RabiitMQ 服務停止：

``` bash
$ sudo service rabbitmq-server stop
```

接著同步 ERLang Cookie：(**/var/lib/rabbitmq/.erlang.cookie**)

### 1、備份 slave nodes 上的 ERLang Cookie

``` bash
$ sudo cp /var/lib/rabbitmq/.erlang.cookie /var/lib/rabbitmq/.erlang.cookie.bak
```

### 2、將 master node 上的 ERLang Cookie 複製到 slave nodes 上

``` bash
$ scp /var/lib/rabbitmq/.erlang.cookie <ssh_user>@slave01:~/
$ scp /var/lib/rabbitmq/.erlang.cookie <ssh_user>@slave02:~/
```

### 3、取代原本的 ERLang Cookie

``` bash
$ sudo mv ~/.erlang.cookie /var/lig/rabbitmq/.erlang.cookie
```

### 4、設定檔案權限

``` bash
$ sudo chown rabbitmq:rabbitmq /var/lig/rabbitmq/.erlang.cookie
$ sudo chmod 400 /var/lig/rabbitmq/.erlang.cookie

```

### 5、重新啟動 slave nodes 上的 rabbitmq-server 服務

``` bash
$ sudo service rabbitmq-server start
```


---------------------------------------------


設定 RabbitMQ Cluster
=====================

### 1、重設 RabbitMQ service @ master

``` bash
$ sudo rabbitmqctl stop_app
$ sudo rabbitmqctl start_app
```

### 2、將 slave nodes 加入 cluster 中

``` bash
$ sudo rabbitmqctl stop_app
$ sudo rabbitmqctl join_cluster --ram rabbit@master
$ sudo rabbitmqctl start_app
```

### 3、確認 cluster 狀態

``` bash
$ sudo rabbitmqctl cluster_status
Cluster status of node rabbit@master ...
[{nodes,[{disc,[rabbit@master]},{ram,[rabbit@slave02,rabbit@slave01]}]},
 {running_nodes,[rabbit@slave02,rabbit@slave01,rabbit@master]},
 {cluster_name,<<"rabbit@master.example.com">>},
 {partitions,[]}]
```

看到以上訊息，表示 RabiitMQ Cluster 已經設定完成囉!

### 4、設定 queue HA policy

雖然 RabbitMQ cluster 已經設定完成，有看到多個 message broker，但事實上 queue 只存在於 master 中，因此為了達到 queue HA 的目的，就必須要透過 policy 將 queue 進行 mirrow，讓 cluster 中的每一個 node 都有完整的 queue 資訊。

我們可以在任一個 node 上輸入以下命令：

``` bash
$ sudo rabbitmqctl set_policy ha-all '^(?!amq\.).*' '{"ha-mode": "all"}'
```

可以檢查目前的 policy 狀況：

``` bash
$ sudo rabbitmqctl list_policies
Listing policies ...
/       ha-all  all     ^(?!amq\\.).*   {"ha-mode":"all"}       0
```

若看到上面的訊息，就表示 queue HA policy 就設定完成了!


---------------------------------------------


參考資料
========

### 基本概念

- [在 Ubuntu Linux 中安裝與使用 RabbitMQ 訊息佇列 - G. T. Wang](http://blogger.gtwang.org/2014/01/ubuntu-linux-install-rabbitmq.html)

- [以 RabbitMQ 實作工作佇列（Work Queues）（Python 版本） - G. T. Wang](http://blogger.gtwang.org/2014/04/rabbitmq-work-queues.html)

- [以 RabbitMQ 實作 Publish/Subscribe 模型（Python 版本） - G. T. Wang](http://blogger.gtwang.org/2014/04/rabbitmq-publish-subscribe-pattern-python.html)

- [同樣都是Message Queue 到底AMQP跟JMS 有什麼差別？ - 阿貝好威的實驗室](http://lab.howie.tw/2012/07/whats-different-between-amqp-and-jms.html)

- [Kuma拔拔的點滴: AMQP協議 ](http://kuma-uni.blogspot.tw/2012/04/amqp.html)

### Cluster 相關

- [RabbitMQ - Distributed RabbitMQ brokers  (Federation/Shovel  V.S.  Cluster )](https://www.rabbitmq.com/distributed.html)

- [RabbitMQ - Clustering Guide](https://www.rabbitmq.com/clustering.html)

- [RabbitMQ - Highly Available Queues](https://www.rabbitmq.com/ha.html)

### 安裝

- [如何安裝RabbitMQ Cluster - 阿貝好威的實驗室](http://lab.howie.tw/2012/07/install-rabbitmq-with-cluster.html)

- [Setup Rabbitmq Clusters on Ubuntu | Binary Pointer](http://blog.hemantthorat.com/setup-rabbitmq-clusters-on-ubuntu/#.VWugDUZNjIV)

- [How To Cluster Rabbit-MQ | Jason Graves](http://collaboradev.com/2010/12/14/how-to-cluster-rabbit-mq/)

### 設定

- [RabbitMQ - Configuration](https://www.rabbitmq.com/configure.html)

### 障礙排除

#### 1、無法使用 guest/guest 登入 RabbitMQ Management Web Console

- [RabbitMQ - Access Control](http://www.rabbitmq.com/access-control.html) 

- [RabbitMQ 3.3.1 can not login with guest/guest - Stack Overflow](http://stackoverflow.com/questions/23669780/rabbitmq-3-3-1-can-not-login-with-guest-guest)
