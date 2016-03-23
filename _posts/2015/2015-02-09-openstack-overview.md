---
layout: post
title:  "[OpenStack] OpenStack 概觀"
date:   2015-02-09 16:00:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack]
---

簡介
====

以下是官網的介紹：

> OpenStack software controls large pools of compute, storage, and networking resources throughout a datacenter, managed through a dashboard or via the OpenStack API. OpenStack works with popular enterprise and open source technologies making it ideal for heterogeneous infrastructure.

OpenStack 是一套用來打造 IaaS 服務的開源軟體，隨著新版本的持續推出(目前為 Juno)，雲端管理平台所需的功能也都逐漸趨於完整，舉凡最基本的運算、儲存、網路之外，也包含了監控、大數據....等不同相關的功能也都逐漸推出，並可相容於多種異質平台，對於想避免 vendor-locking 的使用者來說，可說是一大福音。

--------------------

架構
====

OpenStack 中每個服務的相互關係可以參考下圖：

![Conceptual architecture](http://docs.openstack.org/juno/install-guide/install/apt/content/figures/1/a/common/figures/openstack_havana_conceptual_arch.png)

接著說明每個服務的功能：

### General

| Service | Project | 說明 |
|---------|---------|------|
| Dashboard | **Horizon** | 提供一個管理用網站，可用來啟動新的 VM，配發 IP、設定 ACL 相關規則....等用途 |
| Computer | **Nova** | 提供部署與管理虛擬機器的功能 |
| Networging | **Neutron** | 提供 Network-as-a-Service 給其他 project 所使用，可讓其自行定義相關網路需求設定，並附加在指定的服務上。(各大網通廠商皆可根據標準的 API 實作出可與 Neutron 搭配的硬體與軟體) |

### Storage

| Service | Project | 說明 |
|---------|---------|------|
| Object Storage | **Swift** | 是個分散式儲存平臺，可存放非結構化的資料，並提供 Restful API 供其他專案使用，類似 Amazon S3 |
| Block Storage | **Cinder** | 提供區塊儲存容量，具有快照功能，功能類似 Amazon Elastic Block Store |

### Shared Services

| Service | Project | 說明 |
|---------|---------|------|
| Identity Service | **Keystone** | 提供認證 & 授權之功能；提供了多種驗證方式，能查看哪位使用者可存取哪些服務 |
| Image Service | **Glance** | 提供映象檔尋找、註冊以及服務交付等功能 |
| Telemetry | **Ceilometer** | 監控 OpenStack 各服務運作的狀況，並提供費用計算、效能測試、擴充性數據、統計數據....等資訊 |

### High-level Services

| Service | Project | 說明 |
|---------|---------|------|
| Orchestration | **Heat** | 透過 Heat，可使用一份預定定義好的設定檔，快速的將所指定的運算/儲存/網路...等服務部署起來，功能類似 Amazon CloudFormation |
| Database Service | **Trove** | 提供 Database-as-a-service 的功能，包含 RDBMS & NoSQL 皆有，讓使用者不需要在自行安裝 & 維護資料庫，可以很簡單的在 OpenStack 上就啟動一個資料庫服務，功能類似 Amazon RDS & DynamoDB |
>

--------------------

邏輯架構
========

在 OpenStack 中的每個 service 都有提供 API 與其他 service 進行溝通合作，以下用一張圖來表示 service 相互合作的邏輯關係圖：

![OpenStack Logic Architecture](http://docs.openstack.org/openstack-ops/content/figures/2/figures/osog_0001.png)

--------------------

參考資料
========

- [雲端運算大革命 OpenStack具3大優勢快速發展 | iThome](http://www.ithome.com.tw/node/81091)

- [Chapter 1. Architecture - OpenStack Installation Guide for Ubuntu 14.04 - juno](http://docs.openstack.org/juno/install-guide/install/apt/content/ch_overview.html)
