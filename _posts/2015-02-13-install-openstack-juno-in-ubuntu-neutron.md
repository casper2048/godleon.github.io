---
layout: post
title:  "[OpenStack] 安裝 Juno @ Ubuntu 14.04 (5) - 設定 Network Service(Neutron)"
description: "在 Ubuntu 14.04 中，安裝 OpenStack(Juno) Neutron 服務，提供與其他網通廠商的軟體 or 設備整合的功能"
date:   2015-02-13 16:30:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Ubuntu, Linux]
---

OpenStack 網路功能
====================

OpenStack 的網路功能允許管理者可以針對其他服務所需要的網路功能，建立並附加相關的虛擬介面裝置；此外 OpenStack 在整體架構上提供了很大的彈性，透過 plug-in 的方式，可讓第三方廠商的網路設備 or 軟體與 OpenStack 的網路功能整合。

在 OpenStack 網路功能中，包含了三個重要的部分：

- neutron-server

接收來自與網路功能的 request 後，將其 routing 到真正提供服務的 plug-in 軟體 or 設備上去。

- OpenStack Networking plug-ins and agents

實際上，OpenStack 的網路管理功能是透過 plug-in & agent 的方式所提供的，除了可以直接使用的 Open vSwitch 之外，還有許多其他網通廠提供的設備可整合，例如：Cisco virtual and physical switches, NEC OpenFlow 產品, Linux bridging, Ryu Network Operating System, 以及 VMware NSX ... 等等。 

- Messaging queue

這服務是提供 Neutron server 與其他多個 agent 作訊息交換之用。

------------------------------

Network Service (Neutron) 概觀
==============================

Network Service (**Neutron**) 負責虛擬網路架構與外部實體網路架構(包含各廠商的設備)之間的整合 & 存取，用以提供 tenant 可自行建立像是防火牆、負載平衡、VPN ... 等網路環境。

網路設定的概念其實跟我們之前所學的並沒有差太多，包含 DHCP、VLAN、Routing ... 等等。

比較值得一提的是 Networking Service 支援了 **security group** 的概念：

- 管理者可以針對每個 security group 進行防火牆規則的設定
- 每個 VM 可以屬於一個或多個 security group

有了以上的機制，讓 Firewall-as-a-Service & Load-Balancing-as-a-Service 變的很容易實現。

------------------------------

安裝 & 設定 @ Controller Node
=============================

此部分的安裝設定位於 **Controller Node** 上，別裝錯了!

### 安裝前的準備工作 @ Controller Node

1、安裝 Neutron 用的資料庫

``` bash
# 登入資料庫
root@controller:~# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 150
Server version: 5.5.41-MariaDB-1ubuntu0.14.04.1 (Ubuntu)

Copyright (c) 2000, 2014, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

# 建立 neutron 資料庫
MariaDB [(none)]> CREATE DATABASE neutron;
Query OK, 1 row affected (0.00 sec)

# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
Query OK, 0 rows affected (0.00 sec)

# 設定權限
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
Query OK, 0 rows affected (0.00 sec)

# 將權限設定寫入資料庫
MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit
Bye
```

2、切換為使用者 admin

``` bash
root@controller:~# source ~/openstack/admin-openrc.sh
```

3、建立使用者 neutron

``` bash
root@controller:~# keystone user-create --name neutron --pass NEUTRON_PASS
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |                                  |
| enabled  |               True               |
|    id    | ce83eb6c9d5f4e06aafdfe0635ddae59 |
|   name   |             neutron              |
| username |             neutron              |
+----------+----------------------------------+
```

4、將 user(neutron)指定屬於 tenant(service)，並設定 role(admin) 權限

``` bash
root@controller:~# keystone user-role-add --user neutron --tenant service --role admin
```

5、在 Keystone 註冊 Neutron service

