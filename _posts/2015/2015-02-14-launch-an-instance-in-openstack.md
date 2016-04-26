---
layout: post
title:  "[OpenStack] 啟動第一個 VM"
description: "安裝完 Keystone/Glance/Nova/Neutron 後，測試啟動第一個 VM"
date:   2015-02-14 07:50:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Ubuntu, Linux]
---

要啟動第一個 VM，有兩個部份的工作要完成：

1. 產生 key pair

2. 產生 VM 並啟動


產生 key pair
==============

1、切換成使用者 demo

``` bash
root@controller:~# source ~/openstack/demo-openrc.sh
```

2、建立存放 public/private key pair 的目錄：

``` bash
root@controller:~# mkdir -p ~/openstack/keypair/demo
root@controller:~# mkdir -p ~/openstack/keypair/admin
```

3、使用 SSL 工具產生 key pair：

``` bash
root@controller:~# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): /home/godleon/openstack/keypair/demo/id_rsa	# 指定存放 key pair 的位置
Enter passphrase (empty for no passphrase):	# 輸入密碼
Enter same passphrase again:	# 密碼確認
Your identification has been saved in /home/godleon/openstack/keypair/demo/id_rsa.
Your public key has been saved in /home/godleon/openstack/keypair/demo/id_rsa.pub.
The key fingerprint is:
47:db:04:7c:57:74:49:2b:2d:0f:55:86:ef:7a:9f:69 root@controller
The key's randomart image is:
+--[ RSA 2048]----+
|         ..   .*O|
|          ... =oo|
|          ...= + |
|         . +  = .|
|        S o .  o |
|         .      .|
|               . |
|              .E+|
|              .+o|
+-----------------+
```

4、將 public key 加入到 openstack 環境中：

``` bash
root@controller:~# nova keypair-add --pub-key /home/godleon/openstack/keypair/demo/id_rsa.pub demo-key
```

5、驗證 public key 是否已經正確加入

``` bash
root@controller:~# nova keypair-list
+----------+-------------------------------------------------+
| Name     | Fingerprint                                     |
+----------+-------------------------------------------------+
| demo-key | 47:db:04:7c:57:74:49:2b:2d:0f:55:86:ef:7a:9f:69 |
+----------+-------------------------------------------------+
```

-----------------------------

產生 VM 並啟動
==============

1、查詢目前可以使用的 VM 規格

``` bash
root@controller:~# nova flavor-list
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
```

2、查詢目前 Glance 提供的映像檔清單

``` bash
root@controller:~# nova image-list
+--------------------------------------+---------------------+--------+--------+
| ID                                   | Name                | Status | Server |
+--------------------------------------+---------------------+--------+--------+
| e37ba2d2-b9c1-42fb-8e81-5336efb0c164 | cirros-0.3.3-x86_64 | ACTIVE |        |
+--------------------------------------+---------------------+--------+--------+
```

3、查詢目前可用的網路清單

``` bash
root@controller:~# neutron net-list
+--------------------------------------+----------+------------------------------------------------------+
| id                                   | name     | subnets                                              |
+--------------------------------------+----------+------------------------------------------------------+
| db701eee-164f-4126-ba56-0b548e4838e8 | ext-net  | 555e6dc6-0a67-49df-b848-e71347cec14d                 |
| 3ed88154-cab0-442a-ad61-64c344e31b1d | demo-net | 8df0b029-b92d-43ca-b70b-d4cc6ecd3357 192.168.50.0/24 |
+--------------------------------------+----------+------------------------------------------------------+
```

4、查詢目前可用的 security group

``` bash
root@controller:~# nova secgroup-list
+--------------------------------------+---------+-------------+
| Id                                   | Name    | Description |
+--------------------------------------+---------+-------------+
| d2e1867c-0cf3-4814-a073-56ba3cf6c4bb | default | default     |
+--------------------------------------+---------+-------------+
```

5、建置並啟動 VM

