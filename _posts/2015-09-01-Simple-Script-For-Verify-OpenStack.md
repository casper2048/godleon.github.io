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

``` bash
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
``` bash
$ source admin-openrc.sh
```

------------------------------------------------

各服務驗證方式
============

### Cinder
建立一個 1GB 的 volume
``` bash
$ cinder create 1 --display-name MyFirstVolume
$ cinder list
```

### Glance
下載 & 上傳 cirros image 至 glance service
``` bash
$ wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
$ glance image-create --name "cirros-0.3.4-x86_64-qcow2" --disk-format qcow2 --container-format bare --is-public True --file cirros-0.3.4-x86_64-disk.img --progress
$ glance image-list
```

### Ceph
驗證 cinder & glance 是否有使用 ceph 作為 storage backend
``` bash
$ sudo ceph -s
$ sudo ceph osd lspools
$ sudo ceph -p cinder-ceph list
$ sudo ceph -p glance list
```

### Neutron
#### 1、建立對外網路
``` bash
# 建立連外網路 ext_net
$ neutron net-create ext_net --router:external --provider:physical_network external --provider:network_type flat

# 設定連外網路所使用的 ip 範圍(請根據自己的環境調整參數設定)
$ neutron subnet-create ext_net --name ext_subnet --allocation-pool start=10.10.198.101,end=10.10.198.200 --disable-dhcp --gateway 10.10.198.254 10.10.198.0/24
```

#### 2、建立內部網路(by tenant)
``` bash
# 建立 tenant network
$ neutron net-create demo-net

# 在 tenant network 的基礎上建立 subnet (IP range 可自行定義)
$ neutron subnet-create demo-net --name demo-subnet --gateway 192.168.50.1 192.168.50.0/24
```

#### 3、建立 virtual router 並將上述建立的網路附加上來
router 是用來連接對外網路 & 內部網路之用，概念示意如下：
`demo-subnet <====> demo-router <====> ext_net <====> Internet`
``` bash
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

------------------------------------------------

建立 instance
============
``` bash
# 建立金鑰(假設檔案位置在 ~/.ssh/id_rsa.pub)
$ ssh-keygen

# 將金鑰匯入 nova 中
$ nova keypair-add --pub-key ~/.ssh/id_rsa.pub demo-key
$ nova keypair-list

# 查詢啟用 instance 所需要的資訊
$ nova flavor-list
$ nova image-list
$ neutron net-list
$ nova secgroup-list
# 啟用 instance (下面那一堆代碼是 demo-net 的 ID)
$ nova boot --flavor m1.tiny --image cirros-0.3.4-x86_64-qcow2 --nic net-id=fa589142-ac15-46d4-8c1c-14681064198e --security-group default --key-name demo-key demo-instance1

# 查詢 instance 啟用狀況
$ nova list

# 查詢 vnc console
$ nova get-vnc-console demo-instance1 novnc
```

------------------------------------------------

參考資料
=======

- [OpenStack Docs: OpenStack command-line interface cheat sheet](http://docs.openstack.org/user-guide/cli_cheat_sheet.html)

- [Ceph cheatsheet - ShainMiley.com](http://www.shainmiley.com/wordpress/2014/08/23/ceph-cheatsheet/)
