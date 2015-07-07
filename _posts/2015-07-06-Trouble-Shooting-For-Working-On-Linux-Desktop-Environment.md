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