``` bash
root@controller:~# keystone service-create --name neutron --type network --description "OpenStack Networking"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |       OpenStack Networking       |
|   enabled   |               True               |
|      id     | c5c11e08e5d146b4b0f8d918d922e5bd |
|     name    |             neutron              |
|     type    |             network              |
+-------------+----------------------------------+
```

6、在 Keystone 註冊 Neutron API endpoints

``` bash
root@controller:~# keystone endpoint-create --service-id $(keystone service-list | awk '/ network / {print $2}') --publicurl http://controller:9696 --adminurl http://controller:9696 --internalurl http://controller:9696 --region regionOne
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |      http://controller:9696      |
|      id     | 19995a19fc6b4669972f20089ac46f99 |
| internalurl |      http://controller:9696      |
|  publicurl  |      http://controller:9696      |
|    region   |            regionOne             |
|  service_id | c5c11e08e5d146b4b0f8d918d922e5bd |
+-------------+----------------------------------+
```

### 安裝網路相關元件 @ Controller Node

``` bash
root@controller:~# apt-get -y install neutron-server neutron-plugin-ml2 python-neutronclient
```

### 元件設定 @ Controller Node

其實 controller 是用來控制 network node 的，所以很多管理工作會在 controller node 完成，以下要完成的設定包含資料庫、認證機制、message broker、topology 改變通知、plug-in ... 等等。

編輯 `/etc/neutron/neutron.conf`，並將設定修改如下：

``` bash
[DEFAULT]
rpc_backend = rabbit	# message broker 設定
rabbit_host = controller
rabbit_password = RABBIT_PASS
auth_strategy = keystone	# 認證 & 授權設定
core_plugin = ml2	# network plug-in 相關設定
service_plugins = router
allow_overlapping_ips = True

notify_nova_on_port_status_changes = True	# topology 變更通知設定
notify_nova_on_port_data_changes = True
nova_url = http://controller:8774/v2
nova_admin_auth_url = http://controller:35357/v2.0
nova_region_name = regionOne
nova_admin_username = nova
nova_admin_tenant_id = SERVICE_TENANT_ID  #要替換成下方查詢到的 tenant id
nova_admin_password = NOVA_PASS

[database]
connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron

# 認證 & 授權設定
[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = neutron
admin_password = NEUTRON_PASS
#auth_host = controller
#auth_port = 35357
#auth_protocol = http
```

查詢 `SERVICE_TENANT_ID`：

``` bash
root@controller:~# keystone tenant-get service
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |          Service Tenant          |
|   enabled   |               True               |
|      id     | 8cc687f0f03a49e0ae9f7c45a0c6c62e |
|     name    |             service              |
+-------------+----------------------------------+
```

將查詢到的 tenant id 取代設定檔中的 `SERVICE_TENANT_ID`。

### 設定 Modular Layer 2 (ML2) plug-in @ Controller Node

