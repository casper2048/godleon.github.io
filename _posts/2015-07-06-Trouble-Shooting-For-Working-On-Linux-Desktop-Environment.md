---
layout: post
title:  "[Linux] 使用 Linux(Linux Mint) 作為桌面環境的 trouble shooting 紀錄 "
description: "在這邊文章中，會陸陸續續紀錄使用 Linux 作為 Desktop 環境時所遇到的 bug & trouble shooting 的內容"
date: 2015-07-06 16:00:00
published: true
comments: true
categories: [linux]
tags: [Linux]
---

環境說明
=======

OS：[Linux Mint](http://linuxmint.com/) 17.2 Cinnamon

-----------------------------------

輸入法
======

## 安裝中文輸入法

- [Howard 的記事本: Linux Mint 17 安裝輸入法](http://howardnote.blogspot.tw/2015/01/linux-mint-17.html)

- [Linux Mint 16 更換ibus輸入模組 « Scott's Notebook](http://scotthsieh.logdown.com/posts/184538-linux-mint-16-replacing-the-ibus-input-module)

-----------------------------------

虛擬化相關
=========

## 安裝 VirtualBox + Vagrant

### 出現 the "vboxsf" file system is not available

vagrant up 後，出現以下訊息：

> the "vboxsf" file system is not available. Please verify thatthe guest additions are properly installed in the guest andcan work properly

<font color='red'>**解決方式：**</font>

1. 從 VirtualBox 官網下載 deb & Extension Pack 進行手動安裝

2. 安裝 Vagrant Plugin 如下：

``` bash
$ sudo vagrant plugin install vagrant-vbguest
```

- [ubuntu - Vagrant error : Failed to mount folders in Linux guest - Stack Overflow](http://stackoverflow.com/questions/22717428/vagrant-error-failed-to-mount-folders-in-linux-guest)

- [Error: vagrant failed to mount folders in Linux guest. · Issue #5503 · mitchellh/vagrant · GitHub](https://github.com/mitchellh/vagrant/issues/5503)


## LXC (Linux Container)

- [14.04 - Share a folder between host and ubuntu lxc container - Ask Ubuntu](http://askubuntu.com/questions/610513/share-a-folder-between-host-and-ubuntu-lxc-container)


-----------------------------------

系統工具
=======

## 製作安裝 Linux 用 USB

1. 下載 Linux ISO (e.g. <font color='blue'>**ubuntu-14.04.2-server-amd64.iso**</font>)

2. 透過 `sudo fdisk -l` 確認 USB disk 的 device name (e.g. /dev/sdb)

3. 執行以下指令，卸載 usb disk，並將 ISO 解壓縮到 USB disk 中

``` bash
$ sudo umount $(mount | grep sdb | awk '{print $3}')
$ sudo sh -c "cat ubuntu-14.04.2-server-amd64.iso > /dev/sdb" && sync && sync && sync
```

## 掃描網路

- [Nmap 網路診斷工具基本使用技巧與教學 - G. T. Wang](http://blogger.gtwang.org/2014/10/nmap-command-examples-tutorials.html)

## 清除 ARP cache

``` bash
$ sudo ip -s -s neigh flush all
```

- [clearing the arp cache in linux » Linux Shtuff](http://g33kinfo.com/info/archives/4356)

-----------------------------------

網路
===

## L2TP + IPSec VPN

安裝套件：**l2tp-ipsec-vpn**

接著按照以下連結安裝 gnome network manager for L2TP

- [networking - vpn l2tp connection on ubuntu 14.10 - Ask Ubuntu](http://askubuntu.com/questions/581981/vpn-l2tp-connection-on-ubuntu-14-10)

【註】設定 IPSec Pre-shared Key 之後就無法連，暫時先不設定，問題之後再找時間排除

## PPTP VPN

- [[HowTo] PPTP: Ubuntu Client 連接到 Windows VPN Server [論壇 - Ubuntu基本設定] | Ubuntu 正體中文站](http://www.ubuntu-tw.org/modules/newbb/viewtopic.php?topic_id=40626)

### Ubuntu 12.04

- [第二十四個夏天後: [Linux] VPN L2TP client mode @ Ubuntu 12.04 Desktop](http://blog.changyy.org/2014/02/linux-vpn-l2tp-client-mode-ubuntu-1204.html)


## 測試網路速度 (iperf)

- [利用 iperf 測試網路效能 - 可丁丹尼@一路往前走2.0](http://cms.35g.tw/coding/%E5%88%A9%E7%94%A8-iperf-%E6%B8%AC%E8%A9%A6%E7%B6%B2%E8%B7%AF%E6%95%88%E8%83%BD/)


## 修改 Default gateway

``` bash
# sample
$ sudo ip route change default via 192.168.10.1 dev eth0
```

## 遠端連線工具

- [Connect to a Windows PC from Ubuntu via Remote Desktop Connection](http://www.digitalcitizen.life/connecting-windows-remote-desktop-ubuntu)

-----------------------------------

通訊軟體
=======

## Line

- [ubuntu 安裝 line @ 我的生活 :: 痞客邦 PIXNET ::](http://yyyfly.pixnet.net/blog/post/58092096)

- [關於 LINE 的安裝 [論壇 - Ubuntu安裝問題] | Ubuntu 正體中文站](http://www.ubuntu-tw.org/modules/newbb/viewtopic.php?viewmode=flat&order=DESC&topic_id=50062&forum=1)


-----------------------------------


文書編輯
=======

## Adobe Reader

- [Install Adobe Reader in Ubuntu 14.04 and Ubuntu 14.10](http://sourcedigit.com/15444-install-adobe-reader-in-ubuntu-14-04-and-ubuntu-14-10/)

## 列印 PDF

- [阿剛老師的異想世界: ubuntu軟體教室--在ubuntu上安裝一台可輸出PDF的虛擬印表機](http://kentxchang.blogspot.com/2010/12/ubuntu-ubuntupdf.html)


-----------------------------------


瀏覽器(Browser)
==============

## 更換 Firefox 預設搜尋引擎

- [上鎖者: Linux Mint 13 更換 Firefox 預設搜尋引擎成為 Google](http://way3sec.blogspot.tw/2012/08/linux-mint-13-firefox-google.html)


-----------------------------------


Sublime Text 3
==============

## 安裝

- [How to Install Sublime Text 3 in Ubuntu 14.04 Trusty | UbuntuHandbook](http://ubuntuhandbook.org/index.php/2013/12/install-sublime-text-3-ubuntu-14-04-trusty/)


-----------------------------------


Atom
====

- [在 Ubuntu 14.04 和 Linux Mint 17 上安装 Atom 文本编辑器-技术 ◆ 学习|Linux.中国-开源中文社区](https://linux.cn/article-3663-1.html)

- [Atom 的中文顯示框框問題 ~ Open Jiang](http://jeffyon.blogspot.tw/2015/05/atom-md.html)
