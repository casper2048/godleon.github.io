---
layout: post
title:  "CoreOS 概觀"
description: "此文章簡單的描述了 CoreOS 的特色 & 相關組件，是個學習 CoreOS 的開始"
date: 2016-06-08 14:30:00
published: true
comments: true
categories: [coreos]
tags: [Linux, CoreOS]
---

CoreOS Overview
===============

CoreOS 並沒有使用類似 yum or apt 的套件管理程式，而是把所有服務都放進了 container

## Docker (Conainer Service)

![Run Services with docker](https://coreos.com/assets/images/media/Three-Tier-Webapp.png)

CoreOS 原生就是設計為了處理 multiple host 而生，因此裡面包含了很多工具，可以用來協調整合在不同 host 上的 container，包含了 etcd, fleet .... 等等

### fleet (Cluster Management)

[fleet](https://coreos.com/using-coreos/clustering/) 透過使用 [systemd unit files](https://coreos.com/using-coreos/systemd/) 可以同時對多台 CoreOS host 上的服務進行部屬，確保服務各自安裝在不同的 host 上，藉此提供 high availability 的效果

### etcd (Service Discovery)

每個 CoreOS host 都會帶有 etcd(`http://127.0.0.1:4001`)，負責用來執行 service discovery & 讀寫設定值之用。

以一個簡單例子來說，假設要執行 Wordpress，再也不需要在程式中指定資料庫的位址，取而代之的是向 etcd 發送類似 `http://127.0.0.1:4001/v1/keys/database` 的 request，藉此取得目前提供服務的資料庫位址。

當提供服務的 container 啟動後，都會向 etcd 註冊，之後 proxy 就會持續監控值的變化，當相關資料有變化時(例如 container 重起在另外一台 host 上)，proxy 就會自動得知消息，且會把相關的流量導向新的 container，透過這種方式，進行 service scale 就相對的簡單的多

-------------------------------------------------------------

Container Overview
==================

![single CoreOS host and the services running on it](https://coreos.com/assets/images/media/Host-Diagram.png)

上圖表示在 CoreOS 中，不僅只有 docker engine 存在，同時也存在著 etcd 作為輔助的角色；當每個 container 啟動來提供服務時，都會透過 etcd 來通知 proxy 可以開始把工作流量送給它們

同時，在 CoreOS 上執行的 conainer 會有以下幾個特點：

- CoreOS 會自動的處理 & 設定 container runtime

- container image 會跟著 OS update 同時更新

- 與 etcd 整合

- 自動設定網路相關功能

-------------------------------------------------------------

systemd Overview
================

CoreOS 使用 `systemd` 作為 fleet 的核心並高度整合，而使用 systemd 的理由如下：

- 系統開機速度非常快

- 提供方便且功能豐富的 journal

- 提供 socket activation 的功能，對內部服務的相依管理上幫助很大

systemd 提供非常豐富的方式讓使用者可以用文字檔案的方式描述 service，以下是個 service file 的簡單範例：

```ini
[Unit]
Description=My Service
Requires=docker.service
After=docker.service

[Service]
ExecStart=/usr/bin/docker run busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"

[Install]
WantedBy=multi-user.target
```

-------------------------------------------------------------

fleet - Cluster Management
==========================

透過 fleet，使用者可以把整個 CoreOS cluster 當作是一個單獨的 init 系統，並且讓使用者不用煩惱到底服務是執行在哪些機器的哪些 container 上，fleet 會自動協助安排處理，如下圖：

![fleet scheduling](https://coreos.com/assets/images/media/Fleet-Scheduling.png)

圖中使用者要求要執行 6 個 API server & 2 個 Load Balancer 服務，fleet 會自動安排要將服務放到哪個 host 的哪個 container 上，完全不用使用者擔心。

而 service HA 會透過 fleet 把 service 放在不同的機器 or AZ or region 來自動達成。

以下是 fleet 的幾個特性：

- 當 machine 故障時，fleet 會協助將上面的 container workload 進行 reschedule

- 當有新機器加入 cluster 中時，fleet 會自動發現

- 自動 ssh 進入到 machine 執行該做的事情

-------------------------------------------------------------

etcd Overview
=============

etcd 在一群機器中提供了可靠的分散式 key/value 儲存機制，應用上例如像是儲存 database connection 的資訊，或是實做 distributed locking 機制等等。

而存在於 etcd 上的 key/value 都是會被持續監控的，當值有變化時，會根據使用者需求執行相對應的工作。

-------------------------------------------------------------

References
==========

## [Using CoreOS](https://coreos.com/using-coreos/)

- [docker](https://coreos.com/using-coreos/containers/)

- [systemd](https://coreos.com/using-coreos/systemd/)

- [fleet](https://coreos.com/using-coreos/clustering/)

- [etcd](https://coreos.com/etcd/)

- [Updates & Patches](https://coreos.com/using-coreos/updates/)

## Others

- [CoreOS那些事之系统升级 - InfoQ](http://www.infoq.com/cn/articles/coreos-system-upgrade)