我們在上面的設定中使用 ML2 plug-in，而此 plug-in 使用的是 [Open vSwitch(OVS)](http://openvswitch.org/) 作為建置虛擬網路架構之用。

但在 controller node 上並不需要將 OVS 裝好，僅需作相關控制設定即可，因為實際上的網路流量是由 network node 處理。

編輯 `/etc/neutron/plugins/ml2/ml2_conf.ini`，並修改設定如下：

``` bash
[ml2]
type_drivers = flat,gre
tenant_network_types = gre
mechanism_drivers = openvswitch

# 設定 tunnel identifier (id) range
[ml2_type_gre]
tunnel_id_ranges = 1:1000

# 啟用 安全設定，並使用 iptables 作為 firewall driver
[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
``` 

### 調整 Computer Service 改用 Neutron 服務 @ Controller Node

預設 Compute Service(Nova) 會使用 nova-network，所以要改為使用 Neutron

編輯 `/etc/nova/nova.conf`，加入以下設定：

``` bash
[DEFAULT]
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver

# Neutron 相關設定
[neutron]
url = http://controller:9696
auth_strategy = keystone
admin_auth_url = http://controller:35357/v2.0
admin_tenant_name = service
admin_username = neutron
admin_password = NEUTRON_PASS
```

### 佈署資料庫並重啟服務 @ Controller Node

1、佈署資料庫

``` bash
root@controller:~# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade juno" neutron
```

2、重新啟動 Compute Service(Nova) 相關服務

``` bash
root@controller:~# service nova-api restart
root@controller:~# service nova-scheduler restart
root@controller:~# service nova-conductor restart
```

3、重新啟動 Networing Service(Neutron) 相關服務

``` bash
root@controller:~# service neutron-server restart
```

### 驗證安裝是否完成 @ Controller Node

1、切換成使用者 admin

``` bash
root@controller:~# source ~/openstack/admin-openrc.sh
```

2、查詢 Neutron service 所使用的 extension 清單

``` bash
root@controller:~# neutron ext-list
+-----------------------+-----------------------------------------------+
| alias                 | name                                          |
+-----------------------+-----------------------------------------------+
| security-group        | security-group                                |
| l3_agent_scheduler    | L3 Agent Scheduler                            |
| ext-gw-mode           | Neutron L3 Configurable external gateway mode |
| binding               | Port Binding                                  |
| provider              | Provider Network                              |
| agent                 | agent                                         |
| quotas                | Quota management support                      |
| dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
| l3-ha                 | HA Router extension                           |
| multi-provider        | Multi Provider Network                        |
| external-net          | Neutron external network                      |
| router                | Neutron L3 Router                             |
| allowed-address-pairs | Allowed Address Pairs                         |
| extraroute            | Neutron Extra Route                           |
| extra_dhcp_opt        | Neutron Extra DHCP opts                       |
| dvr                   | Distributed Virtual Router                    |
+-----------------------+-----------------------------------------------+
```

若是有看到上述的資料顯示，表示在 **Controller Node** 上的設定完成囉!

------------------------------

安裝 & 設定 @ Network Node
=============================

Network Node 主要的功能在於負責 routing 內部與外部之間的網路流量，並提供 DHCP 服務。

此部分的安裝設定位於 **Network Node** 上，別裝錯了!

### 安裝前的準備工作 @ Network Node

首先先調整 Linux kernel 中的網路相關設定

1、編輯 `/etc/sysctl.conf`，修改設定如下：

``` bash
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```

2、套用設定

``` bash
root@network:~# sysctl -p
```

### 安裝 Network 相關套件 @ Network Node

``` bash
root@network:~# apt-get -y install neutron-plugin-ml2 neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent
```

### Network Service 共同元件設定 @ Network Node

這裡進行 Neutron 共同元件的設定，包含了認證機制、message broker、plug-in ... 等等。

編輯 `/etc/neutron/neutron.conf`，修改成以下設定：

``` bash
# 註解掉所有 database 連線設定，Network 只負責網路流量的處理，跟資料庫無關
[database]

[DEFAULT]
rpc_backend = rabbit	# message broker 設定
rabbit_host = controller
rabbit_password = RABBIT_PASS
auth_strategy = keystone	# 認證 & 授權設定
core_plugin = ml2	# plug-in 相關設定
service_plugins = router
allow_overlapping_ips = True

# 認證 & 授權設定
[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = neutron
admin_password = NEUTRON_PASS
#auth_host = 127.0.0.1
#auth_port = 35357
#auth_protocol = http
```

### 設定 Modular Layer 2 (ML2) plug-in @ Network Node

ML2 plug-in 使用 Open vSwitch(OVS) 作為 agent 來提供虛擬網路架構給 VM 使用。

編輯 `/etc/neutron/plugins/ml2/ml2_conf.ini`，並修改設定如下：

``` bash
# driver, network type 相關設定
[ml2]
type_drivers = flat,gre
tenant_network_types = gre
mechanism_drivers = openvswitch

[ml2_type_flat]
flat_networks = external

[ml2_type_gre]
tunnel_id_ranges = 1:1000

# 安全性相關設定
[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

[ovs]
local_ip = 10.0.1.21	# 設定 tunnel interface ip @ network node
enable_tunneling = True
bridge_mappings = external:br-ex

# 啟用 GRE tunnel
[agent]
tunnel_types = gre
```

### 設定 L3 agent @ Network Node

L3 agent 提供虛擬網路中 routing 的服務。

編輯 `/etc/neutron/l3_agent.ini`，並修改設定如下：

``` bash
[DEFAULT]
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver		# driver 相關設定
use_namespaces = True
external_network_bridge = br-ex
router_delete_namespaces = True
```

### 設定 DGCP agent @ Network Node

提供虛擬網路中的 DHCP 服務。

編輯 `/etc/neutron/dhcp_agent.ini`，並修改設定如下：

``` bash
verbose = True

interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver		# driver 相關設定
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
use_namespaces = True
dhcp_delete_namespaces = True
```

### 設定 metadata agent @ Network/Controller Node

metadata agent 的功能是用來提供設定相關資訊(例如：憑證)給 VM。

1、編輯 `/etc/neutron/metadata_agent.ini`，修改設定如下：

``` bash
[DEFAULT]
auth_url = http://controller:5000/v2.0	# 認證授權相關設定
auth_region = regionOne
admin_tenant_name = service
admin_user = neutron
admin_password = NEUTRON_PASS
nova_metadata_ip = controller	# 設定提供 nova metadata 資訊的 host 位址
metadata_proxy_shared_secret = METADATA_SECRET	# 設定 metadata proxy 密碼
```

2、回到 **controller node**，編輯 `/etc/nova/nova.conf`，修改設定如下：

``` bash
[neutron]
service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```

3、重新啟動 controller node 上的 compute api 服務

``` bash
root@controller:~# service nova-api restart
```

### 設定 Open vSwitch (OVS) service @ Network Node

OVS service 提供虛擬架構中的網路功能，其中 **br-int** 處理對內的網路流量，**br-ex** 則負責處理對外的網路流量，其中 br-ex 必須附加在對外的實體網路卡上以提供對網際網路的存取。

1、重新啟動 OVS 服務

``` bash
root@network:~# service openvswitch-switch restart
```

2、增加 br-ex interface

``` bash
root@network:~# ovs-vsctl add-br br-ex
```

3、將 br-ex 附加到實體對外的網路卡介面(範例中對外網路介面為 **eth0**)

``` bash
root@network:~# ovs-vsctl add-port br-ex eth0
```

### 重新啟動網路相關服務 @ Network Node

``` bash
root@network:~# service neutron-plugin-openvswitch-agent restart
root@network:~# service neutron-l3-agent restart
root@network:~# service neutron-dhcp-agent restart
root@network:~# service neutron-metadata-agent restart
```

### 驗證是否安裝完成 @ Controller Node

驗證動作需要在 **Controller Node** 上執行。

首先切換為使用者 admin：

``` bash
root@controller:~# source ~/openstack/admin-openrc.sh
```

查詢目前 Neutron 上所安裝好的 agent 清單：

``` bash
root@controller:~# neutron agent-list
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
| id                                   | agent_type         | host    | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
| 1bb499bc-dcc4-4d5d-8ae0-e64fc1b8c43b | DHCP agent         | network | :-)   | True           | neutron-dhcp-agent        |
| 22fc052c-51cb-4c9d-9ed5-76a6b2ae1987 | Metadata agent     | network | :-)   | True           | neutron-metadata-agent    |
| 41beba50-3076-4fae-ab29-53ab748de28f | Open vSwitch agent | network | :-)   | True           | neutron-openvswitch-agent |
| bf189bbf-5f73-46d3-b9b9-57d8b9c6720f | L3 agent           | network | :-)   | True           | neutron-l3-agent          |
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
```

如果出現以上訊息，就表示在 Network Node 上的設定正確完成囉!

------------------------------

安裝 & 設定 @ Compute Node
=============================

Compute 在網路的功能部分，主要是負責 VM 的網路連線 & security group 的相關安全管理功能。

### 安裝前的準備工作 @ Compute Node

由於在 Compute Node 上面的 VM 是要使用網路的，因此必須先把 kernel 的網路相關參數設定好才能正常運作。

1、編輯 `/etc/sysctl.conf`，並修改設定如下：

``` bash
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```

2、套用設定

``` bash
root@compute1:~# sysctl -p
```

### 安裝網路相關套件 @ Compute Node

``` bash
root@compute1:~# apt-get -y install neutron-plugin-ml2 neutron-plugin-openvswitch-agent
```

### Network Service 共同元件設定 @ Compute Node

這裡進行 Neutron 共同元件的設定，包含了認證機制、message broker、plug-in ... 等等。

編輯 `/etc/neutron/neutron.conf`，修改成以下設定：

``` bash
# 註解掉所有 database 連線設定，因為 Compute Node 沒有網路相關資訊需要連線到資料庫存取
[database]

[DEFAULT]
rpc_backend = rabbit	# message broker 設定
rabbit_host = controller
rabbit_password = RABBIT_PASS
auth_strategy = keystone	# 認證授權設定
core_plugin = ml2	# network plug-in 設定
service_plugins = router
allow_overlapping_ips = True

# 認證授權設定
[keystone_authtoken]
#auth_host = 127.0.0.1
#auth_port = 35357
#auth_protocol = http
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = neutron
admin_password = NEUTRON_PASS
```

### 設定 Modular Layer 2 (ML2) plug-in @ Compute Node

ML2 plug-in 使用 Open vSwitch(OVS) 提供網路功能。

編輯 `/etc/neutron/plugins/ml2/ml2_conf.ini`，修改成以下設定：

``` bash
# agent driver 設定
[ml2]
type_drivers = flat,gre
tenant_network_types = gre
mechanism_drivers = openvswitch

# 設定 tunnel identifier (id) range
[ml2_type_gre]
tunnel_id_ranges = 1:1000

# 網路安全相關設定
[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

# 啟用 tunnel 功能並設定 local tunnel endpoing
# local_ip 要設定 Compute Node 的 Tunnel Interface IP (此範例中是 10.0.1.31)
[ovs]
local_ip = 10.0.1.31
enable_tunneling = True

# 啟用 GRE tunnel
[agent]
tunnel_types = gre
```

### 設定 Open vSwitch (OVS) service @ Compute Node

重新啟動 OVS service：

``` bash
root@compute1:~# service openvswitch-switch restart
```

### 設定讓 Nova 使用 Neutron 服務 @ Compute Node

在預設情況下，Nova 使用的是 nova-network 來提供網路功能，但範例中換成 Neutron，因此要進行一些調整設定。

編輯 `/etc/nova/nova.conf`，修改成以下設定：

``` bash
[DEFAULT]
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[neutron]
url = http://controller:9696
auth_strategy = keystone
admin_auth_url = http://controller:35357/v2.0
admin_tenant_name = service
admin_username = neutron
admin_password = NEUTRON_PASS
```

### 重新啟動服務 @ Compute Node

1、重新啟動 Compute Service

``` bash
root@compute1:~# service nova-compute restart
```

2、重新啟動 Open vSwitch(OVS) agent

``` bash
root@compute1:~# service neutron-plugin-openvswitch-agent restart
```

### 驗證是否安裝完成 @ Controller Node

驗證動作需要在 **Controller Node** 上執行。

首先切換為使用者 admin：

``` bash
root@controller:~# source ~/openstack/admin-openrc.sh
```

查詢目前 Neutron 上所安裝好的 agent 清單：

``` bash
root@controller:~# neutron agent-list
+--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host     | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
| 1bb499bc-dcc4-4d5d-8ae0-e64fc1b8c43b | DHCP agent         | network  | :-)   | True           | neutron-dhcp-agent        |
| 22fc052c-51cb-4c9d-9ed5-76a6b2ae1987 | Metadata agent     | network  | :-)   | True           | neutron-metadata-agent    |
| 41beba50-3076-4fae-ab29-53ab748de28f | Open vSwitch agent | network  | :-)   | True           | neutron-openvswitch-agent |
| bf189bbf-5f73-46d3-b9b9-57d8b9c6720f | L3 agent           | network  | :-)   | True           | neutron-l3-agent          |
| e8936978-6399-47e9-99b1-e6b6b3ebda05 | Open vSwitch agent | compute1 | :-)   | True           | neutron-openvswitch-agent |
+--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
```

除了原本的 Network Node 上的 agent 設定之外，也出現了一個在 Compute Node 上的 OVS agent 了! 如果出現以上訊息，就表示在 Compute Node 上的設定正確完成囉!

------------------------------

建立初始化網路
==============

在建立 VM 之前，我們要先將虛擬網路環境先準備好，VM 建立後才可以正常地進行通訊；初始化網路的架構中包含兩個部分，分別是 **external network** & **tenant network**。

來看看預計產生的網路架構圖：

![Neutron Initial Networks](http://docs.openstack.org/juno/install-guide/install/apt/content/figures/3/a/common/figures/installguide-neutron-initialnetworks.png)

我們這邊對外的環境跟 openstack 文件中的環境不同，所以對外網路的設定調整如下：

- IP/Netmask：192.168.20.0/24

- Gateway：192.168.20.254

- DHCP Range：192.168.20.[11-20]

### 建立 External Network @ Controller Node

external network 是提供 VM 可以連上網際網路之用。

要建立 External Network 必須要透過 controller node 發布命令，讓 Neutron 服務接收指令後來產生，所以以下的指令都需要在 controller node 上執行。

1、切換成使用者 admin

``` bash
root@controller:~# source ~/openstack/admin-openrc.sh
```

2、建立網路

``` bash
root@controller:~# neutron net-create ext-net --router:external True --provider:physical_network external --provider:network_type flat
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | db701eee-164f-4126-ba56-0b548e4838e8 |
| name                      | ext-net                              |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 095227f7415a4bdc9a4f48f620208fd7     |
+---------------------------+--------------------------------------+
```

3、建立子網路並附加在 External Network 上

建立了 external network 之後，接著就是要在上面指定相關網路設定(例如：IP 範圍 / CIDR / Gateway ... 等等)

``` bash
# 請根據自己的環境修改下面指令的 IP，包含 start/end ip, gateway, CIDR ... 等等
root@controller:~# neutron subnet-create ext-net --name ext-subnet --allocation-pool start=192.168.20.11,end=192.168.20.20 --disable-dhcp --gateway 192.168.20.254 192.168.20.0/24
Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "192.168.20.11", "end": "192.168.20.20"} |
| cidr              | 192.168.20.0/24                                    |
| dns_nameservers   |                                                    |
| enable_dhcp       | False                                              |
| gateway_ip        | 192.168.20.254                                     |
| host_routes       |                                                    |
| id                | 555e6dc6-0a67-49df-b848-e71347cec14d               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | ext-subnet                                         |
| network_id        | db701eee-164f-4126-ba56-0b548e4838e8               |
| tenant_id         | 095227f7415a4bdc9a4f48f620208fd7                   |
+-------------------+----------------------------------------------------+
```


### 建立 Tenant Network @ Controller Node

tenant network 是用來提供 VM 在內部通訊之用，每個 tenant 都屬於不同的網路架構，是完全隔離的!

以下指令同樣也是在 controller node 上執行，但由於我們要建立 tenant demo 的網路，因此**要切換身分為 demo**!

1、切換成使用者 demo

``` bash
root@controller:~# source ~/openstack/demo-openrc.sh
```

2、建立網路(tenant demo 使用)

``` bash
root@controller:~# neutron net-create demo-net
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | 3ed88154-cab0-442a-ad61-64c344e31b1d |
| name            | demo-net                             |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | 824e59f8952849648d19b99275e3a913     |
+-----------------+--------------------------------------+
```

3、建立子網路並附加在 Tenant Network 上

跟 external network 一樣，內部網路也是需要網路設定的相關資訊(CIDR、IP Range、Gateway ... 等等)，因此以下建立一個子網路供 tenant network 使用：

``` bash
# 使用 192.168.50.0/24 作為 tenant network 的網段
root@controller:~# neutron subnet-create demo-net --name demo-subnet --gateway 192.168.50.1 192.168.50.0/24
Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "192.168.50.2", "end": "192.168.50.254"} |
| cidr              | 192.168.50.0/24                                    |
| dns_nameservers   |                                                    |
| enable_dhcp       | True                                               |
| gateway_ip        | 192.168.50.1                                       |
| host_routes       |                                                    |
| id                | 8df0b029-b92d-43ca-b70b-d4cc6ecd3357               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | demo-subnet                                        |
| network_id        | 3ed88154-cab0-442a-ad61-64c344e31b1d               |
| tenant_id         | 824e59f8952849648d19b99275e3a913                   |
+-------------------+----------------------------------------------------+
```

4、建立 Router 並附加在 External/Tenant Network 上

上面將網路都建立好之後，問題是設定了 Gateway，但是都還沒串起來，封包要怎麼從 internal network 一路到 external network 呢?

這就是 router 要做的事情了，於是這裡要建立一個 router，並把內外部網路都串起來，概念上的示意如下：

`demo-subnet  <====>  demo-router  <====> ext-net  <====> Internet`

1、建立 router

``` bash
root@controller:~# neutron router-create demo-router
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | ce6803a7-90c7-4c35-b3bc-f0b14fd05e1e |
| name                  | demo-router                          |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | 824e59f8952849648d19b99275e3a913     |
+-----------------------+--------------------------------------+
```

2、將 tenant network 附加在 router 上

``` bash
root@controller:~# neutron router-interface-add demo-router demo-subnet
Added interface 0f845d2b-2af7-4183-bf4c-e0ba20d4171b to router demo-router.
```

3、指定 ext-net 為 router 對外的 gateway

``` bash
root@controller:~# neutron router-gateway-set demo-router ext-net
Set gateway for router demo-router
```


### 驗證通訊是否成功

在正常情況下，上面所建立的 router 會使用一個外部 IP(上面設定 192.168.20.[11-20])，預設所使用的是第一個(192.168.20.11)

因此我們可以嘗試 `ping 192.168.20.11` 來測試看看是否網路有安裝設定成功!

以下是測試結果：

``` bash
C:\>ping 192.168.20.11

Ping 192.168.20.11 (使用 32 位元組的資料):
回覆自 192.168.20.11: 位元組=32 時間=1ms TTL=63
回覆自 192.168.20.11: 位元組=32 時間=1ms TTL=63
回覆自 192.168.20.11: 位元組=32 時間<1ms TTL=63
回覆自 192.168.20.11: 位元組=32 時間=1ms TTL=63

192.168.20.11 的 Ping 統計資料:
    封包: 已傳送 = 4，已收到 = 4, 已遺失 = 0 (0% 遺失)，
大約的來回時間 (毫秒):
    最小值 = 0ms，最大值 = 1ms，平均 = 0ms
```

若明明按照設定一步一步完成，網路卻還是不通，請檢查：

1. 如果 openstack nodes 安裝在虛擬化環境上，對外網路的部分必須把 **promiscuous mode** 開啟

2. 建立網路需要一點時間，不會馬上通，等個幾分鐘也許就會通了 (我遇到的情況是這樣!)

------------------------------

參考資料
========

- [OpenStack Installation Guide for Ubuntu 14.04 - juno][1]

[1]:  http://docs.openstack.org/juno/install-guide/install/apt/content/