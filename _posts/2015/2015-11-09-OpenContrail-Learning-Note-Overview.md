---
layout: post
title:  "OpenContrail 學習筆記 - 概觀"
description: "此文章記錄學習 OpenContrail 的過程，此篇謹為 overview 的筆記，記錄了 OpenContrail 的組成元件以及運作方式"
date: 2015-11-09 13:50:00
published: true
comments: true
categories: [sdn]
tags: [SDN, OpenContrail]
---


OpenContrail Controller and the vRouter
=======================================

OpenContrail 系統分為兩個主要部份：

1. **OpenContrail Controller**
> controller 是個實際上分散，但邏輯上集中的 SDN controller，負責管理、控制、分析 virtual network，提供一個邏輯上集中的 **control plane** & **management plane** 來管理整個系統以及 vRouter。

2. **OpenContrail vRouter**
> vRouter 是運行在 hypervisor 上的 **forwarding plane**(可參考下圖)，並將實體的 router & switch 轉化為 virtual overlay，藉此方式將不同實體機器上的 hypervisor 中的 VMs 連接起來。

![vRouter Forwarding Plane](http://www.opencontrail.org/wp-content/uploads/2014/10/Figure04.png)

------------------------------------

Virtual Networks
================

virtual network 是 OpenContrail 中相當重要的一個概念，此功能的特色與目的為：

1. 取代 VLAN-based 的方式，將網路隔離

2. 在整體虛擬網路環境中提供 multi-tenant 的功能

3. virtual network 可以輕易的透過 MPLS / L3VPN 的方式擴展(使用 edge router)

4. 實現 NFV & service chaining 的功能，為網路服務進行加值

------------------------------------

Overlay Networking
==================

在自家資料中心要打造出有 virtual network 的環境，需要實體的網路環境 & 虛擬的 overlay network。

而什麼是 overlay network 呢? 關於 overaly network，以下借一下 [MidoNet](http://www.midokura.com/midonet/) 的圖來說明：

！[Overlay Network](http://bradhedlund2.s3.amazonaws.com/2012/midonet/midonet1.PNG)

其中下層的部份屬於實體的網路環境，包含了 server / storage / router / switch ....等設備，這些設備在區域網路中透過 TCP/IP 協定，以低延遲的實體線路相連，也就是圖中下層的部份。

接著運行在 hypervisor 中的 vRouter 會以底層的實體線路為基礎，以 mesh 的方式對其他機器上的 hypervisor 建立出許多 tunnel，因而在其之上建立出 virtual overlay network，也就是圖中上層的部份。

在 OpenContrail 中，會以 MPLS over GRE/UDP/VxLAN 的方式建立 tunnel。

在實體環境中的 router & switch，並不會紀錄每個 VM 的 MAC，這些設備僅會紀錄實體設備的 MAC & IP prefixes 資訊，
像是 server / router / switch 等。

但 vRouter 則會紀錄 tenant 的狀態，以及 VM 的 MAC & IP prefixes 等資訊，vRouter 會為每個跟其有相關的 virtual network(<font color='red'>**非全部**</font>) 建立出一個獨立的 forwarding table 並維護。

-------------------------

Overlays based on MPLS L3VPNs and EVPNs
=======================================

OpenContrail 沿用了原本已經就很成熟的 MPLS L3VPNs (for L3 overlays) & MPLS EVPNs (for L2 overlays) 兩個技術，試圖完成在 overlay network 上 L2 & L3 的 function。

### data plane

此部份 OpenContrail 支援 MPLS over GRE，目前已經有許多網通設備供應商支援此功能；也支援 MPLS over UDP，對 multi-pathing 有較好的支援，並且 CPU 利用率會比較好；當然 MPLS over VXLAN 也同樣支援。

其他封裝技術像是 NVGRE 則是可以在未來的更新中陸陸續續加入。

### control plane

在 OpenContrail 中，control node 與其他實體網通設備(route, switch) 進行資料交換協定與 MPLS L3VPN & MPLS EVPN 相同，都是 BGP。

而 control node 與 vRouter 之間的資訊交換，則是透過 XMPP，其中交換資訊的描述方式與 BGP 所使用的相當接近。

-------------------------

OpenContrail and Open Source
============================

目前 OpenContrail 可與其他 open source solution 的整合狀況如下：

1. Hypervisor 的部份可以與 KVM & Xen 整合

2. Cloud 平台可與 OpenStack & CloudStack 整合

3. 可以其他管理系統整合，例如：chef, puppet, cobbler, ganglia .... 等等。

-------------------------

Scale-Out Architecture and High Availability
============================================

OpenContrail controller 設計成在邏輯上可集中管理，但實際可分散安裝佈署的架構；OpenContrail controller 是由多種不同的 node 所組合而成，而每種 node type 都可以進行水平擴展 or 因應 HA，而同時佈署多個 node instance。(可以是實體 or 虛擬型式)

而若要組成 OpenContrail controller 的完整功能，需要以下三種 node type instance：

### 1. configuration node

在北向(north-bound)提供 REST API，讓外部可透過 REST API 進行 network 配置要求 & 取得相關的操作資訊要求；另外也透過 IF-MAP protocol 與 control node 進行資料的傳遞。

此外，REST API 也是以相當安全的方式提供，除了以 HTTPS 進行認証 & 加密外，也提供了 role-based 的認証；此外，還可透過 multiple configuration node instance 的配置，來分散 API 的 loading。

### 2. control node

實作了管理 control plane 的功能，而並非所有的 control plane function 邏輯上都是集中管理的，有些還是分散在實體/虛擬 router or switch 上。

control node 透過 IF-MAP protocol 監控來自於 configuration node 所傳遞過來的網路狀態資訊，根據來自 configuration node 所發出的 network topology 的配置要求，透過 XMPP 控制在各個 compute node 上的 vRouter，並搭配現有的 BGP/DNS 等協定，產生出相對應的 network topology。

### 3. analytics node

執行 analytic data collector，負責收集、整理、展示網路管理相關的分析資訊，並提供 analytics API & Query engine 等功能。

來自所有 network componet 的大量資料，都會傳到 analytic node 上，並經過處理後，以最適合作為時間序列分析的方式，存到由 analytic node 所管理的分散式資料庫中。

此外，當有緊急事件發生時，analytic node 還有能力去要求 network component 提供更多更詳細的資訊，以便後續問題處理之用。


> 透過實體上的分散的 OpenContrail controller，達到了 redundant 的效果；以 active-active 的方式運作，讓系統可以在任何一個 node 損壞時，無中斷的繼續提供服務。
> 而 OpenContrail 其中一個功能亮點，則是當有某個 node 負載過重時，系統會再產生一個相同的 node 來平衡負載，這樣的作法可以避免單一個 node 成為效能上的 bottleneck，讓系統得以擴展至非常大的規模。

-------------------------

The Central Role of Data Models: SDN as a Compiler
==================================================

Data Model 在 OpenContrail 中是個很重要的概念，包含了多個 object，object 所擁有的能力，以及 object 之間的關係。

Data Model 提供了透過宣告式語法來定義 network 的能力，因此也表示了可以達到 Network Infrastructure as Code 的目的。

Data Model 目前都是透過 IF-MAP XML schema 來描述(未來有可能會改用 YANG)，共分成兩種：

- high-level service data model

> 此種 Data Model 用較上層的抽象方式來描述網路的狀態，內容包含了像是 Tenant、Virtual Network、Connectivity Policy、Security Policy 等。

- low-level technology data model

> 此種 Data Model 則是用底層的抽象方式來描述網路狀態，內容像是 BGP route-target、VXLAN network identifier 等。

其中 configuration node 負責將任何網路相關的變動，從 high-level service data model 轉譯為 low-level service data model(利用一個稱為 <font color='red'>**transformation engine**</font> 的機制)；而 control node 則是負責解析 low-level service data model，並透過南向(south-bound)的相關協定(例如：XMPP、BGP、Netconf)，在實際的網路環境中實現。

-------------------------

參考資料
=======

- [What is IF-MAP? | IF-MAP](http://www.if-map.org/what_is_if-map)

- [IF-MAP solutions](http://www.if-map.org/solutions)

- [Juniper看好IF-MAP協定將成為NAC的共通標準 | iThome](http://www.ithome.com.tw/node/55834)

- [Jan Ho 的 Cisco 網絡世界 CCNA CCNP CCIE 教學網站: Border Gateway Protocol (BGP)]()

- [什麼是Mpls? - iT邦幫忙::IT知識分享社群](http://ithelp.ithome.com.tw/question/10098559)

- [Seednet教室 = =MPLS 概論 = =](http://eservice.seed.net.tw/class/class0801c.html)

- [YANG - Wikipedia, the free encyclopedia](https://en.wikipedia.org/wiki/YANG)