``` bash
root@controller:~# nova boot --flavor m1.tiny --image cirros-0.3.3-x86_64 --nic net-id=3ed88154-cab0-442a-ad61-64c344e31b1d --security-group default --key-name demo-key demo-instance1
+--------------------------------------+------------------------------------------------------------+
| Property                             | Value                                                      |
+--------------------------------------+------------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                     |
| OS-EXT-AZ:availability_zone          | nova                                                       |
| OS-EXT-STS:power_state               | 0                                                          |
| OS-EXT-STS:task_state                | scheduling                                                 |
| OS-EXT-STS:vm_state                  | building                                                   |
| OS-SRV-USG:launched_at               | -                                                          |
| OS-SRV-USG:terminated_at             | -                                                          |
| accessIPv4                           |                                                            |
| accessIPv6                           |                                                            |
| adminPass                            | qZVYVtaA9Utj                                               |
| config_drive                         |                                                            |
| created                              | 2015-02-13T10:07:10Z                                       |
| flavor                               | m1.tiny (1)                                                |
| hostId                               |                                                            |
| id                                   | 9dce2984-7c49-43db-a733-f64cbb125992                       |
| image                                | cirros-0.3.3-x86_64 (e37ba2d2-b9c1-42fb-8e81-5336efb0c164) |
| key_name                             | demo-key                                                   |
| metadata                             | {}                                                         |
| name                                 | demo-instance1                                             |
| os-extended-volumes:volumes_attached | []                                                         |
| progress                             | 0                                                          |
| security_groups                      | default                                                    |
| status                               | BUILD                                                      |
| tenant_id                            | 824e59f8952849648d19b99275e3a913                           |
| updated                              | 2015-02-13T10:07:10Z                                       |
| user_id                              | 31cce302b6f449a8b81615aa6c2bd309                           |
+--------------------------------------+------------------------------------------------------------+
```

6、查詢目前 VM 狀態

``` bash
root@controller:~# nova list
+--------------------------------------+----------------+--------+------------+-------------+-----------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks              |
+--------------------------------------+----------------+--------+------------+-------------+-----------------------+
| 9dce2984-7c49-43db-a733-f64cbb125992 | demo-instance1 | ACTIVE | -          | Running     | demo-net=192.168.50.2 |
+--------------------------------------+----------------+--------+------------+-------------+-----------------------+
```

可以看到 VM 已經正確啟動，並有 IP 的資訊在上面。

------------------------------

存取 VM VNC console
===================

由於外部的網路跟 openstack VM 的內部網路預設是不通的，要測試 VM 的內部通訊是否正常，必須要通過 vnc console，可以透過以下指令取得 VNC console URL 資訊：

``` bash
root@controller:~# nova get-vnc-console demo-instance1 novnc
+-------+---------------------------------------------------------------------------------+
| Type  | Url                                                                             |
+-------+---------------------------------------------------------------------------------+
| novnc | http://controller:6080/vnc_auto.html?token=53a85f8e-073e-4236-8597-ddef994824b6 |
+-------+---------------------------------------------------------------------------------+
```

透過[上述網址](http://10.0.0.11:6080/vnc_auto.html?token=53a85f8e-073e-4236-8597-ddef994824b6)登入到 VNC console 的介面

登入後透過 ifconfig 查詢配發到的 IP，並使用 ping 測試之前設定的 Gateway(192.168.50.1)

![檢查 instance 狀態](https://lh4.googleusercontent.com/-XjQ6uWORqFE/VN4DSKdI0BI/AAAAAAAAJ9I/d8dt1RIGIAY/w679-h310-no/check_instance.png)

看到上面圖中的內容，就可以確定目前 VM 是在正常運作中喔!

------------------------------

參考資料
========

- [OpenStack Installation Guide for Ubuntu 14.04 - juno][1]

[1]:  http://docs.openstack.org/juno/install-guide/install/apt/content/