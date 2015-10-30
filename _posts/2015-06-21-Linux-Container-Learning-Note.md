---
layout: post
title:  "[LXC] Linux Container(LXC) 學習筆記"
description: "專門學習 LXC 的筆記"
date: 2015-06-21 18:55:00
published: true
comments: true
categories: [lxc]
tags: [Linux, Container]
---

安裝 LXC @Ubuntu 14.04.3
========================

使用以下指令安裝 LXC 相關的套件：

```bash
$ sudo apt-get -y install lxc lxc-templates bridge-utils debootstrap
```

--------------------------------------------------

操作 Linux Container
====================

### 建立 Container

透過 <font color='blue'>**lxc-create**</font> 指令，建立全新的 container：

``` bash
# type = ubuntu
# name = lxc_ubuntu
$ sudo lxc-create -t ubuntu -n lxc_ubuntu
......
......
...... (很多訊息)
##
# The default user is 'ubuntu' with password 'ubuntu'!
# Use the 'sudo' command to run tasks as root in the container.
##
```

若 container type 是 ubuntu，則預設的帳號密碼皆為 <font color='blue'>**ubuntu**</font>。

--------------------------------------------------

### 查詢 container 結構

安裝好的 container 都會放在 <font color='red'>**/var/lib/lxc**</font> 目錄中：

``` bash
$ sudo tree -L 2 /var/lib/lxc/lxc_ubuntu
/var/lib/lxc/lxc_ubuntu
├── config
├── fstab
└── rootfs
    ├── bin
    ├── boot
    ├── dev
    ├── etc
    ├── home
    ├── lib
    ├── lib64
    ├── media
    ├── mnt
    ├── opt
    ├── proc
    ├── root
    ├── run
    ├── sbin
    ├── srv
    ├── sys
    ├── tmp
    ├── usr
    └── var
```

### 啟動並登入 container

透過 <font color='blue'>**lxc-start**</font> 指令，啟動並登入 container：

``` bash
$ sudo lxc-start -n lxc_ubuntu
......
......
...... (很多訊息)
Ubuntu 14.04.2 LTS lxc_ubuntu console

lxc_ubuntu login: ubuntu
Password:
Welcome to Ubuntu 14.04.2 LTS (GNU/Linux 3.13.0-52-generic x86_64)
```

### 以背景方式啟動 container

加上 <font color='blue'>**-d**</font> 參數，就可以讓 container 在背景啟動：

``` bash
$ sudo lxc-start -n lxc_ubuntu -d
```

接著我們可以透過 <font color='blue'>**lxc-console**</font> 指令來登入背景啟動的 container：

``` bash
$ sudo lxc-console -n lxc_ubuntu
```

### 複製 container

透過 <font color='blue'>**lxc-clone**</font> 指令，可以快速地複製出 container：

```
# src = lxc_ubuntu
# clone = lxc_ubuntu_clone
$ sudo lxc-clone -o lxc_ubuntu -n lxc_ubuntu_clone
```

### 移除 container

透過 <font color='blue'>**lxc-destroy**</font> 指令移除 container：

``` bash
$ sudo lxc-destroy -n lxc_ubuntu_clone
```


--------------------------------------------------

基本網路設定
============

### lxcbr0

安裝好 lxc 套件後，系統會自動產生一個 <font color='red'>**lxcbr0**</font> 作為管理 container 網路之用。

可以透過以下指令看到 lxcbr0 的 process 資訊：

``` bash
$ ps -aux | grep lxcbr0
qct      13511  0.0  0.0  11748  2208 pts/1    S+   15:14   0:00 grep --color=auto lxcbr0
lxc-dns+ 35285  0.0  0.0  28212  2364 ?        S    13:53   0:00 dnsmasq -u lxc-dnsmasq --strict-order --bind-interfaces --pid-file=/run/lxc/dnsmasq.pid --conf-file= --listen-address 10.0.3.1 --dhcp-range 10.0.3.2,10.0.3.254 --dhcp-lease-max=253 --dhcp-no-override --except-interface=lo --interface=lxcbr0 --dhcp-leasefile=/var/lib/misc/dnsmasq.lxcbr0.leases --dhcp-authoritative
```

可以看的出來 lxcbr0 中跑了一個 dhcp server(包含派發的 ip 範圍)，並設定 gateway 為 10.0.3.1。

若要設定 lxcbr0 的 dhcp 設定，可以修改 <font color='blue'>**/etc/default/lxc-net**</font>

修改完後使用以下指令重新啟動即可套用新設定：

