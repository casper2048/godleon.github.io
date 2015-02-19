---
layout: post
title:  "[Vagrant] 初探 Vagrant @ Windows"
description: "初探 Vagrant，嘗試在 Windows 下快速佈署虛擬化環境"
date:   2015-02-18 20:40:00
published: true
comments: true
categories: [vagrant]
tags: [vagrant, Virtualization]
---


環境安裝 & 設定
===============

1. 安裝 [VirtualBox](https://www.virtualbox.org/wiki/Downloads) (使用版本：**4.3.22**)

2. 安裝 [VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads) (使用版本：**4.3.22**)

3. 安裝 Vagrant (使用版本)

4. 安裝 [OpenSSH for Windows](http://www.codedata.com.tw/social-coding/vagrant-tutorial-2-playing-vm-with-vagrant/)，並把 <font color='red'>**C:\Program Files (x86)\OpenSSH for Windows\bin**</font> 加入到環境變數 **path** 中

-----------------------------------

啟動第一個虛擬機
================

首先可以到 [Vagrant Cloud](https://vagrantcloud.com/) 上搜尋合適的 box(其實就是做好某種程度配置的 VM) 來安裝，以下以 **ubuntu/trusty64** 為例：

``` bash
$ mkdir demo-1
$ cd demo-1
demo-1 $ vagrant init ubuntu/trusty64
demo-1 $ vagrant up
```

接著 vagrant 就會自動到 repository 將指定的 box download 回來，並在 VirtualBox 中產生 VM 並啟動。

> 過程中有遇到 Connection Timeout 的錯誤，後來把 GUI 打開檢查後，發現原來我的硬體虛擬化加速功能沒開，導致 VirtualBox 中的 VM 無法正常啟動。(因為我的 Host OS 也是在 vSphere ESXi 的虛擬機，後來把硬體虛擬化加速的功能開啟後就正常了!)

-----------------------------------

簡易指令介紹
============

VM 啟動成功後，可以透過以下指令使用透過 Vagrant 建立起來的虛擬環境：

``` bash
# 登入 VM 
demo-1 $ vagrant ssh

# 檢查 VM OS 版本
vagrant@vagrant-ubuntu-trusty-64:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 14.04.1 LTS
Release:        14.04
Codename:       trusty

# 登出 VM
vagrant@vagrant-ubuntu-trusty-64:~$ exit
logout
Connection to 127.0.0.1 closed.

# 關閉 VM
demo-1 $ vagrant halt
==> default: Attempting graceful shutdown of VM...

# 查詢目前 VM status
demo-1 $ vagrant status

# 查詢目前已經下載的 Vagrant Box
demo-1 $ vagrant box list
```

-----------------------------------

自動化配置設定檔範例
====================

``` ruby
$script = <<SCRIPT
# 更新套件
sudo apt-get update
sudo apt-get -y dist-upgrade
# 安裝 Redis...
sudo apt-get install redis-server -y
# 允許 Redis bind 至全部 network interface...
sudo sed -i -e 's/^bind/#bind/' /etc/redis/redis.conf
# 重啟 Redis，讓新設定生效。
sudo service redis-server restart
SCRIPT

Vagrant.configure(2) do |config|
  # Vagrant Box 名稱
  config.vm.box = "ubuntu/trusty64"

  # port forwarding 設定
  # 從原本的輸出畫面可看出已經有 22 <---> 2222 的設定內建了
  config.vm.network "forwarded_port", guest: 6379, host: 6379

  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # shared folder 設定
  # config.vm.synced_folder "../data", "/vagrant_data"

  # provider-specific 設定 (for VirtualBox)
  config.vm.provider "virtualbox" do |vb|
     # 顯示 VM GUI
     vb.gui = true
	 # 記憶體設定(預設為 512 MB)
	 vb.memory = "1024"
  end
end
```

-----------------------------------

移除 VM 配置
============

假設從 Vagrant Cloud 上安裝了 **ubuntu/trusty64**，目錄名稱為 **demo-1** 為例：

``` bash
$ cd demo-1
 
$ vagrant halt
$ vagrant destroy --force
$ del /F /Q  Vagrantfile
$ rmdir /S /Q  .vagrant
 
$ vagrant box remove ubuntu/trusty64
$ rmdir /S /Q %USERPROFILE%\.vagrant.d\boxes\ubuntu-VAGRANTSLASH-trusty64
```

-----------------------------------

參考資料
========

- [Vagrant Tutorial（1）雲端研發人員，你也需要虛擬機！ by William Yeh | CodeData](http://www.codedata.com.tw/social-coding/vagrant-tutorial-1-developer-and-vm/)

- [Vagrant Tutorial（1）雲端研發人員，你也需要虛擬機！ by William Yeh | CodeData](http://www.codedata.com.tw/social-coding/vagrant-tutorial-2-playing-vm-with-vagrant/)

- [Vagrant Tutorial（3）細說虛擬機生滅狀態 by William Yeh | CodeData](http://www.codedata.com.tw/social-coding/vagrant-tutorial-3-vm-lifecycle/)

- [Vagrant Tutorial（4）虛擬機，若即若離的國中之國 by William Yeh | CodeData](http://www.codedata.com.tw/social-coding/vagrant-tutorial-4-guest-host-communication/)

- [Vagrant Tutorial（5）客製化虛擬機內容的幾種方法 by William Yeh | CodeData](http://www.codedata.com.tw/social-coding/vagrant-tutorial-5-vm-customization/)

- [Vagrantfile - Vagrant Documentation](http://docs.vagrantup.com/v2/vagrantfile/)