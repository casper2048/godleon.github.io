---
layout: post
title:  "[Ansible] Inventory 使用介紹"
description: "在這邊文章中，介紹如何撰寫 Inventory file，讓 Ansible 可以對多台機器同時進行操作"
date:   2015-04-3 11:14:00
published: true
comments: true
categories: [ansible]
tags: [Ansible, Linux]
---

{% assign db_primary_host = '{{ db_primary_host }}' %}
{% assign db_primary_host_yaml = '{{ db["primary"]["host"] }}' %}

測試環境說明
============

我們透過 [Vagrant](https://www.vagrantup.com/) 建立了三台 VM，名稱與 IP 分別如下：

- vagrant1 (192.168.30.11)

- vagrant2 (192.168.30.12)

- vagrant3 (192.168.30.13)

--------------------------------------------

Inventory 入門
==============

為了讓 control node 可以對 remote 進行操作，這裡設定了一個很簡單的 inventory，內容如下：(<font color='blue'>**/etc/ansible/hosts**</font>)

``` ini
vagrant1 ansible_ssh_host=192.168.30.11
vagrant2 ansible_ssh_host=192.168.30.12
vagrant3 ansible_ssh_host=192.168.30.13
```

當然也可以將 inventory 定義在其他檔案中，透過 <font color='red'>**-i**</font> 參數指定檔案即可。

有了以上的 inventory 後，我們可以做幾個測試，確認 control node 是否可以與 remote node 進行溝通：

``` bash
ansible-control-node:~$ ansible vagrant2 -a "ifconfig eth1"
vagrant2 | success | rc=0 >>
eth1      Link encap:Ethernet  HWaddr 08:00:27:62:02:68
          inet addr:192.168.30.12  Bcast:192.168.30.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe62:268/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:604 errors:0 dropped:0 overruns:0 frame:0
          TX packets:190 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:184597 (184.5 KB)  TX bytes:26423 (26.4 KB)

ansible-control-node:~$ ansible vagrant3 -m ping
vagrant3 | success >> {
    "changed": false,
    "ping": "pong"
}

vagrant@ansible-control-node:~$ ansible vagrant1 -a "pwd"
vagrant1 | success | rc=0 >>
/home/vagrant
```

若有出現以上訊息，表示 control node 可以透過 ansible 對 remote node 進行操作。

【註】若要透過以上簡單的 inventory 就要對 remote node 操作，必須先把 control node 的 ssh private key 匯入到每一個 remote node 上，可以參考[此篇文章](http://godleon.github.io/blog/2015/02/22/ansible-introduction/)進行喔!

--------------------------------------------

Behavioral inventory parameters
===============================

上面介紹的 inventory 內容很簡單，只有 alias & ip 而已，若是要指定登入的帳號密碼 or ssh private key 呢? 就必須透過額外的參數進行設定，ansible 會根據這些參數來改變連到 remote node 的行為，以下為參數說明：

| Name | Default | Description |
|------|---------|-------------|
| ansible_ssh_host | localhost | 要連接的 remote node 的 hostname(ip) |
| ansible_ssh_port | 22 | 登入 remote node ssh service 所用的 port number |
| ansible_ssh_user | root | 登入 remote node 所使用的帳號 |
| ansible_ssh_pass | none | 登入 remote node 所使用的密碼 |
| ansible_connection | smart | 若 SSH client 支援 ControlPersist，則使用 **smart**(openssh client)；若不支援 ControlPersist，則改用 **paramiko**(Python-based ssh client) |
| ansible_shell_type | sh | remote node 所用的 shell，預設為 bash(/bin/sh)，也支援像是 csh or powershell .... 等 shell type |
| ansible_python_interpreter | /usr/bin/python | 所有的 ansible module 都是以 python 2 開發而成的，因此這邊指定的是 python 2 的所在位置；如果 remote node 所安裝的 Linux，python 2 的位置不是在 /usr/bin/python(例如：Arch Linux)，那就要另外指定這個參數。(目前 ansible module 與 python 3 無法完全相容) |

【註】以上這些參數也可以設定在 <font color='red'>**/etc/ansible/ansible.cfg**</font> 中的 **[default]** 區段內。

--------------------------------------------

Inventory 撰寫方式
==================

原本的 inventory 內容如下：

``` ini
vagrant1 ansible_ssh_host=192.168.30.11
vagrant2 ansible_ssh_host=192.168.30.12
vagrant3 ansible_ssh_host=192.168.30.13
```

其中 **vagrant1**、**vagrant2**、**vagrant3** 是 alias 的設定，ansible_ssh_host 可以是 ip 也可以是 domain name 設定。

### 最簡單設定

若不需要 alias 設定，可以改成以下最簡單的形式：

``` ini
192.168.30.11
192.168.30.12
192.168.30.13
web.example.com.tw
```

### Group + Alias

也可以透過 Group 的方式將特定的主機進行分類：

``` ini
vagrant1 ansible_ssh_host=192.168.30.11
vagrant2 ansible_ssh_host=192.168.30.12
vagrant3 ansible_ssh_host=192.168.30.13

[vagrant]
vagrant1
vagrant2
vagrant3
```

如此一來就可以透過 <font color='red'>**vagrant**</font> 關鍵字，一次對三台主機進行操控。

### 混合設定

在機器數量有限，卻要達成多種任務的情況下，可以一機多用，當然在 inventory 的設定上也是有相對應的設定彈性：

``` ini
vagrant1 ansible_ssh_host=192.168.30.11
vagrant2 ansible_ssh_host=192.168.30.12
vagrant3 ansible_ssh_host=192.168.30.13

[vagrant]
vagrant1
vagrant2
vagrant3

[production]
prod1.example.com.tw
prod2.example.com.tw

[staging]
stag1.example.com.tw
stag2.example.com.tw

[db]
prod1.example.com.tw
stag1.example.com.tw

[redis]
redis.example.com.tw

[task]
prod1.example.com.tw
prod2.example.com.tw
redis.example.com.tw
vagrant1
```

### 群組中的群組

若群組中還需要再定義額外的群組，可使用下面的設定方式：

``` ini
[django:children]
db
redis
task
```

### 快速定義

若 remote node 名稱有規則性的命名，可以把 inventory 設定如下：

``` ini
# 主機名稱為 web1.example.com.tw, web2.example.com.tw, ......, web20.example.com.tw
[web]
web[1:20].example.com.tw

# 主機名稱為 web01.example.com.tw, web02.example.com.tw, ......, web20.example.com.tw
[web]
web[01:20].example.com.tw

# 主機名稱為 web-a.example.com.tw, web-b.example.com.tw, ......, web-z.example.com.tw
[web]
web-[a:z].example.com.tw
```

--------------------------------------------

變數(vars)的定義方式
====================

### 定義在 inventory 中

在不同環境中，通常都會有不同的變數，例如：在 production/staging/test 環境中，所使用的主機的 ip(hostname)/帳號密碼/db 都會不一樣，因此在設定時所使用的參數都會不相同，這些變動的部份我們都可以定義在 inventory 的變數中(使用 <font color='red'>**[<group_name>:vars]**</font> 的方式宣告)，以下用個範例來說明在佈署不同環境時，變數要如何定義：

``` ini
[all:vars]
ntp_server=tock.stdtime.gov.tw

[production:vars]
db_primary_host=prod1.example.com
db_primary_port=5432
db_replica_host=prod2.example.com
db_name=widget_production
db_user=widgetuser
db_password=mypassword
redis_host=redis.example.com
redis_port=6379

[staging:vars]
db_primary_host=stag1.example.com
db_name=widget_staging
db_user=widgetuser
db_password=anotherpassword
redis_host=redis_stag.example.com
redis_port=6379

[vagrant:vars]
db_primary_host=vagrant3
db_primary_port=5432
db_primary_port=5432
db_name=widget_vagrant
db_user=widgetuser
db_password=lastpassword
redis_host=vagrant3
redis_port=6379
```

### 定義在外部檔案

當 remote host 數量不多時，把變數定義在 inventory 中是 ok 的；但若 remote host 的數量越來越多時，將變數的宣告定義在外部的檔案中會是比較好的方式。

ansible 會自動尋找 playbook 所在的目錄中的 <font color='red'>**host_vars 目錄**</font> & <font color='red'>**roup_vars 目錄**</font> 中所包含的檔案，並使用定義在這兩個目錄中的變數資訊。

舉例來說，inventory / playbook / host_vars / group_vars 可以用類似以下的方式進行配置：如果

- **inventory**：/home/vagrant/ansible/playbooks/inventory

- **playbook**：/home/vagrant/ansible/playbooks/myplaybook

- **host_vars**：/home/vagrant/ansible/playbooks/host_vars/prod1.example.com.tw

- **group_vars**：/home/vagrant/ansible/playbooks/group_vars/production

變數定義的方式有兩種方式：

``` ini
db_primary_host: prod1.example.com.tw
db_replica_host: prod2.example.com.tw
db_name: widget_production
db_user: widgetuser
db_password: lastpassword
redis_host: redis_stag.example.com.tw
```

存取變數方式 => {{ db_primary_host }}


也可以用 YAML 的方式定義：

``` yaml
---
db:
    user: widgetuser
    password: lastpassword
    name: widget_production
    primary:
        host: prod1.example.com.tw
        port: 5432
    replica:
        host: prod2.example.com.tw
        port: 5432
redis:
    host: redis_stag.example.com.tw
    port: 6379
```

存取變數方式 => {{ db_primary_host_yaml }}

甚至可以在繼續細分，定義檔案 **/home/vagrant/ansible/playbooks/group_vars/production/db**：

``` yaml
---
db:
    user: widgetuser
    password: lastpassword
    name: widget_production
    primary:
        host: prod1.example.com.tw
        port: 5432
    replica:
        host: prod2.example.com.tw
        port: 5432
```

--------------------------------------------

Dynamic Inventory
=================

之前介紹的 inventory 的內容，都是已經預先定義好的靜態檔案，而 ansible 除了靜態的 inventory 之外，也可以支援動態的 inventory。

動態的 inventory 是怎麼達成的呢? **答案是透過執行 script(一般會是可執行的 python script)，取得包含 inventory 資訊的 json 資訊來使用**

以下是透過 script 產生出來的 json inventory 資訊範例：

``` json
{
    "production": ["delaware.example.com", "georgia.example.com",
        "maryland.example.com", "newhampshire.example.com",
        "newjersey.example.com", "newyork.example.com",
        "northcarolina.example.com", "pennsylvania.example.com",
        "rhodeisland.example.com", "virginia.example.com"
    ],
    "staging": ["ontario.example.com", "quebec.example.com"],
    "vagrant": ["vagrant1", "vagrant2", "vagrant3"],
    "lb": ["delaware.example.com"],
    "web": ["georgia.example.com", "newhampshire.example.com",
        "newjersey.example.com", "ontario.example.com", "vagrant1"
    ]
    "task": ["newyork.example.com", "northcarolina.example.com",
        "ontario.example.com", "vagrant2"
    ],
    "redis": ["pennsylvania.example.com", "quebec.example.com", "vagrant3"],
    "db": ["rhodeisland.example.com", "virginia.example.com", "vagrant3"]
}
```

這份資料是一個 json object，包含了 group 的定義(production、staging ... 等等)以及 group 中多個 host 的定義。

目前 ansible 官方有提供許多可以取得 public cloud(AWS、Azure、OpenStack ... 等等) remote hosts 資訊的 script，若有這樣的需求，可以到 [ansible 的 github repository](https://github.com/ansible/ansible/tree/devel/plugins/inventory) 找找(當然也可以自己開發)。

使用方式如下：

1. 加上執行(x)的權限給 script

2. 將 script 與 inventory file 放在同一目錄

如此一來 ansible 就會自動讀取 inventory file 取得靜態的 inventory 資訊，並執行 script 取得動態的 inventory 資訊，將兩者 merge 後並使用。

--------------------------------------------

參考資料
========

- [Ansible Documentation — Ansible Documentation](http://docs.ansible.com/index.html)

- [Ansible: Up and Running - O'Reilly Media](http://shop.oreilly.com/product/0636920035626.do)