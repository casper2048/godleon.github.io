---
layout: post
title:  "[OpenStack] Nova Network 簡介"
description: "在這邊文章中，介紹 OpenStack 中 Nova Network 不同的運作型態與方式"
date: 2015-07-01 15:00:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Networking]
---

Network Concept
===============

目前 nova-network 僅支援透過 Linux Bridge 的方式連外。

nova-network 支援三種 network mode，分別是：

1. Flat Network Manager

2. Flat DHCP Network Manager

3. VLAN Network Manager

-------------------------------------------------------

Flat Network Manager
====================

在此模式下，與 VM 相關的網路設定都必須由管理員來負責處理，包含 IP、Bridge、Subnet .... 等等，所有的 VM 都會掛載 Compute node 上的 Linux bridge。

在 Flat Network Manager 的環境中，DHCP & DNS 的服務是來自於外部

![FlatDHCPManager – network topology](https://www.mirantis.com/wp-content/uploads/2012/07/flatdhcpmanager-topology-single-instance.png)

-------------------------------------------------------

Flat DHCP Network Manager
=========================

與上一個模式相同，所有的 VM 都會掛載 Compute node 上的 Linux bridge；但在此模式下，VM IP 的派送則會由掛載在 Linux bridge 上的 **dnsmasq** 程序來處理，管理員只要設定以下資訊：

- VM IP 所屬的 subnet

- Linux bridge 橋接的實體網路卡(透過 <font color='red'>**flat_interfac**</font> 參數，預設為 **eth0**)

- Linux bridge (透過 <font color='red'>**flat_network_bridge**</font> 參數，預設為 **br100**)

> 在 single-host Flat DHCP 模式下，可以從安裝 nova-network 的 node(一般為 controller) ping 到 VM，但無法從 compute node ping 到 VM。

<font color='red'>**【註】**</font>dnsmasq 發給 VM 的是固定 ip，這樣可以確保 VM 反覆啟動都可以取得相同 IP 以便於管理。 


在 Flat DHCP Network Manager 的環境中，使用 **dnsmasq** 與 br100 綁定，用以提供 DHCP & DNS 服務給內部的 instance，而每個 instance 的 default gateway 則會因為所在 compute node 的不同而不同。(例如：vm_1 的 Default GW 為 10.0.0.1，vm_3 的 Default GW 為 10.0.0.4)

![Network gateways for instances running on different compute nodes](https://www.mirantis.com/wp-content/uploads/2012/07/flat-dhcp-networking-diagrams-4.png)

在 FLAT 的環境下，表示所有 vm 都在同一個 broadcast domain，因此在 L2 的層級上可以相互看到對方，若是要停止 vm 之間的互通，可用以下設定：

**/etc/nova.conf**
``` bash
# instance(vm) 之間無法相互溝通
allow_same_net_traffic=False
```

### Single-host Flat DHCP Network Manager

Single-host 的環境下，運作模式如下：

1. nova-network & dnsmasq 會安裝在 controller 上，DHCP & DNS 的服務由 controller 提供。

2. VM 的 default gateway 會設定為 controller br100 的 ip。(也就是下圖中的 10.0.0.1)

3. 所有連外的網路流量會透過 controller 上的 br100 進出。

4. VM 之間的流量並不會經過 controller。

5. 在 controller 上的 nove-network 扮演著連接 vm 的 switch 角色。

6. 沒有 HA 的保護

![Single-host FlatDHCPManager](https://www.mirantis.com/wp-content/uploads/2012/07/FlatDHCP.png)

### Promiscuous Mode 

網路的預設情況中，NIC 只會接收屬於包含其 MAC address 的封包，但對外連線的 NIC 必須處理許多 VM 的封包，而每個 VM 都有各自的 MAC address，因此若沒有開啟 Promiscuous Mode，NIC 就不會接收 VM 的網路封包並處理；因此要將 NIC 的 Promiscuous Mode  開啟，這樣才可以接收不屬於 NIC MAC address 的封包並進行後續處理。

-------------------------------------------------------


Flat Network Manager & Flat DHCP Network Manager
================================================

- 在 Flat 的環境中，每個 compute node 都會有一個對外連線用的 bridge device(預設為 **br100**，使用 **flat_network_bridge** 參數設定)。

- 所有在 compute node 裡的 VM，都是透過 **br100** 對外連線 or 溝通。

**/etc/nova.conf**
``` bash
flat_network_bridge=br100
```

![Network bridging on OpenStack compute node](https://www.mirantis.com/wp-content/uploads/2012/07/generic-bridge-config-2.png)

- 透過 Linux Bridge 的作法有個限制，就是 Bridge Device 僅能與一個 NIC 連接。

- 在 FlatManager & FlatDHCPManager 的環境中，無法做到 L2 isolation，因此所有 VM 都在同一個 ARP broadcast domain 內(極為同一個 ip pool)。


-------------------------------------------------------

VLAN Network Manager
====================

VLAN 為 OpenStack Compute 的預設模式，主要用來解決 FLAT 模式的兩個缺點：

1. 缺乏擴充彈性(只有一個 Layer 2 的網段可用)

2. 沒有 tenant isolation 的功能，所有 VM 都在同一個 ARP domain 中

在設定使用者可用的 subnet 時，管理者必須多設定兩種資訊，分別是：

1. VLAN ID

2. Tenant ID

設定範例如下：

``` bash
$ nova-manage network create --fixed_range_v4=10.0.1.0/24 --vlan=102 \
 --project_id="tenantID"
```

### single tenant @ single host

在 single tenant(t1) 的狀況下，若產生一個 VM(t1_vm_1)，並指派 VLAN 102，則 network topology 會如下圖：

![single tenant with 1 VM](https://www.mirantis.com/wp-content/uploads/2012/07/vlanmanager-generic-config-2-tenants-2.png)

> tenant 在 compute node 上有一個專屬的 Linux bridge(br102)，且 tenant 所擁有的 VM 都會連結到這個 Linux bridge 上，並且會有專屬的 dnsmasq 來作為 IP & DNS 的管理


若 single tenant(t1) 有兩個 VM(t1_vm_1, t1_vm_2)，則 network topology 會變成下圖：

![single tenant with 2 VMs](https://www.mirantis.com/wp-content/uploads/2012/07/vlanmanager-generic-config-v2-2-hosts-same-tenant-1.png)

> 同樣一個 dnsmasq & Linux bridge，每個 VM 都連到相同的 Linux bridge 上

### multiple tenants @ single host

若 multiple tenants(t1 & t2) 的狀況下，且 t2 有一個 VM(t2_vm_1)，則 network topology 會如下圖：

![multiple tenants with 3 VMs](https://www.mirantis.com/wp-content/uploads/2012/07/vlanmanager-generic-config-v2-2-tenants-2.png)

> 每個 tenant 屬於不同的 VLAN(t1 使用 VLAN 102，t2 使用 VLAN 103)，每個 VLAN 都會有專屬的 Linux bridge 與 dnsmasq，每個 tenant 各自的 VM 都連結到各自的 Linux bridge

從上圖可以想見，若是有很多 tenant，自然就會有很多個 Linux bridge & dnsmasq，只是這些都會由 OpenStack 來負責管理，透過這種方式達成 L2 層級的分割，讓不同的 tenant 的網路各自屬於不同的 ARP(Broadcast) domain。

### multiple tenants @ multiple hosts

Network topology 如下：

![multiple tenants in multiple hosts](https://www.mirantis.com/wp-content/uploads/2012/07/vlanmanager-2-hosts-2-tenants.png)

跟傳統 VLAN 環境的設定很類似，有幾個重點需要了解：

1. 需要一個支援 VLAN Trunking 的實體 switch

2. switch 連接 compute node 的 port 都必須是 trunk port，才能傳輸多個 VLAN 的流量

3. 不同 compute node 但相同 VLAN 的 VM 傳遞資料，則是透過兩個 compute node 上的 Linux bridge 經由實體 switch 的 trunk port 來溝通

### 設定方式

只要在 **/etc/nova.conf** 中設定以下內容即可：

``` ini
# 設定 network manager 為 VLAN Manager
network_manager=nova.network.manager.VlanManager 

# VLAN 網路流量所使用的實體網卡
vlan_interface=eth0 

# 可配置的 VLAN number
vlan_start=100
```

--------------------------------------------------------

參考資料
========

- [OpenStack Networking - FlatManager and FlatDHCPManager - Pure Play OpenStack.](https://www.mirantis.com/blog/openstack-networking-flatmanager-and-flatdhcpmanager/)

- [OpenStack Networking Tutorial: Single-host FlatDHCPManager](https://www.mirantis.com/blog/openstack-networking-single-host-flatdhcpmanager/)

- [Openstack Networking | Scalability and Multi-tenancy with VlanManager](https://www.mirantis.com/blog/openstack-networking-vlanmanager/)

- [Networking with nova-network - OpenStack Cloud Administrator Guide](http://docs.openstack.org/admin-guide-cloud/content/section_networking-nova.html)