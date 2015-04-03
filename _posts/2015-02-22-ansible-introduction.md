---
layout: post
title:  "[Ansible] 初探 Ansible"
description: "初探 Ansible，並嘗試安裝，進行簡單測試"
date:   2015-02-22 16:25:00
published: true
comments: true
categories: [ansible]
tags: [Ansible, Linux]
---

Ansible 是什麼?
===============

以下來自官網的定義：

> Ansible is the simplest way to automate apps and IT infrastructure. Application Deployment + Configuration Management + Continuous Delivery.

Ansible 是一個 python 為基礎的自動化配置與部署 IT 架構的工具，達成 Infrastructure as code 的目標，可以很快速地為系統管理者(or 開發者)佈署出一致的環境；這樣的工具一般多用在大量的伺服器環境管理上

-------------------------------

Why Ansible ?
=============

IT Automation 的工具很多，除了 Ansible 之外，還有像是 Puppet / Chef / Salt .... 等等，而 Ansible 的優點在哪裡呢?

Ansible 與其他工具的差別在於，Ansible 是屬於 <font color='red'>**push-based**</font>，其他則屬於 <font color='red'>**pull-based**</font>。

Ansible 是透過 SSH 對 remote server 進行控制 & 設定，這也就是為什麼 Ansible 可以不需要在各 remote server 安裝 agent 而其他工具需要的原因。

也因為這樣的設計，讓使用 Ansible 所需要的前置作業少了很多，也相對簡單。

-------------------------------

安裝 Ansible
============

Ansible 預設是使用 SSH 去管理其他遠端的機器，因此只要安裝在一台機器上作為中央控管即可(<font color='red'>**control manchine**</font>)，但由於 Ansible 是用 Python 開發的，因此機器上必須有安裝 Python 2.6。

> 目前不支援 Windows 作為 Control Machine

被管理的機器稱為 <font color='red'>**managed node**</font>，上面需要安裝 Python 2.4 以上的版本；但如果安裝的 Python 版本低於 2.5，則必須要另外安裝 `python-simplejson` 套件。

若是在 managed node 上有啟用 SELinux，那就必須額外再安裝 `libselinux-python` 套件，Ansible 的功能才能正確使用。

在 <font color='red'>**control node**</font> 上安裝 Ansible 套件：

``` bash
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
```

-------------------------------

開始使用 Ansible
================

## 遠端連線上需注意的事項

Ansible 1.3 之後預設使用 OpenSSH 作為遠，端溝通之用，若是比較舊版的 Linux 版本可能不支援新版 OpenSSH 中的 ControlPersist 功能，此時可以改用以 Python 實作出來的 ssh([paramiko](http://www.paramiko.org/)) 作為取代 OpenSSH 之用。

此外，與遠端機器連線一般都會使用 SSH Key pair 作為認證之用，但也可以使用 `--ask-pass` & `--ask-sudo-pass` 改用密碼的方式進行認證。

## Hello Ansible

### 設定 SSH Key Authentication

1、在 control node 上產生 ssh key pair

``` bash
# 不要設定密碼，使用 ssh key 登入時才不需額外輸入一次密碼
# 假設 ssh key pair 存放在 ~/.ssh 目錄下
$ ssh-keygen
```

上面的指令會在 **~/.ssh** 目錄中產生 `id_rsa`(private key) & `id_rsa.pub`(public key) 兩個檔案。

2、將 public key 傳到 remote managed node 上

``` bash
$ ssh [USER_NAME]@[HOST_ADDRESS] 'mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```

3、透過以下指令免密碼登入遠端主機

``` bash
$ ssh [USER_NAME]@[HOST_ADDRESS]
```

### 設定 Ansible Invetory File

接著我們要設定 Ansible Invetory File，指定要進行遠端管理控制的主機有哪些，編輯 **/etc/ansible/hosts**，設定內容如下：

``` bash
# 假設我們設定的遠端主機 ip 分別為 192.168.33.20 & 192.168.33.21
# 且必須在這兩台機器上設定好 SSH Key Authentication
192.168.33.20
192.168.33.21
```

### 測試遠端主機回應

透過以下指令測試 Ansible 是否可以 work：

``` bash
# 使用 ping 模組測試 managed node 是否有回應
$ ansible all -m ping
192.168.33.20 | success >> {
    "changed": false,
    "ping": "pong"
}

192.168.33.21 | success >> {
    "changed": false,
    "ping": "pong"
}

# 以 vagrant 的身分執行 ping 模組功能
$ ansible all -m ping -u bruce

# 以 vagrant 身分登入，並切換成 root 執行 ping 模組功能 (執行 module 使用 -m 參數)
$ ansible all -m ping -u bruce --sudo

# 以 vagrant 身分登入，並切換成 root 執行更新 APT 套件資訊(執行命令使用 -a 參數)
$ ansible all -a "apt-get update" -u vagrant --sudo
```

看到遠端主機的回應，就表示我們到目前為止的設定算是正確無誤囉!

-------------------------------

參考資料
========

- [Ansible is Simple IT Automation](http://www.ansible.com/)

- [ssh keygen 免輸入密碼 - Tsung's Blog](http://blog.longwin.com.tw/2005/12/ssh_keygen_no_passwd/)

- [SSH 公開金鑰認證（Public Key Authentication）：不用打密碼登入 Linux，安全又方便 - G. T. Wang](http://www.gtwang.org/2014/05/linux-ssh-public-key-authentication.html)



