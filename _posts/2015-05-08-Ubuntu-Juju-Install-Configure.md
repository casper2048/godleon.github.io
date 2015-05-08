---
layout: post
title:  "[Ubuntu] Juju 的安裝 & 設定"
description: "在這篇文章中，介紹 Ubuntu Juju 的特色與強大的功能所在，並介紹安裝 & 設定方式，最後是簡單的使用說明"
date:   2015-05-08 21:35:00
published: true
comments: true
categories: [juju]
tags: [Juju, Ubuntu, Linux]
---


Juju 是甚麼?
============

首先先看一下 Juju 官網對 Juju 的定義：

- Juju is the best solution to orchestrate your services in the cloud. You can use it through a GUI or from the command line, scale your services and easily move your environment between clouds.

再來是官網對 Juju feature 的說明：

- Juju’s powerful GUI and command-line interface make it easy to define, configure, deploy, manage, monitor and scale out your services to any public or private cloud.



Juju 與目前很紅的 Puppet / Chef / Ansible 一樣，用途是 Auto Deployment，目的是為了快速的進行 Service Orchestration 用。

但 Juju 拉高了一個層級，透過 charm 的概念，把每個 service(例如：MySQL、Apache、nginx、MongoDB、redis ... 等等) 變成了一個個的 [charm][1]，再透過設定 [charm][1] 的內容並定義不同的 charm 之前的關聯，讓使用者可以很快速的建置 & 串起所需要的服務。

此外，Juju 還支援了許多公有雲的平台，在不同的平台上，管理者可以透過相同的 [charm][1] 設定(集合起來稱為 [Bundle][2])，

因此，假設你是開發 PostgreSQL 的開發商，就可以開發自己的 charm，並分享到 charm store 上，其他人就可以很方便的透過 Juju 並搭配你開發的 charm 來將 PostgreSQL 與其他的服務進行整合。

如果以上還不了解 Juju 是甚麼，可以參考以下影片：


<iframe width="560" height="315" src="https://www.youtube.com/embed/1yoIdgdqzLk" frameborder="0" allowfullscreen></iframe>

並搭配官方網站的「[Top 12 questions about Juju](https://insights.ubuntu.com/2013/08/29/top-12-questions-about-juju/)」

我想應該對 Juju 就會比較有完整的了解。

-------------------------------------

Juju 的運作方式
===============

Juju 要進行運作必須要達成三個要素：

### 1、Juju client

client 端在不同的平台(Ubuntu / OSX / Windows)都有，以 Ubuntu 為例，client 套件的名稱為 **juju-core**。

### 2、可以 on-demand 提供 Ubuntu Image 並自動生成 Ubuntu Server 的環境

一般公有雲都可以提供這樣的環境，例如：Amazon EC2 / Microsoft Azure；OpenStack 同樣也可以提供這樣的功能。

也可搭配 Ubuntu MAAS 來完成一整套的 solution，從 Bare Metal 到 Service 的完整自動化部屬 

### 3、用來控制 Ubuntu Server 的 SSH key pair

這一對 ssh key pair 是 Juju 用來遠端登入到 Ubuntu server 用，登入之後才可以進行 Service Orchestration 的作業。


-------------------------------------


安裝 & 設定 Juju
================

### 新增 APT repository

``` bash
$ sudo add-apt-repository -y ppa:juju/stable
$ sudo apt-get update && sudo apt-get -y install juju-core
```

### 設定 cloud provider

這個步驟分為兩個階段：

#### 1、產生設定檔 <font color='blue'>**~/.juju/environments.yaml**</font>

``` bash
$ juju generate-config
```

產生預設設定檔如下：

``` yaml
default: amazon
environments:
    openstack:
        type: openstack
    hpcloud:
        type: openstack
    manual:
        .... <manual-related configuration>
    local:
        type: local
    joyent:
      type: joyent
    gce:
      type: gce
      auth-file:
      project-id:
    amazon:
        type: ec2
    azure:
        .... <azure-related configuration>
```

#### 2、修改設定檔至正確的 cloud provider


以下以 AWS EC2 為例，修改檔案(<font color='blue'>**~/.juju/environments.yaml**</font>)以下內容：

``` yaml
default: amazon
    amazon:
        type: ec2
		
    my_amazon:
        type: ec2
        region: us-east-1
        access-key: <your_access_key>
        secret-key: <your_secret_key>
```

以上看的出來，主要是設定 **access-key** & **secret-key** 兩個部分。


### 設定 bootstrap 環境 (建置 juju state-server)

安裝 & 設定完 juju 之後，接著要產生一個 bootstrap instance(juju state-server)，指令如下：

``` bash
$ juju bootstrap -e my_amazon
```

到底 bootstrap instance 是要幹嘛用? 

簡單來說，juju client / juju bootstrap node / cloud provider 三者的關係，說明如下：

1. juju client 透過 bootstrap 指令，在 cloud provider 上產生一台用來部署 juju charms 的 bootstrap instance。

2. 接著 juju client 就可以透過 `juju deploy` 的指令，呼叫 juju bootstrap instance 安裝指定的 charms 到 cloud provider 的 VM 中。(若 cloud provider 是 MAAS，則是安裝到實體機上)

3. 最後在 cloud provider 上，除了提供 bootstrap 功能的 instance 外，還有安裝了指定 charms 的 instance。 

所以整個關係如以下這樣：

> **juju client** <---> **juju bootstrap instance** <---> **cloud provider**


-------------------------------------


使用 Juju 進行 Service Orchestration
====================================

當所有設定都完成後，就可以來試試看 Juju 是不是如官網上說的這麼神奇，可以透過幾行簡單的指令來完成 service orchestration。

在官網上有一段 Live Demo 影片可以看：

<iframe width="420" height="315" src="https://www.youtube.com/embed/0AT6qKyam9I" frameborder="0" allowfullscreen></iframe>

整個流程的步驟如下：

1. 使用 `juju bootstrap` 指令在 cloud provider 上產生第一個 instance，這個 instance 會用來進行後續的 service orchestration 用。

2. 使用 `juju deploy` 指令呼叫 bootstrap instance，在 cloud provider 上啟動新的 machine 並安裝指定的 juju charms。

3. 使用 `juju add-relation` 指令將多個 charms 進行關聯，例如範例中的 wordpress & mysql 的關聯。(wordpress 使用 mysql 作為資料庫工具)

4. 若是有特定服務是要對外公開的，則可以使用 `juju expose` 指令指定公開的服務。

5. 整個佈署過程需要一點時間，可以使用 `juju status` 指令來查詢目前在 cloud provider 上 machine & charms 佈署的狀態。

6. 最後若要移除安裝好的 charms & machine，可以透過 `juju destroy-environment` 指令來移除。(此指令會連 bootstrap instance 一起移除)


-------------------------------------


Juju Cheatsheet
===============

最後放個官網提供的 [juju cheatsheet](https://github.com/juju/cheatsheet)

這樣可以方便的查詢 juju 相關的指令如何使用。


-------------------------------------


參考資料
========

- [Welcome to the Juju charm browser | Juju][1]

- [Introduction | Documentation | Juju](https://jujucharms.com/docs/stable/getting-started)


[1]: https://jujucharms.com/	"Ubuntu Juju Charm"
[2]: https://jujucharms.com/docs/devel/charms-bundles	"Ubuntu Juju Bundle"