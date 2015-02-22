---
layout: post
title:  "[Ansible] 使用 module & ad-hoc command"
description: "在使用 playbook 之前，在 Ansible 中使用 module 功能，並執行 ad-hoc command，直接從 control node 對 managed node 下達命令"
date:   2015-02-22 23:00:00
published: true
comments: true
categories: [ansible]
tags: [Ansible, Linux]
---

ad-hoc command 是什麼?
======================

以下是 ad-hoc command 在官網的定義：

> An ad-hoc command is something that you might type in to do something really quick, but don’t want to save for later.

其實 ad-hoc command 就是直接執行的指令，而非定義在 playbook 內的命令，當要發布的指令很簡單時(例如：重開機)，與其寫一個 playbook 來執行，不如直接下一個直接的指令來的快。

----------------------------

module 是什麼?
==============

module 在 Ansible 中的功能就是可用來在本地端 or 遠端管理的機器上執行各式各樣的管理工作，包含使用者管理、套件管理、服務管理、檔案管理 .... 等功能。

當然也可以開發屬於自己使用的 module，詳細的資料可以參考[官方網站](http://docs.ansible.com/modules.html)。

----------------------------

調整 Invetory File
===================

為了方便後面 command 的示範，先將 invetory file(**/etc/ansible/hosts)** 調整如下：

``` ini
[group1]
md1 ansible_ssh_host=192.168.33.20
md2 ansible_ssh_host=192.168.33.21
```

----------------------------

同步執行 & Shell Commands
=========================

#### 1、將 md1 主機重新開機

``` bash
$ ansible md1 -a "/sbin/reboot" --sudo
```

#### 2、將 group1 內的主機重開機

``` bash
# 同時執行 2 個執行緒呼叫遠端主機重開機
# 不使用 -f 2 也可以，預設是 -f 5 (5 個執行緒)
$ ansible group1 -a "/sbin/reboot" -f 2 --sudo
```

> 若沒有指定 -f 參數，則預設使用的執行緒為 5 個

#### 3、使用 SHELL module，執行命令查詢 Linux kernel 版本低於

``` bash
$ ansible group1 -m shell -a "uname -r"
md2 | success | rc=0 >>
3.13.0-45-generic

md1 | success | rc=0 >>
3.13.0-45-generic
```

----------------------------

File Transfer (使用 copy module)
================================

#### 將 control node 的檔案透過 SCP 複製到 managed nodes 上

``` bash
$ ansible group1 -m copy -a "src=/etc/hosts dest=/tmp/hosts"
md1 | success >> {
    "changed": true,
    "checksum": "254f398edc4258429dec5cd1f57f52ca0af88ffd",
    "dest": "/tmp/hosts",
    "gid": 1000,
    "group": "vagrant",
    "md5sum": "43667f0d1c5ebcd0c8e05219ca6a2a4e",
    "mode": "0664",
    "owner": "vagrant",
    "size": 257,
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1424597427.02-112457028174613/source",
    "state": "file",
    "uid": 1000
}

md2 | success >> {
    "changed": true,
    "checksum": "254f398edc4258429dec5cd1f57f52ca0af88ffd",
    "dest": "/tmp/hosts",
    "gid": 1000,
    "group": "vagrant",
    "md5sum": "43667f0d1c5ebcd0c8e05219ca6a2a4e",
    "mode": "0664",
    "owner": "vagrant",
    "size": 257,
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1424597427.03-76352406601557/source",
    "state": "file",
    "uid": 1000
}
```

----------------------------

目錄/檔案管理 (使用 file module)
================================

#### 1、變更檔案的 permission & Owner & Group

``` bash
$ ansible md1 -m file -a "dest=/tmp/changeme mode=600 owner=puppet group=puppet" --sudo
md1 | success >> {
    "changed": true,
    "gid": 112,
    "group": "puppet",
    "mode": "0600",
    "owner": "puppet",
    "path": "/tmp/changeme",
    "size": 0,
    "state": "file",
    "uid": 107
}
```

#### 2、建立目錄，指定 permission & Owner & Group

``` bash
$ ansible md1 -m file -a "dest=/tmp/dir/nextdir mode=755 owner=root gr
oup=root state=directory" --sudo
md1 | success >> {
    "changed": true,
    "gid": 0,
    "group": "root",
    "mode": "0755",
    "owner": "root",
    "path": "/tmp/dir/nextdir",
    "size": 4096,
    "state": "directory",
    "uid": 0
}
```

#### 3、刪除目錄

``` bash
$ ansible md1 -m file -a "dest=/tmp/dir state=absent" --sudo
md1 | success >> {
    "changed": false,
    "path": "/tmp/dir",
    "state": "absent"
}
```

-------------------------------

套件管理 (使用 apt module)
==========================

#### 1、使用 APT 安裝最新套件

``` bash
$ ansible md1 -m apt -a "name=redis-server state=latest" --sudo
md1 | success >> {
    "changed": true,
    "stderr": "",
    "stdout": "a lot of messages about package installation....."
}
```

#### 2、安裝指定版本套件

``` bash
$ ansible md1 -m apt -a "name=openjdk-6-jre state=present" --sudo
```

#### 3、移除指定套件

``` bash
$ ansible md1 -m apt -a "name=openjdk-6-jre state=absent" --sudo
```

-------------------------------

使用者 & 群組管理 (使用 user module)
====================================

#### 1、新增使用者

``` bash
$ ansible md1 -m user -a "name=godleon password=1234567" --sudo
md1 | success >> {
    "changed": true,
    "comment": "",
    "createhome": true,
    "group": 1002,
    "home": "/home/godleon",
    "name": "godleon",
    "password": "NOT_LOGGING_PASSWORD",
    "shell": "",
    "state": "present",
    "system": false,
    "uid": 1002
}
```

#### 2、刪除使用者

``` bash
$ ansible md1 -m user -a "name=godleon state=absent" --sudo
md1 | success >> {
    "changed": true,
    "force": false,
    "name": "godleon",
    "remove": false,
    "state": "absent"
}
```

-------------------------------

Service 管理 (使用 service module)
==================================

#### 重新啟動服務

``` bash
$ ansible md1 -m service -a "name=redis-server state=restarted" --sudo

md1 | success >> {
    "changed": true,
    "name": "redis-server",
    "state": "started"
}
```

若要啟動服務，則 `state=started`；停止服務則用 `state=stopped`

-------------------------------

總結
====

從上面的範例中可以看出 Ansible 的作業方式：

1. 由 Invetory File 中取得可以管理的遠端主機清單(managed nodes)

2. 使用各種 module 功能進行配置(檔案管理 / 使用者管理 / 套件管理 / 服務管理 .... 等等)

3. 後續還可透過 playbook(劇本) 的方式，將所需要自動化完成的工作，透過 module 組合起來，並透過配置檔快速佈署環境。

Ansible 運作的大痣方式就是如以上所述，其他細節的部份就留待後面研究後再來分享囉!

-------------------------------

參考資料
========

- [Introduction To Ad-Hoc Commands — Ansible Documentation](http://docs.ansible.com/intro_adhoc.html)

- [About Modules — Ansible Documentation](http://docs.ansible.com/modules.html)

- [Module Index — Ansible Documentation](http://docs.ansible.com/modules_by_category.html)