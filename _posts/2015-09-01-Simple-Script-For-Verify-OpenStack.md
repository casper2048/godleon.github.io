---
layout: post
title:  "[OpenStack] 簡單驗證 OpenStack 環境的 script"
description: "此篇文章中提供了可以用來驗證 OpenStack 環境中的每個 service 是否有正常提供服務的 script，可用來提供簡單快速的驗"
date: 2015-09-01 15:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack]
---

切換身份用的 script
=================

可用以下的 script(**admin-openrc.sh**) 切換成 admin 的身份，但必須提供 admin password

```bash
export OS_VOLUME_API_VERSION=2
export OS_AUTH_URL=http://[YOUR_KEYSTONE_ENDPOINT_IP]:5000/v2.0
export OS_TENANT_NAME="admin"
export OS_USERNAME="admin"

echo "Please enter your OpenStack Password: "
read -sr OS_PASSWORD_INPUT
export OS_PASSWORD=$OS_PASSWORD_INPUT

export OS_REGION_NAME="RegionOne"
if [ -z "$OS_REGION_NAME" ]; then unset OS_REGION_NAME; fi
```

使用方法如下：
```bash
$ source admin-openrc.sh
```

------------------------------------------------

各服務驗證方式
============

### Cinder
建立一個 1GB 的 volume

```bash
$ cinder create 5 --display-name MyFirstVolume
$ cinder list
```

### Glance
下載 & 上傳 cirros image 至 glance service

```bash
# 上傳 cirros cloud image
$ glance image-create --name "cirros-0.3.4-x86_64-qcow2" --disk-format qcow2 --container-format bare --is-public True --copy-from http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

# 上傳 ubuntu cloud image
$ glance image-create --name "ubuntu-trusty-server-amd64-qcow2" --disk-format qcow2 --container-format bare --is-public True --copy-from https://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img

# 上傳 fedora cloud image
$ glance image-create --name "Fedora-22-x86_64-qcow2" --disk-format qcow2 --container-format bare --is-public True --copy-from https://download.fedoraproject.org/pub/fedora/linux/releases/22/Cloud/x86_64/Images/Fedora-Cloud-Base-22-20150521.x86_64.qcow2

$ glance image-list
```

### Ceph
驗證 cinder & glance 是否有使用 ceph 作為 storage backend

```bash
$ sudo ceph -s
$ sudo ceph osd lspools
$ sudo rbd -p cinder-ceph list
$ sudo rbd -p glance list
```

### Neutron
#### 1、建立對外網路
```bash
# 建立連外網路 ext_net
$ neutron net-create ext_net --router:external --provider:physical_network external --provider:network_type flat

# 設定連外網路所使用的 ip 範圍(請根據自己的環境調整參數設定)
$ neutron subnet-create ext_net --name ext_subnet --allocation-pool start=10.10.198.101,end=10.10.198.200 --disable-dhcp --gateway 10.10.198.254 10.10.198.0/24 --dns-nameservers list=true 8.8.8.8 8.8.4.4
```

#### 2、建立內部網路(by tenant)
```bash
# 建立 tenant network
$ neutron net-create demo-net

# 在 tenant network 的基礎上建立 subnet (IP range 可自行定義)
$ neutron subnet-create demo-net --name demo-subnet --gateway 192.168.50.1 192.168.50.0/24 --dns-nameservers list=true 8.8.8.8 8.8.4.4
```

#### 3、建立 virtual router 並將上述建立的網路附加上來

router 是用來連接對外網路 & 內部網路之用，概念示意如下：

> demo-subnet <====> demo-router <====> ext_net <====> Internet

```bash
# 建立 router
$ neutron router-create demo-router

# 附加 tenant subnet(demo-subnet) 到 router 上
$ neutron router-interface-add demo-router demo-subnet

# 附加對外網路(ext_net)到 router 上
$ neutron router-gateway-set demo-router ext_net
```

#### 4、驗證 router 可否對外
通常對外網路附加到 router 上後，router 會取得到指定對外 IP 範圍中的第一個 IP，以上面的例子來說，應該會取得 10.10.198.101，
因此可以透過 ping 來測試是否有設定成功。

#### 5、產生 floating IPs

可以多下幾次指令取得多個 floating ip

``` bash
$ neutron floatingip-create ext_net
```

#### 6、檢視 floating ip list

``` bash
$ neutron floatingip-list
```

------------------------------------------------

建立 instance
=============

### 1、建立金鑰 (ssh keypair)

透過 ssh keypair，就可以從外部透過 ssh key 登入 instance：(instance 必須連結 floating ip)

