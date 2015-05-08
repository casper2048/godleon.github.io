---
layout: post
title:  "[Ubuntu] MAAS(Metal as a Service) 的安裝 & 設定"
description: "在這篇文章中，介紹如何安裝 Ubuntu MAAS，讓管理實體機的工作變得更為方便與彈性，以及介紹在安裝設定上有可能會遇到的問題，並說明如何排除"
date:   2015-05-07 14:40:00
published: true
comments: true
categories: [maas]
tags: [MAAS, Ubuntu, Linux]
---


甚麼是 MAAS (Metal as a Service) ?
==================================

首先來看看 [Ubuntu MAAS 官網](https://maas.ubuntu.com/)的定義：

> Metal as a Service brings the language of the cloud to physical servers. It makes it easy to set up the hardware on which to deploy any service that needs to scale up and down dynamically; a cloud being just one example.

> It lets you provision your servers dynamically, just like cloud instances – only in this case, they’re whole physical nodes. “Add another node to the Hadoop cluster, and make sure it has at least 16GB RAM” is as easy as asking for it.

> With a simple web interface, you can add, commission, update and recycle your servers at will. As your needs change, you can respond rapidly, by adding new nodes and dynamically re-deploying them between services. When the time comes, nodes can be retired for use outside the MAAS.

> MAAS works closely with the service orchestration tool Juju to make deploying services fast, reliable, repeatable and scalable.


簡單來說，MAAS 將原本在雲端環境上快速部屬應用的那一套方法拉到實體機器的層級，讓使用者可以動態的以「實體機器」為單位進行擴展配置。


MAAS 的架構如下圖：

![MAAS Architecture](https://maas.ubuntu.com/wp-content/uploads/2013/01/maas-diagram.png)


--------------------------


環境介紹
========

OS version：**Ubuntu 14.04.02 LTS**

Ubuntu MAAS 的環境安裝於 VirtualBox 上，MAAS server 上只有一張網卡，與本機的網卡 bridge，網路設定如下：

- IP: **192.168.127.100 / 24**

- Gateway: **192.168.127.254**

<font color='red'>**【註】**</font>以上的設定是可以上網的!


--------------------------


安裝 MAAS @ Ubuntu 14.04.02
===========================

### 更新 repository

``` bash
$ sudo add-apt-repository -y ppa:maas-maintainers/stable
$ sudo apt-get -y update 
```

### 安裝 MAAS

``` bash
$ sudo apt-get -y install maas maas-dhcp maas-dns
```

### 建立 super user for MAAS region control

``` bash
$ sudo maas-region-admin createadmin --username=[YOUR_ADMIN_NAME] --password=[ADMIN_PASSWORD] --email=[ADMIN_EMAIL]
```

### 設定 MAAS Web GUI

只有一張網卡的話，可以不用下面的指令：

``` bash
$ sudo dpkg-reconfigure maas-region-controller
```

若是有多張網卡，而 Web Portal 無法連上時，可透過上面的指令修改 MAAS Web portal 的 IP address。

--------------------------

Import Boot Image
=================

此時要透過 Web portal 匯入 Boot Image，步驟如下：

1. 使用剛剛建立的 super user 登入 [MAAS Server Web Portal](http://192.168.127.100/MAAS)

2. 點選 web portal 上方的 <font color='red'>**Images**</font>

3. 選擇要 import 的 image 版本，點選 **Import images**，MAAS server 便會自動到網路上下載相對應的 boot image 並 import

importing 的畫面如下：

![Importing Ubuntu 14.04 LTS amd64 boot image](https://lh3.googleusercontent.com/-0Il8YUzM2jE/VUNYtNdtRgI/AAAAAAAAKyA/PPm0Onzwcxo/w1034-h694-no/1_Importing_Boot_Image.png)

當 import 完成後(這要跑蠻久的)，可以在畫面檢視目前的 boot image 清單：

![Boot image list on MAAS server](https://lh6.googleusercontent.com/-4IUGgqijgXM/VUNYtJfs08I/AAAAAAAAKyA/nT6j4GVIpYo/w1049-h826-no/2_Boot_Image_List.png)

從上面可以知道，MAAS server 需要在可以上網的環境才可以完成這件事情，但如果環境是不允許上網的呢?

那就要必須透過指令(**maas-import-pxe-files**)，並透過 YAML 將 boot image 的 repository 指定在本地端的環境來進行 import boot image 的動作，相關細節可以參考[官方文件的說明](https://maas.ubuntu.com/docs/man/maas-import-pxe-files.8.html)。


--------------------------

Import Boot Image by CLI
========================

若是要將工作自動化，透過 CLI 來完成就是必要的了!

要如何透過 CLI 完成 import boot image 的工作呢? 執行以下指令即可：

``` bash
maas [YOUR_PROFILE_NAME] boot-resources import
```

接著系統就會自動幫我們 import 最新版本的(only amd64) boot image，目前執行結果是取得 `trusty amd64` 版本的 boot image。

### 更改 import boot image 的版本

若要 import 的不是最新的版本，或是可能也需要 i386 的版本呢? 可以透過修改 boot source 來完成，有以下幾個步驟要進行：

##### 1、檢視 boot source 狀態

``` bash
$ maas [YOUR_PROFILE_NAME] boot-sources read
Success.
Machine-readable output follows:
[
    {
        "url": "http://maas.ubuntu.com/images/ephemeral-v2/releases/",
        "keyring_data": "",
        "resource_uri": "/MAAS/api/1.0/boot-sources/1/",
        "keyring_filename": "/usr/share/keyrings/ubuntu-cloudimage-keyring.gpg",
        "id": 1
    }
]
```

看的出來目前系統中僅有一個預設的 boot source，且 ID = 1。

#### 2、讀取 boot source 詳細設定

``` bash
$ maas [YOUR_PROFILE_NAME] boot-source-selections read 1
Success.
Machine-readable output follows:
[
    {
        "labels": [
            "release"
        ],
        "arches": [
            "amd64"
        ],
        "subarches": [
            "*"
        ],
        "release": "trusty",
        "os": "ubuntu",
        "id": 1,
        "resource_uri": "/MAAS/api/1.0/boot-sources/1/selections/1/"
    }
]
```

從上面的內容可以看出，目前只會 import `ubuntu-trusty-amd64-release` 版本的 boot image。

### 3、新增 boot image 版本

``` bash
# 在 boot source(ID=1) 中新增一個不同版本設定的區段(section)
$ maas [YOUR_PROFILE_NAME] boot-source-selections create 1 os="ubuntu" release="precise" arches="amd64" subarches="*" labels= "*"
Success.
Machine-readable output follows:
{
    "labels": [
        "*"
    ],
    "arches": [
        "amd64"
    ],
    "subarches": [
        "*"
    ],
    "release": "precise",
    "os": "ubuntu",
    "id": 2,
    "resource_uri": "/MAAS/api/1.0/boot-sources/1/selections/2/"
}
```

上面指令已經完成，此區段的 ID = 2。

如此一來再重新執行一次 import 的指令，就會把 `ubuntu-precise-amd64` 的 boot image 給 import 進來囉!

--------------------------


DHCP service 設定
=================

在設定 MAAS DHCP service 之前，必須確定要使用 MAAS 管理的 node 必須與 MAAS server NIC 位在同一個實體網段中，目的是：

1. 讓受管理的 node 可以使用到 MAAS DHCP service

2. 讓 MAAS 可以透過 ARP 協定確認到受管理 node 的 MAC & IP address 的對應資訊


### 使用 API Key Login 到 API 介面

``` bash
$ maas login [YOUR_PROFILE_NAME] http://192.168.127.100/MAAS/api/1.0 $(sudo maas-region-admin apikey --username=[YOUR_USER_NAME])

You are now logged in to the MAAS server at
http://192.168.34.10/MAAS/api/1.0/ with the profile name '[YOUR_PROFILE_NAME]'.

For help with the available commands, try:

  maas [YOUR_PROFILE_NAME] --help
```

Login 有幾個部分可以調整：

1. 命令中的 **[YOUR_PROFILE_NAME]** 稱為 profile name，這是可以自定的，使用者可以自定一個比較好記或是其他有意義的名稱

2. API Key 的部分可以透過 standard input，由 file 提供

### 設定 DHCP service 資訊

首先先取得 MAAS server NIC 的設定資訊：

``` bash
# 讀取 MAAS server NIC 設定
$ maas [YOUR_PROFILE_NAME] networks read
Success.
Machine-readable output follows:
[
    {
        "dns_servers": null,
        "name": "maas-eth0",
        "default_gateway": null,
        "ip": "192.168.127.0",
        "netmask": "255.255.255.0",
        "vlan_tag": null,
        "resource_uri": "/MAAS/api/1.0/networks/maas-eth0/",
        "description": "Auto created when creating interface eth0 on cluster maas"
    }
]

```

再來設定 DHCP service 相關參數：

``` bash
# 取得 node group uuid
$ uuid=$(maas [YOUR_PROFILE_NAME] node-groups list | grep uuid | cut -d\" -f4)
b4ec5c4f-7801-4556-b701-bcfcede24019

# 確認 uuid 變數值
$ echo $uuid
b4ec5c4f-7801-4556-b701-bcfcede24019

# 設定 DHCP service 參數
# managemment 參數值意義：0 (no management), 1 (manage DHCP) and 2 (manage DHCP and DNS)
$ maas [YOUR_PROFILE_NAME] node-group-interface update $uuid eth0 ip_range_low=192.168.127.121 ip_range_high=192.168.127.140 static_ip_range_low=192.168.127.221 static_ip_range_high=192.168.127.240 management=2 subnet_mask=255.255.255.0 router_ip=192.168.127.254 broadcast_ip=192.168.127.255
Success.
Machine-readable output follows:
{
    "ip_range_high": "192.168.127.140",
    "ip_range_low": "192.168.127.121",
    "broadcast_ip": "192.168.127.255",
    "static_ip_range_low": "192.168.127.221",
    "name": "eth0",
    "ip": "192.168.127.100",
    "subnet_mask": "255.255.255.0",
    "management": 2,
    "static_ip_range_high": "192.168.127.240",
    "interface": "eth0"
}
```

--------------------------

增加 Node 到 MAAS 中
====================

增加 node(VM) 到 MAAS server 中是很簡單的，只要確定 node NIC 與 MAAS server 提供 DHCP service 的 NIC 可以互通，再設定<font color='blue'>**網路開機**</font>即可。

整個 node 加入到 MAAS server 的流程如下：

1. node 開機，取得來自 MAAS server 的 dhcp 資訊，並使用先前 import 的 boot image 進行 PXE boot

2. node 載入 image 後開機，經過了一長串的開機流程，最後與 MAAS server 溝通完成，成為受 MAAS server 管理的 node 之一

3. node 確定加入 MAAS server，進入關機程序

node 加入到 MAAS server 成功之後，可以在 MAAS web portal 上看到 node 的數量與資訊，例如下圖：

![MAAS server node count](https://lh3.googleusercontent.com/-JF7b6EYFbNU/VUNts8UBa_I/AAAAAAAAKyk/QqGTUrbJzKo/w1050-h788-no/MAAS_node1.png)

![MAAS server node list](https://lh5.googleusercontent.com/-Ak0U4NyHgvs/VUNttOGLRhI/AAAAAAAAKyk/F9X-pidZWYs/w1050-h748-no/MAAS_node2.png)

點進去每個 node，可以看到 node 的相關資訊：

![MAAS node detail info](https://lh3.googleusercontent.com/-YpbcaeBXRSY/VUOEfoWsN1I/AAAAAAAAKzE/k-wSWPQXOHs/w1040-h831-no/maas_node_detail.png)

### 這個階段完成了什麼工作? 

1. 在這個階段中，每台透過 PXE boot 的 node，會取得來自 MAAS server 之前所 import 的 boot image，並執行一段開機流程

2. 在開機的過程中，<font color='red'>**每個 node 會告知 MAAS server 如何遠端的對它進行電源的管理**</font>的相關資訊。

若是在實體環境，可能會取得像是 IPMI 的資訊，例如下圖：

![IPMI information on MAAS server](https://lh5.googleusercontent.com/-5hdeWh4wqB4/VUsEFoCOpaI/AAAAAAAAKzw/j6FA6dvozls/w490-h567-no/MAAS_edit_node.png)

但若是在虛擬機上呢? <font color='blue'>**抱歉....抓不到任何資料**</font>

那若是用虛擬環境怎麼辦呢? 後來查了一下，如果 Host OS 是 Linux 的話，應該可以透過 [virsh](http://libvirt.org/virshcmdref.html) 來解決，讓 MAAS server 懂得怎麼呼叫 Virtual Host(例如：VirtualBox、VMware ... 等等)，讓指定的 VM 開機。([官網上有簡單的介紹如何新增 virtual machine node](http://maas.ubuntu.com/docs1.5/nodes.html#virtual-machine-nodes))

但如果 Host 不是在 Linux，就.......<font color='red'>**手動把 VM 開機也行**</font>。


--------------------------


Commission Node
===============

最後要 commission node，點選特定的 node，進入到 node 詳細畫面中，點選右邊 **Actions** -> **Commission node** 即可。

若是在虛擬環境中，若沒用 [virsh](http://libvirt.org/virshcmdref.html) 處理好，就手動將指定的虛擬機開機吧! 這樣其實也可以.....

若是在實體機，應該是都不會有甚麼問題才對.....

## 這個階段完成了什麼工作? 

此時 node 會再一次的開機，並載入 boot image 之後，回報更多資訊給 MAAS server，例如：CPU core 數量、記憶體大小、硬碟大小 ... 等等。

所有 node 都完成 commission 的動作後，應該會有類似下面圖中的詳細資料：

![MAAS node list after commissioning](https://lh4.googleusercontent.com/-jco2tiGOSIw/VUsH8IA-p1I/AAAAAAAAK0I/nWbCdNY3Qqo/w974-h567-no/MAAS_node_detail_list.png)


--------------------------


參考資料
========

- [MAAS - Metal as a Service](https://maas.ubuntu.com/)

- [MAAS: Metal As A Service — MAAS dev documentation](https://maas.ubuntu.com/docs/index.html)

- [Installing MAAS](http://people.canonical.com/~gavin/maas/building-packages/install.html)

- [Boot images import configuration — MAAS dev documentation](https://maas.ubuntu.com/docs/bootsources.html)