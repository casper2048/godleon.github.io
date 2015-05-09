---
layout: post
title:  "[Ubuntu] Service Orchestration by Juju"
description: "在這篇文章中，介紹如何使用 Juju 快速的進行 service orchestration"
date:   2015-05-09 20:55:00
published: true
comments: true
categories: [juju]
tags: [Juju, Ubuntu, Linux]
---


Deploying Services
==================

這個部分要來說明如何使用 Juju 來進行 service deployment 的工作：

### 從 Charm Store 進行佈署

透過 charm store 是最標準的做法，以下是 deply 的範例：

``` bash
# 透過 charm store 佈署 mysql
$ juju deploy mysql


# 指定版本的佈署方式，格式為 <repository>:<series>/<service>
$ juju deploy cs:precise/mysql


# 不想每次都指定 series，則可以設定 juju charms 的預設版本為 14.04(trusty)
$ juju set-env "default-series=trusty"
```

### 從 Local Charm Repository 進行佈署

``` bash
# 使用 local charm repository 安裝 service
# repository 的路徑為 /usr/share/charms
# repository name: local
# series: trusty
# service: vsftp
$ juju deploy --repository=/usr/share/charms/ local:trusty/vsftpd


# 預先指定環境變數 JUJU_REPOSITORY
export JUJU_REPOSITORY=/usr/share/charms/
juju deploy local:trusty/vsftpd
```

### 使用設定檔進行佈署

Juju 的設定檔是以 [YAML](http://zh.wikipedia.org/wiki/YAML) 格式撰寫，可以在設定檔中預先將 charm 的相關設定完成，再透過指定設定檔的方式，完成複雜 service 的佈署。

若是要佈署的 charm 有非常多的參數要進行調整時，將參數全部放到指令內，會讓指令變得一大串，但若是可以透過設定檔預先定義好，就只要在指令中指定好設定檔即可。

假設這裡要佈署一個 MediaWiki 的服務，於是做了以下的設定檔(**myconfig.yaml**)：

``` yaml
mediawiki:
  name: Juju Wiki
  skin: monobook
  admins: admin:admin
```

就可以用以下指令進行 service 的佈署：

``` bash
$ juju deploy --config myconfig.yaml mediawiki
```


--------------------------------------------


使用 constraints
================

在 Juju 指令中加入 <font color='red'>**--constraints**</font>，可以在進行 service deployment 時指定特定的設備規格(CPU、RAM ... etc)、平台(i386、amd64 ... etc)

以下幾個指令可以加入 --constraints 參數：

- juju deploy

- juju bootstrap

- juju add-machine

以下指令則無法加入 --constraints 參數：

- juju add-unit


``` bash
$ juju deploy --constraints "cpu-cores=8 mem=32G" mysql

$ juju bootstrap --constraints "cpu-power=0 cpu-cores=0 mem=512M"

juju bootstrap --constraints arch=i386

$ juju set-constraints --service mysql mem=8G cpu-cores=4
```

透過以下指令可以取得 Juju 環境中目前的 constraints 內容：

``` bash
$ juju get-constraints

# 也可以指令 charm 名稱來取得綁定在上面的 constraints 內容
$ juju get-constraints mysql
```

### MAAS 環境下的 constraints

若 cloud provider 為 MAAS，則可以搭配 <font color='red'>**tags**</font> 使用：

``` bash
$ juju deploy mysql --constraints tags=foo,bar
```

--------------------------------------------


Unit 的使用
===========

每個由 Juju 管理的 node(不論是實體 or container) 都被稱為 Unit。

由於在進行 service 佈署時，除非是特定要在某台機器上執行的服務，一般是不會指定位置進行佈署。

那......假設 mysql 佈署到 10 台機器上，我要連到第三台去進行一些管理工作時，要怎麼辦呢? 可以透過以下方式：

### juju ssh

juju ssh 指令可以讓使用者透過指定 unit 的方式，以 ssh 的方式登入到<font color=''red>**某一台**</font>遠端機器並執行 shell 命令。

``` bash
# 透過 ssh 連線到 mysql unit 3
$ juju ssh mysql/3

# 直接查詢 mysql unit 3 上機器的 Linux kernel version
$ juju ssh mysql/3 uname -a

# 查詢佈署 rabbitmq-server 服務的
$ juju ssh rabbitmq-server/0 ifconfig

# 執行遠端機器的 shell script
$ juju ssh rabbitmq-server/0 sh /tmp/echo_ip.sh
```

### juju scp

有 ssh 的功能，自然就會有 scp 可以用，透過 scp 我們可以將檔案放到 service 存在的 machine or container，也可以將檔案 copy 回來，使用範例如下：

``` bash
# 將檔案 echo_ip.sh 放到 rabbitmq-server 服務存在的機器上的 /tmp 目錄中
$ juju scp echo_ip.sh rabbitmq-server/0:/tmp

# 將安裝 mongodb 服務的遠端機器的 log 複製到本機儲存
$ juju scp -r mongodb/0:/var/log/mongodb/ remote-logs/

# 甚至可以拉遠端兩台機器的檔案回來.....
$ juju scp -v mysql/0:/path/file1 mysql/1:/path/file2 backup/
```

### juju run

juju run 同樣也是以 ssh 的方式，登入到遠端機器去執行 shell 命令，但與 juju ssh 有甚麼不同呢?

- **juju run**：可以同時操控多台機器，可以透過指定 machine / service / unit 的方式指定機器，也可以使用 <font color='blue'>**--all**</font> 參數一次對所有的機器 & container 進行操控。

- **juju ssh**：可以透過指定 machine / service 的方式指定機器，但一次只能操控一台。

以下是 juju run 的幾個範例：

``` bash
# 查詢所有機器 & container 的 kernel version
$ juju run "uname -a" --all

# 也可以用指定 machine or service constainer 的方式
$ juju run "uptime" --machine=2
$ juju run "uptime" --service=mysql
```

### 更多資訊

上面介紹的 juju ssh/scp/run 等指令，可以用以下指令查詢詳細使用方式：

``` bash
$ juju help ssh

$ juju help scp/run

$ juju help run
```


--------------------------------------------


Juju 的移除功能
===============

Juju 的移除功能可強大了.....除了可以把 service 移除，machine 也可以移除，就連整個 environment 都可以移除。

用法如下：

``` bash
# 移除 service
$ juju remove-service <service-name>

# 移除 unit
$ juju remove-unit mediawiki/1

# 同時移除多個 unit
$ juju remove-unit mediawiki/1 mediawiki/2 mediawiki/3 mysql/2 ...

# 移除 machine (如果有 service 佈署在上面時無法移除)
$ juju remove-machine <number>

# 移除 environment
$ juju destroy-environment <environment>
```


--------------------------------------------


參考資料
========

- [Welcome to the Juju charm browser | Juju](https://jujucharms.com/)

- [Introduction | Documentation | Juju](https://jujucharms.com/docs/stable/getting-started)