```bash
# 建立金鑰(假設檔案位置在 ~/.ssh/id_rsa.pub)
$ ssh-keygen

# 將金鑰匯入 nova 中
$ nova keypair-add --pub-key ~/.ssh/id_rsa.pub demo-key
$ nova keypair-list
```

### 2、啟用 instance

```bash
# 查詢啟用 instance 所需要的資訊
$ nova flavor-list
$ nova image-list
$ neutron net-list
$ nova secgroup-list

# 啟用 instance (cirros)
$ nova boot --flavor m1.tiny --image cirros-0.3.4-x86_64-qcow2 --nic net-id=$(neutron net-list | grep 'demo-net' | awk '{print $2}') --security-group default --key-name demo-key cirros-instance

# 啟用 instance (ubuntu)
$ nova boot --flavor m1.medium --image ubuntu-trusty-server-amd64-qcow2 --nic net-id=$(neutron net-list | grep 'demo-net' | awk '{print $2}') --security-group default --key-name demo-key ubuntu-instance

# 啟用 instance (fedora)
$ nova boot --flavor m1.medium --image Fedora-22-x86_64-qcow2 --nic net-id=$(neutron net-list | grep 'demo-net' | awk '{print $2}') --security-group default --key-name demo-key fedora-instance

# 查詢 instance 啟用狀況
$ nova list
```

### 3、查詢 instance console URL

```bash
# 查詢 vnc console
$ nova get-vnc-console cirros-instance novnc
$ nova get-vnc-console ubuntu-instance novnc
$ nova get-vnc-console fedora-instance novnc
```

上述步驟執行完後，可以從 Horizon 進入 VM console，也可以透過上面的指令取得的 vnc console url 進入 instance console 畫面。

------------------------------------------------

Security
========

### 1、修改 security group(default) 規則

``` bash
# 開啟 icmp(ping) 功能
$ nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

# 允許外部透過 ssh 存取 instance
$ nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
```

### 2、顯示目前可用的 floating ip

``` bash
$ neutron floatingip-list
```

### 3、連結 flaoting ip 至 instance

```bash
# 連結 floating ip 至 cirros-instance
$ nova floating-ip-associate cirros-instance 10.10.198.102

# 連結 floating ip 至 ubuntu-instance
$ nova floating-ip-associate ubuntu-instance 10.10.198.103

# 連結 floating ip 至 fedora-instance
$ nova floating-ip-associate fedora-instance 10.10.198.104
```


### 3、從外部連線至 instance

``` bash
# cirros cloud image 預設使用者名稱為 cirros
$ ssh -i ~/.ssh/id_rsa cirros@10.10.198.102

# ubuntu cloud image 預設使用者名稱為 ubuntu
$ ssh -i ~/.ssh/id_rsa ubuntu@10.10.198.103

# fedora cloud image 預設使用者名稱為 fedora
$ ssh -i ~/.ssh/id_rsa fedora@10.10.198.104
```

如此一來就可以不用輸入密碼就可以登入囉!

<font color='red'>【註】</font>上面的 cloud image 中只有 cirros 可以用密碼登入，其他的 cloud image 都必須透過 ssh keypair 來進行登入。

------------------------------------------------

參考資料
=======

- [Launch an instance with OpenStack Networking (neutron) - OpenStack Installation Guide for Ubuntu 14.04](http://docs.openstack.org/kilo/install-guide/install/apt/content/launch-instance-neutron.html)

- [OpenStack Docs: OpenStack command-line interface cheat sheet](http://docs.openstack.org/user-guide/cli_cheat_sheet.html)

- [Ceph cheatsheet - ShainMiley.com](http://www.shainmiley.com/wordpress/2014/08/23/ceph-cheatsheet/)

- [Support for ISO images - OpenStack Configuration Reference  - kilo](http://docs.openstack.org/kilo/config-reference/content/iso-support.html)

- [OpenStack Docs: Launch an instance using ISO image](http://docs.openstack.org/user-guide/cli_nova_launch_instance_using_ISO_image.html)

- [OpenStack Docs: Launch an instance from a volume](http://docs.openstack.org/user-guide/cli_nova_launch_instance_from_volume.html)

- [OpenStack Nova: Boot From Volume - Chen Xiao, OpenStack博客 - 博客频道 - CSDN.NET](http://blog.csdn.net/juvxiao/article/details/22614663)

- [Where to Find OpenStack Cloud Images](http://thornelabs.net/2014/06/01/where-to-find-openstack-cloud-images.html)

- [BlockDeviceConfig - OpenStack](https://wiki.openstack.org/wiki/BlockDeviceConfig)

- [Rackspace Developer Center - Neutron Networking: Simple Flat Network](https://developer.rackspace.com/blog/neutron-networking-simple-flat-network/)
