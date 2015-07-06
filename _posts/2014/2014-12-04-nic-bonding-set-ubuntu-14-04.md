---
layout: post
title:  "[Ubuntu] 設定 NIC Bonding"
date:   2014-12-04 11:20:00
comments: true
categories: [linux]
tags: [Linux, Ubuntu]
---

最近要測試 storage，但因為 server 上的網卡(四張)的速率只有 1Gb，所以決定透過 bonding 的方式將所有網路卡串起來使用，儘量的消除資料傳輸的瓶頸出現在網路的機會....

### 安裝 `ifenslave`

``` bash
$ sudo apt-get -y install ifenslave
```


### 載入 bonding 模組

``` bash
$ sudo modprobe bonding 
```


### 修改 /etc/network/interfaces

假設 server 上網卡的名稱為 `em2`, `em3`, `em4`, `em5`

``` bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

auto em2
iface em2 inet manual
bond-master bond0
#bond-primary em2

auto em3
iface em3 inet manual
bond-master bond0

auto em4
iface em4 inet manual
bond-master bond0

auto em5
iface em5 inet manual
bond-master bond0

# The primary network interface
auto bond0
iface bond0 inet static
        address xxx.xxx.xxx.xxx
        netmask xxx.xxx.xxx.xxx
        network xxx.xxx.xxx.xxx
        broadcast xxx.xxx.xxx.xxx
        gateway xxx.xxx.xxx.xxx
        dns-nameservers xxx.xxx.xxx.xxx yyy.yyy.yyy.yyy
        dns-search your.domain
        bond-slaves none
        bond-mode 6
        bond-miimon 100
```

全部設定好之後，重新啟動 server 即可!


額外附註
=======

1. Switch 端不需要設定 LACP

2. 可透過 `cat /proc/net/bonding/bond0` 指令檢查 bonding 的狀態


參考資料
=======

- [Ubuntu Server 14.04 LTS NIC Bonding](http://www.paulmellors.net/ubuntu-server-14-04-lts-nic-bonding/)

- [Ubuntu bonding 小技巧 » 奇科電腦](http://www.geego.com.tw/technical-discussion-forum/tech-tips-the-tips-of-ubuntu-bonding-ubuntu-%E5%B0%8F%E6%8A%80%E5%B7%A7)

- [Ubuntu 12.04 LTS bonding (LACP) « roan's Blog](http://roan.logdown.com/posts/177335-ubuntu-1204-lts-bonding-lacp)

- [鳥哥的 Linux 私房菜 -- 區域網路的環境設定](http://linux.vbird.org/linux_enterprise/0110network.php)

- [傲笑紅塵路: Linux 網路結合(network bonding)技術與實務](http://www.lijyyh.com/2011/11/0-balance-rr-l-round-robin-salve-salve.html)
