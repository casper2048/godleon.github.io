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

- **juju deploy**

- **juju bootstrap**

- **juju add-machine**

以下指令則無法加入 --constraints 參數：

- **juju add-unit**


``` bash
$ juju deploy --constraints "cpu-cores=8 mem=32G" mysql

$ juju bootstrap --constraints "cpu-power=0 cpu-cores=0 mem=512M"

$ juju bootstrap --constraints arch=i386

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

若要了解如何在 MAAS 環境建立 tag 相關設定，可以參考 [Making use of Tags — MAAS dev documentation](https://maas.ubuntu.com/docs/tags.html)

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



管理 Charm 之間的關聯性
=======================

### 簡單關聯

每個 charm 佈署之後都是一個服務，而服務鮮少有獨立運作的；因此如何建立服務之間的關聯性就很重要了。

假設我們透過 juju 佈署了 **[mysql](https://jujucharms.com/mysql/trusty/25)** & **[wordpress](https://jujucharms.com/wordpress/trusty/2)** 兩個 charm，要建立兩個 charm 的關聯，可透過以下指令：

``` bash
$ juju add-relation mysql wordpress
```

### 關聯是如何建立的?

從上述的範例可看出，建立兩個 charm 的關聯只要一行指令即可完成，但到底兩個 charm 是如何知道「**要怎麼關聯**」呢?

答案就是 charm 中已經包含了如何與其他服務相關聯的程式了。

以上面的例子來說，當佈署完 wordpress 之後，wordpress 會知道自己還需要一個後端資料庫用來儲存資料用；而 mysql 被佈署之後，它自己也會知道就是要扮演一個資料庫的角色。

於是當兩者關聯被建立起來時，wordpress 會通知 mysql 執行建立所需要的使用者權限、Database、Table .... 等工作，而當關聯建立完成後，wordpress 就可以存取 mysql 做為後端資料庫之用了!


### 複雜關聯

但有時候不是每個服務都是這麼簡單就可以關聯起來，例如 **[mediawiki](https://jujucharms.com/mediawiki/trusty/3)** & **[mysql](https://jujucharms.com/mysql/trusty/25)** 的關聯：

``` bash
$ juju deploy mediawiki --to lxc:1
Added charm "cs:trusty/mediawiki-3" to the environment.

maasqct@maas:~$ juju deploy mysql --to lxc:2
Added charm "cs:trusty/mysql-25" to the environment.

# 發生錯誤
$ juju add-relation mediawiki mysql
ERROR ambiguous relation: "mediawiki mysql" could refer to "mediawiki:db mysql:db"; "mediawiki:slave mysql:db"
```

從上面可以看出，其實 mediawiki 是有多個 <font color='blue'>**hook identifier**</font>(也稱為 <font color='red'>**Role**</font>)，而每個 cahrm 有那些 Role 可以使用，可以到 [charm store](https://jujucharms.com/store) 查詢，資訊可以從下圖中的位置找到：

![Charm's Role Information](https://lh3.googleusercontent.com/-Y6llD17rWy8/VVMJ1E_30iI/AAAAAAAAK1g/MeVwM3GpfJ0/w775-h567-no/charm_desc.png)

因此要建立 mediawiki 與 mysql 關係，要將指令改成如下：

``` bash
$ juju add-relation mediawiki:db mysql
```

--------------------------------------------


Scaling Services
================

在雲端環境佈署服務，最大的好處就是可以有強大的水平擴展(scale out)的能力。

而在 Juju 中，使用者可以很簡單的對服務進行 scaling，舉例如下：

### 1. 先佈署一個 MediaWiki 應用，並前置 Load Balance 服務

``` bash
$ juju deploy haproxy
$ juju deploy mediawiki
$ juju deploy mysql
$ juju add-relation mediawiki:db mysql
$ juju add-relation mediawiki haproxy
$ juju expose haproxy
```

### 2. 擴展 MediaWiki

``` bash
# 以五台機器為例
$ juju add-unit -n 5 mediawiki
```

如此一來就完成水平擴展了，整個步驟相當簡單。

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