``` bash
$ sudo service lxc-net restart
```

### 建立新的 bridge device 並與 container 綁定

#### 1、建立 bridge device

``` bash
# bridge device name = br_private
$ sudo brctl addbr br_private
```

#### 2、啟動 bridge device

``` bash
$ sudo ifconfig br_private up
```

#### 3、檢視新的 bridge device 狀態

``` bash
$ sudo brctl show
[sudo] password for qct:
bridge name     bridge id               STP enabled     interfaces
br_private      8000.fe642c445344       no
lxcbr0          8000.000000000000       no
```

可以看到目前沒有任何 nic 與新的 bridge device 綁定。

#### 4、修改 container config (/var/lib/lxc/lxc_ubuntu/config)

``` bash
# 原始設定
#lxc.network.type = veth
#lxc.network.flags = up
#lxc.network.link = lxcbr0
#lxc.network.hwaddr = 00:16:3e:d3:22:54

# 改成以下設定
lxc.network.type = veth
lxc.network.flags = up
lxc.network.link = br_private
lxc.network.ipv4 = 172.16.100.11/24
lxc.network.hwaddr = 00:16:3e:d3:22:54
```

如此一來，將 container 重新啟動之後，container 就會與新的 bridge device 綁定並改為新的 ip。

#### 5、檢視新的 bridge device 狀態

``` bash
$ sudo brctl show
[sudo] password for qct:
bridge name     bridge id               STP enabled     interfaces
br_private      8000.fe642c445344       no              veth47MR72
                                                        vethWNVYY0
lxcbr0          8000.000000000000       no
```

因為我在系統裡面啟動了兩個 container，所以可以看得出這兩個 container 的 nic 都已經跟新的 bridge device 綁定了!

--------------------------------------------------

連接跨 Host 的 container
=========================

#### 1、安裝 open vswitch

``` bash
$ sudo apt-get install openvswitch-controller openvswitch-switch openvswitch-datapath-source
```

#### 1、建立 bridge device

在兩台 host 上執行以下指令：

``` bash
$ sudo brctl addbr superbr0
```

#### 2、建立 GRE tap device

分別在不同的 host 執行指令：

``` bash
# @host01
$ sudo ip link add testgre type gretap remote 10.5.23.52 local 10.5.23.51 ttl 255

# @host02
$ sudo ip link add testgre type gretap remote 10.5.23.51 local 10.5.23.52 ttl 255

```

#### 3、連結 bridge device & GRE tap device

在兩台 host 上執行以下指令：

``` bash
$ sudo brctl addif superbr0 testgre
```


# Host A

``` bash
$ sudo ifconfig superbr0 192.168.20.1 netmask 255.255.255.0 up
```


``` bash
$ sudo dnsmasq --strict-order --bind-interfaces --pid-file=/var/run/dnsmasq-superbr0.pid --listen-address 192.168.20.1 --dhcp-range 192.168.20.2,192.168.20.254 --dhcp-lease-max=253 --dhcp-no-override --interface=superbr0
```


參考資料
========

### 基本概念

- [淺談 Linux Containers](http://fourdollars.github.io/lxc-intro)

- [TechTalk 專訪 Episode 27 逐字稿 – Linux Container (LXC) by TechTalk@TW | CodeData](http://www.codedata.com.tw/social-coding/techtalk-episode27-linux-container/)

- [A Brief Introduction to Linux Containers with LXC · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2013/11/25/a-brief-introduction-to-linux-containers-with-lxc/)

### 網路概念

- [雲端資料中心開放式與虛擬化網路環境的實踐— 軟體定義網路(Software Defined Network, SDN)](http://www.ringline.com.tw/index.php/zh/support/techpapers/cloud-storage-virtualization/729--software-defined-network-sdn.html)

- [VXLAN簡明學習筆記(原創)_我們關註網](https://wefollownews.appspot.com/cittopnews201408_64/9964.html)

- [overlay網路技術之VXLAN詳解|網路管理 - 開源互助社區](http://www.coctec.com/subject/about/68146.html)

- [網路虛擬化NSX技術文章系列二：採用傳統網路安全架構於軟體定義資料中心的問題](https://www.facebook.com/notes/vmware-taiwan/319602371581924)

- [網路虛擬化NSX 技術文章系列七：邏輯交換器：什麼是VXLAN](https://www.facebook.com/notes/vmware-taiwan/332338360308325)

- [網路虛擬化NSX 技術文章系列八：邏輯交換器：邏輯交換器的運作方式_Part I](https://www.facebook.com/notes/vmware-taiwan/334705080071653)

- [網路虛擬化NSX 技術文章系列九：邏輯交換器：邏輯交換器的運作方式_Part II](https://www.facebook.com/notes/vmware-taiwan/339982132877281)

- [Generic Routing Encapsulation « Levi's Blog](http://levichen.logdown.com/posts/2013/11/26/generic-routing-encapsulation)

- [Linux 2.4 Advanced Routing HOWTO: GRE 及其它通道技術 (GRE and other tunnels)](http://www.linux.org.tw/CLDP/OLD/Adv-Routing-HOWTO-5.html)

### Linux Network Namespace

- [2013-09-24 Introducing Linux Network Namespaces · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)

- [Linux Network Namespaces | Open Cloud Blog](http://www.opencloudblog.com/?p=42)

### Linux Bridge

- [程式扎記: [Linux 文章收集] Linux Bridge With ‘brctl’ Tutorial](http://puremonkey2010.blogspot.tw/2015/03/linux-linux-bridge-with-brctl-tutorial.html)

- [Maxsolar's Linux Blog: (9) NIC bridging concepts](http://maxubuntu.blogspot.tw/2012/10/9-nic-bridging-concepts.html)


### Open vSwitch

- [苦命工程師: [轉載] 雲中的網絡：Open vSwitch帶來的巨變](http://adaam-tw.blogspot.tw/2012/10/open-vswitch.html)

- [Open vSwitch 架構概觀 | 阿喵就像家](http://mlwmlw.org/2012/04/open-vswitch-component-overview/)

- [OpenvSwitch Debug$ Enviroment « roan's Blog](http://roan.logdown.com/posts/238771-openvswitch-debug-enviroment)

### 實作相關

- [Ubuntu 14.04 » Ubuntu 伺服器指南 » Virtualization » LXC](https://help.ubuntu.com/lts/serverguide/lxc.html)

- [Linux KVM 研究室 - 第一章 認識與安裝 Linux Container](http://tobala.net/download/lxc/ch01.pdf)

- [Linux KVM 研究室 - 第二章 使用 Linux Container 虛擬電腦](http://tobala.net/download/lxc/ch02.pdf)

- [Linux KVM 研究室 - 第三章 Linux Container 虛擬網路管理](http://tobala.net/download/lxc/ch03.pdf)

- [Linux KVM 研究室 - 第四章 LXC 獨佔實體網路卡](http://tobala.net/download/lxc/ch04.pdf)

- [Flockport - LXC networking guide](https://www.flockport.com/lxc-networking-guide/)

- [Linux Containers - LXC - Articles](https://linuxcontainers.org/lxc/articles/)

- [Flockport - LXC advanced networking guide](https://www.flockport.com/lxc-advanced-networking-guide/)

- [Flockport - Flockport Labs - Extending layer 2 across container hosts](https://www.flockport.com/flockport-labs-extending-layer-2-across-container-hosts/)

- [Connecting containers on several hosts with Open vSwitch | S3hh's Blog](https://s3hh.wordpress.com/2012/05/28/connecting-containers-on-several-hosts-with-open-vswitch/)

- [LXC, Open vSwitch, and GRE Tunnels · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2013/11/26/lxc-open-vswitch-and-gre-tunnels/)

- [Linux 雲端運算進階(三部曲) - 核心容器系統 - V1.0 (2012/08/01) 由大福知識聯盟設計與維護](http://tobala.net/x/Cloud2010/CloudTech-ADV03-201101.html)

#### Others

[2013-11-25 A Brief Introduction to Linux Containers with LXC · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2013/11/25/a-brief-introduction-to-linux-containers-with-lxc/)

[2013-11-26 LXC, Open vSwitch, and GRE Tunnels · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2013/11/26/lxc-open-vswitch-and-gre-tunnels/)

[2013-11-27 Linux Containers via LXC and Libvirt · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2013/11/27/linux-containers-via-lxc-and-libvirt/)

[2013-12-03 Connecting LXC to Open vSwitch Using Libvirt · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2013/12/03/connecting-lxc-to-open-vswitch-using-libvirt/)

[2014-01-12 Automatically Connecting LXC to Open vSwitch · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2014/01/23/automatically-connecting-lxc-to-open-vswitch/)

[2013-12-18 Managing Open vSwitch with Puppet · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2013/12/18/managing-open-vswitch-with-puppet/)

[2015-05-06 A Quick Introduction to LXD · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2015/05/06/quick-intro-lxd/)
