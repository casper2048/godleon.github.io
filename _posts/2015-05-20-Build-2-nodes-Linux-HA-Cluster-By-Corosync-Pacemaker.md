---
layout: post
title:  "[HA] 建置 Two Nodes HA Cluster by corosync & pacemake @ Ubuntu 14.04"
description: "此篇文章介紹如何在 Ubnutu 14.04 的環境中，使用 corosync + pacemake 兩種工具，建置 Linux HA cluster，並搭配實際服務進行驗證"
date:   2015-05-20 14:50:00
published: true
comments: true
categories: [ha]
tags: [HA, Corosync, Pacemaker, Linux, Ubuntu]
---


淺談 Pacemaker 架構
===================

### Pacemake Stack

首先用以下的圖來說明如何打造出 pacemaker stack 的 HA 架構：

![Pacemaker Stack](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Clusters_from_Scratch/images/pcmk-stack.png)

依照上圖，將 pacemaker Stack 分為三個部分。

#### 1、受到 HA 架構保護的服務 or 元件

此為受 HA 保護的資源 or 服務本身，在圖中為 cLVM2 / GFS2 / OCFS2 ... 等服務。

這些服務都有符合 [OCF(Open Cluster Framework)](http://www.opencf.org/home.html) 的標準，因此開發人員可以針對 OCF 的 API 開發出可以管理服務的 script(稱為 <font color='blue'>**Resource Agent**</font>)，然後接受上層(pacemaker)的管理控制。

#### 2、資源管理層

此部分為 pacemaker 本身，在整個架構中，pacemaker 扮演著大腦的角色，所有 node 與 cluster 之間互動(Join / Leave)所會觸發的事件都是由 pacemaker 來管理。

pacemaker 會根據使用者的設定，維持 cluster 在最好的狀態；當有服務中止時，馬上會找到另一個可替代的 node 來繼續提供服務；或是當有新的 node 加入時，可能又會針對不同的服務進行資源配置的調整。

#### 3、溝通底層

要讓 pacemaker 知道所有 resource(nodes / services ... 等等) 的狀態，就要仰賴底層元件的溝通了，而這底層元件，不僅執行溝通的工作，同時也包含了 membership & quorum 等其他工作。

目前這些元件的選擇有 Corosync、Heartbeat、CMAN .... 等等，在此架構中，我們選擇最新的 Corosync 作為底層溝通的元件。


### Pacemaker 內部元件

![Pacemaker Internal Components](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Clusters_from_Scratch/images/pcmk-internals.png)

pacemaker 扮演資源管理的大腦角色，內部自然就有多個元件緊密的運作著，而主要的元件共有五個：

1. **Cluster Information Base (CIB)**

2. **Cluster Resource Management daemon (CRMd)**

3. **Local Resource Management daemon (LRMd)**

4. **Policy Engine (PEngine or PE)**

5. **Fencing daemon (STONITHd)**

pacemaker 的運作方式大概如下：

- 在整個 Cluster 中的每一個 node 都會安裝 pacemaker 的服務，並推選出一個作為主要管理的 node，而這個 node 上面的 CRMd 則是作為 master 的角色來管理整個 cluster 的運作；若是 master node 掛掉了，則另外一個會很快地被推舉出來接手後續工作。

- CIB 是儲存 cluster 中所有 resource 資訊的地方，而這些資訊會同步到所有 pacemaker 的 CIB 上，確保每一個 pacemaker 服務都有 cluster 完整的 resource 資訊。

- CIB 資訊會被 PEngine 拿來運用，在依照制定好的 policy 為前提下，作為將 cluster 最佳化的依據。

- CRMd 會接收來自 PEngine 的資訊，傳遞控制訊息至 LRMd 來進行 resource 的調配；甚至是透過底層架構(Corosync)傳訊息至其他節點的 CRMd，藉此與其他 node 的 LRMd 進行溝通。

- 所有非 master 的節點都會透過底層架構持續與 master CRMd 通訊，並把相關資料交由 PEngine 來決定後續 resource 是否需要進行調整。

- 在某些特殊情況下，CRMd 可能會下命令給 STONITHd，將遠端的某個 resource 關閉，甚至把機器關機(進行隔離)。

- STONITH 裝置在 pacemaker 架構下也會被視為 resource 的一種，因此相關資訊都會存在 CIB 中。


---------------------------------


準備 Cluster 環境
=================

### 1. 機器設定

我們所準備的 Cluster Nodes 如下：(OS 為 **Ubuntu 14.04.02**)

- Cluster Node 1：192.168.122.101 (pcmk-1)

- Cluster Node 2：192.168.122.102 (pcmk-2)

- NFS Node：192.168.122.10
> 開放 /var/nfs 讓 node 1 & 2 掛載到 /usr/share/nginx/html(nginx 預設目錄)

編輯 NFS node 上的 /var/nfs/index.html，內容如下：

``` html
<!DOCTYPE html>
<html>
<head>
    <title>Cluster Test</title>
</head>
<body>
    <h1>This is Index page from NFS server</h1>
</body>
</html>
```

### 2. SSH 互連信任設定

node 1 & 2 之間必須設定 SSH 連線信任的機制(其實就是免密碼登入 SSH)，詳細設定方式可參考[此篇文章](http://www.clearcenter.com/support/documentation/clearos_guides/setting_up_ssh_trust_between_two_servers)。

---------------------------------


安裝 & 設定 cluster
=====================

### 1. 安裝套件

為了使用完整功能，下面把相關的套件都安裝起來：(在所有的 node 上)

``` bash
$ sudo apt-get -y install corosync pacemake fence-agents resource-agents pssh crmsh nginx
```

### 2. 設定 corosync

修改 <font color='blue'>**/etc/corosync/corosync.conf**</font> 如下：

``` bash
totem {
    version: 2
    secauth: off
    cluster_name: mycluster
    
    interface {
        ringnumber: 0
        bindnetaddr: 192.168.122.0
        mcastaddr: 226.94.1.1
        mcastport: 5405
    }
}

amf {
    mode: disabled
}
service {
    ver: 0
    name: pacemaker
}
aisexec {
    user: root
    group: root
}

quorum {
    provider: corosync_votequorum
    expected_votes: 1
    # Enables two node cluster operations
    two_node: 1
}

logging {
    fileline: off
    to_stderr: yes
    to_logfile: yes
    to_syslog: yes
    logfile: /var/log/cluster/corosync.log
    debug: off
    timestamp: on
    logger_subsys {
        subsys: AMF
        debug: off
    }
}
```

<font color='red'>**【註】**</font>在 Ubuntu 上沒有 pcs 套件(RedHat 才有)，因此 Corosync 設定檔就必須要靠手動來同步到 cluster nodes 上了。

### 3. 設定 corosync 開機時啟動

``` bash
$ sudo sh -c "echo 'START=yes' > /etc/default/corosync"
```

### 4. 產生 Authentication Key

接著要產生 cluster nodes 之間通訊加密時所使用的金鑰：

``` bash
# 使用簡易型的金鑰
$ sudo corosync-keygen -l
```

<font color='red'>**【註1】**</font>若不加 **-l** 則可以產生較為複雜的金鑰，但需要等待蠻久的。

<font color='red'>**【註2】**</font>此檔案也同必須同步到每個 node 的 **/etc/corosync** 目錄中。


### 5. 調整 STONITH & quorum 相關設定

STONITH(Shoot The Other Node In The Head) 的功能是將設備進行隔離之用，當 cluster 中有 node 出現問題影響到整體運作時，STONITHd 就可以將其隔離(甚至關機)來維持 cluster 的順暢運作。

而 STONITH + quorum(選舉 DC 的機制，最低人數限制) 的功能是為了避免 cluster 中出現 brain split 的現象產生，詳細的說明可以參考[此篇文章](http://nmshuishui.blog.51cto.com/1850554/1399811)。
 
在 two nodes cluster 中不需要 STONITH & quorum，因此這兩個功能必須透過 crm 命令將其關閉：

``` bash
$ sudo crm configure property stonith-enabled=false
$ sudo crm configure property no-quorum-policy=ignore
```

### 5. 重新啟動 corosync & pacemaker 服務

``` bash
$ sudo service corosync restart
$ sudo service pacemake restart
```

最後到 **/var/log/cluster/corosync.log** 可以看到 corosync 啟動以及 cluster 運作的相關資訊。


---------------------------------


建立 Resource 並測試 (以 Virtual IP 為例)
=========================================

### 1. 建立 VIP(Virtual IP)

我們要在 cluster 外層建立一個 Virtual IP(192.168.122.200)，讓外部來的連線都指向此 IP，而在後端實際上提供服務的 IP 則是 192.168.122.101 & 192.168.122.102 兩個內部的 IP。

在 node 1 執行以下指令，加入 resource：

``` bash
# resource name = VIP (ocf:heartbeat:IPaddr2)
# 設定 op type = monitor，指定 pacemake 對 resource 進行監控(間隔 10 秒，20 秒則算逾時)
$ sudo crm configure primitive VIP ocf:heartbeat:IPaddr2 params ip=192.168.122.200 nic=eth1 op monitor interval=10s timeout=20s on-fail=restart
```

接著查詢目前 Cluster 狀態，可以看見 VIP 已經設定完成

``` bash
$ sudo crm status
Last updated: Wed May 20 01:51:40 2015
Last change: Wed May 20 01:33:01 2015 via cibadmin on pcmk-2
Stack: corosync
Current DC: pcmk-2 (3232266854) - partition with quorum
Version: 1.1.10-42f2063
2 Nodes configured
1 Resources configured


Online: [ pcmk-1 pcmk-2 ]

 VIP    (ocf::heartbeat:IPaddr2):       Started pcmk-1
```

也可以使用以下指令查詢目前 cluster 的設定狀態：

``` bash
$ sudo crm configure show
node $id="3232266853" pcmk-1 \
        attributes standby="off"
node $id="3232266854" pcmk-2
primitive VIP ocf:heartbeat:IPaddr2 \
        params ip="192.168.122.200" \
        op monitor interval="10s" on-fail="restart"
property $id="cib-bootstrap-options" \
        dc-version="1.1.10-42f2063" \
        cluster-infrastructure="corosync" \
        stonith-enabled="false" \
        no-quorum-policy="ignore"
```

上面比較特別的是<font color='red'>**透過設定 op type = monitor，指定 pacemake 對 resource 進行監控**</font>，這是甚麼意思呢? 以下用列表來說明：

由於 corosync 是屬於底層訊息溝通用，因此<font color='blue'>**只會監控 node 的心跳狀態**</font>，若是在 node 上的服務(resource) 掛掉了，corosync 是不會知道的.....那怎辦? 於是就可以透過將 op type 設定為 monitor，請 pacemake 來對服務進行心跳的監控。

因此以上的組合會得出下表結論：

|          | corosync | corosync + pacemake |
|----------|----------|---------------------|
| 節點掛掉 |  可偵測/可轉移 | 可偵測/可轉移 |
| 服務掛掉 | 不可偵測/不可轉移  | 可偵測/可轉移 |

### 2. 測試 VIP 是否有效

我們透過以下步驟進行測試：

#### (1) 外部測試 VIP

使用 ping 指令持續偵測 VIP 的回應狀況。

#### (2) 從 node2 停用 node1 的 corosync 服務

``` bash
$ ssh pcmk-1 "sudo service corosync stop"
```

接著觀察目前 cluster 狀態：

``` bash
$ sudo crm status
Last updated: Wed May 20 02:01:43 2015
Last change: Wed May 20 01:33:01 2015 via cibadmin on pcmk-2
Stack: corosync
Current DC: pcmk-2 (3232266854) - partition with quorum
Version: 1.1.10-42f2063
2 Nodes configured
1 Resources configured


Online: [ pcmk-2 ]
OFFLINE: [ pcmk-1 ]

 VIP    (ocf::heartbeat:IPaddr2):       Started pcmk-2
```

此時會發現 VIP 的流量已經轉到 pcmk-2 來處理 & 回應。

<font color='red'>**【註】**</font>由於之前在 corosync 中的 quorum 的設定為 2(最少需要 2 nodes 進行 DC 的票選)，因此當 node 1 無法正常時，整個 cluster 中的 node 數量將會低於 2(剩下 1)，因此無法選出新的 DC 而產生問題。但由於先前已經將設定 no-quorum-policy 設定為 ignore，因此這邊不會發生錯誤。

#### (3) 確認 VIP 是否可以正常運作

回到外部的 client，觀察 ping 指令的工作狀況，並沒有斷線的情況發生，一切都很穩定。

#### (4) 從 node 2 重新啟動 node1 的 corosync & pacemake 服務

``` bash
$ ssh pcmk-1 "sudo service corosync start"
$ ssh pcmk-1 "sudo service pacemaker start"
```

接著確認 VIP 目前的服務狀況：

``` bash
$ sudo crm status
Last updated: Wed May 20 02:08:17 2015
Last change: Wed May 20 01:33:01 2015 via cibadmin on pcmk-2
Stack: corosync
Current DC: pcmk-2 (3232266854) - partition with quorum
Version: 1.1.10-42f2063
2 Nodes configured
1 Resources configured


Online: [ pcmk-1 pcmk-2 ]

 VIP    (ocf::heartbeat:IPaddr2):       Started pcmk-1
```

可以看出已經重新改由 node1 來處理 VIP 的流量了。


### 3. 移除 resource

若要移除 resource 要怎麼做呢? 直接移除是沒辦法的，因此必須先停止 resource 後再移除：

``` bash
# 停止 resource
$ sudo crm resource stop VIP

# 移除 resource
$ sudo crm configure delete VIP
```

檢查目前 cluster 狀態：

``` bash
$ sudo crm configure show
node $id="3232266853" pcmk-1 \
        attributes standby="off"
node $id="3232266854" pcmk-2
property $id="cib-bootstrap-options" \
        dc-version="1.1.10-42f2063" \
        cluster-infrastructure="corosync" \
        stonith-enabled="false" \
        no-quorum-policy="ignore"
```

已經沒有看到 resource 的資訊囉!


---------------------------------


同時納入 NFS / nginx / VIP 進行可用性測試
=========================================

### 1. 設定 cluster resource

由於之前的步驟已經將 resource VIP(**ocf:heartbeat:IPaddr2**) 移除了，因此我們這邊要重新設定 VIP，並額外針對 NFS(**ocf:heartbeat:Filesystem**) 與 nginx(**ocf:heartbeat:nginx**) 兩個服務加入監控：

``` bash
# 設定 VIP(ocf:heartbeat:IPaddr2)
$ sudo crm configure primitive VIP ocf:heartbeat:IPaddr2 params ip=192.168.122.200 nic=eth1 op monitor interval=10s timeout=20s on-fail=restart

# 設定 NFS(ocf:heartbeat:Filesystem)
$ sudo crm configure  primitive NFS_STORE ocf:heartbeat:Filesystem params device="192.168.122.10:/var/nfs" directory="/usr/share/nginx/html" fstype="nfs" op monitor interval=60s timeout=40s op start timeout=60s op stop timeout=60s

# 設定 nginx(ocf:heartbeat:nginx)
$ sudo crm configure primitive NGINX_WEB_SITE ocf:heartbeat:nginx params status10url="http://192.168.122.200" op monitor interval=10s timeout=30s op start interval=0 timeout=40s op stop interval=0 timeout=60s
```

接著，為了完成 Active/Passive 的架構，我們要將 service 集中到某一個 node 上去(Active node)，因此我們將上面三個 resource 設為同一個群組，並設定 resource 的啟動順序：

``` bash
# 設定群組
$ sudo crm configure group WEB_CLUSTER VIP NFS_STORE NGINX_WEB_SITE

# 設定 resource 啟動順序
$ sudo crm configure order WEB_CLUSTER_SERVICE_ORDER mandatory: VIP NFS_STORE NGINX_WEB_SITE
```

接著來看看目前 cluster 的設定狀況：

``` bash
$ sudo crm configure show
node $id="3232266853" pcmk-1 \
        attributes standby="off"
node $id="3232266854" pcmk-2
primitive NFS_STORE ocf:heartbeat:Filesystem \
        params device="192.168.122.10:/var/nfs" directory="/usr/share/nginx/html" fstype="nfs" \
        op monitor interval="60s" timeout="40s" \
        op start timeout="60s" interval="0" \
        op stop timeout="60s" interval="0"
primitive NGINX_WEB_SITE ocf:heartbeat:nginx \
        params status10url="http://192.168.122.200" \
        op monitor interval="10s" timeout="30s" \
        op start interval="0" timeout="40s" \
        op stop interval="0" timeout="60s"
primitive VIP ocf:heartbeat:IPaddr2 \
        params ip="192.168.122.200" nic="eth1" \
        op monitor interval="10s" timeout="20s" on-fail="restart"
group WEB_CLUSTER VIP NFS_STORE NGINX_WEB_SITE
order WEB_CLUSTER_SERVICE_ORDER inf: VIP NFS_STORE NGINX_WEB_SITE
property $id="cib-bootstrap-options" \
        dc-version="1.1.10-42f2063" \
        cluster-infrastructure="corosync" \
        stonith-enabled="false" \
        no-quorum-policy="ignore"
        
        
$ sudo crm status
Last updated: Wed May 20 06:22:27 2015
Last change: Wed May 20 06:15:40 2015 via cibadmin on pcmk-1
Stack: corosync
Current DC: pcmk-1 (3232266853) - partition with quorum
Version: 1.1.10-42f2063
2 Nodes configured
3 Resources configured


Online: [ pcmk-1 pcmk-2 ]

 Resource Group: WEB_CLUSTER
     VIP        (ocf::heartbeat:IPaddr2):       Started pcmk-2
     NFS_STORE  (ocf::heartbeat:Filesystem):    Started pcmk-2
     NGINX_WEB_SITE     (ocf::heartbeat:nginx): Started pcmk-2
```

從上面看的出來所有服務目前都集中到 pcmk-2 這一個 node 上了。

### 2. 驗證 cluster HA 功能是否成功

首先先連到 VIP 來看看網頁是否存在：

![nginx Index Page](https://lh3.googleusercontent.com/gXFcD3xHUEgJ0oCDMFYA7dFO7Q_N0fsegtiSsTgySCk=w604-h167-no)

確認無誤，表示目前 node 2 的服務正常運作中。，因此我們在

接著為了測試 HA 功能是否正常，我們讓 node 2 進入 standby 狀態，模擬 node 2 已經掛掉的情況：

``` bash
$ sudo crm node standby
```

查詢目前 cluster 狀態，確認 node 2 正處於 standby 的狀態，並由 node 1 接手服務：

``` bash
$ sudo crm status
Last updated: Wed May 20 06:27:52 2015
Last change: Wed May 20 06:27:14 2015 via crm_attribute on pcmk-2
Stack: corosync
Current DC: pcmk-1 (3232266853) - partition with quorum
Version: 1.1.10-42f2063
2 Nodes configured
3 Resources configured


Node pcmk-2 (3232266854): standby
Online: [ pcmk-1 ]

 Resource Group: WEB_CLUSTER
     VIP        (ocf::heartbeat:IPaddr2):       Started pcmk-1
     NFS_STORE  (ocf::heartbeat:Filesystem):    Started pcmk-1
     NGINX_WEB_SITE     (ocf::heartbeat:nginx): Started pcmk-1
```

確認 VIP 網頁依然運作無誤，表示整個 Cluster HA 的功能是沒問題的喔!


---------------------------------


corosync.conf 詳細說明
======================

``` bash
# totem protocol 相關設定
totem {                                                               
    version: 2
    # cluster_name: mycluster   # 自定義 cluster 名稱
    token: 3000                 # 用來研判 processor 是否已經掛掉的時間(ms)
    token_retransmits_before_loss_const: 10     # 產生新的設定前，重複傳送 token 的次數
    join: 60                    # 等待由 membership protocol 傳來的 join message 的逾時時間(ms)
    consensus: 3600             # 重新進行 membership 設定調整的逾時時間(ms)，通常會設定為 token*1.2 
    vsftype: none               # 設定 master node 的選舉機制，有 ykd(若 cluster nodes 數量龐大會消耗太多記憶體)/none(建議值) 可選。
    max_messages: 20
    clear_node_high_bit: yes    # 應付有些 openAIS client 需要大於 0 的 node id
    secauth: off                # corosync 底層間傳遞訊息時是否加密
    threads: 0                  # 設定使用多少個執行緒來處理訊息加解密的工作，若訊息沒加密則設定為 0
    rrp_mode: none              # 共有 none(預設)/active/passive 三種選項，用來設定 redundant ring 的模式
                                # 有多個 interface 的設定時才可以使用 active or passive
    
    # downcheck: 10000          # 設定當 network interface 掛掉時，多久檢查一次是否已經恢復 
    # fail_recv_const: 2500     # 設定在新的設定重新產生前，token 可以接受幾次沒收到應該收到的訊息
              
    # merge: 200                # 設定當沒有 multicast 相關流量被送出時，執行檢查的逾時時間(ms)
    # nodeid: 0                 # 此選項僅適用 IPv6 的網路環境
    # netmtu: 1500              # 若網路環境中有設定 jumbo frame，可以調大此值
    # send_join: 0              # 送出 join message 前的等待時間(32 nodes 以下不用設定，128 nodes 設定 80)(ms)
    # seqno_unchanged_const: 30 # 設定在 merge detection 逾時開始計算之前，可以接受 token 未發生 multicast 流量的次數
    # token_retransmit: 238     # 再度傳送 token 的時間(ms)
    # transportL udp            # 傳輸協定；有 iba(若 interface 為 RoCEE or Infiniband 時設定) / udpu(Unicast) / udp(預設，Multicast) 三個選項
    # window_size: 50           # 設定每個 token 循環所可以傳送的訊息數量，以不超過 (256000/netmtu) 為主。
    
    # max_messages: 17
    # miss_count_const: 5
    # rrp_problem_count_timeout: 2000
    # rrp_problem_count_mcast_threshold: 10
    # rrp_token_expired_timeout: 47
    # rrp_autorecovery_check_timeout: 1000
    
    # 設定 cluster node 的資訊
    interface {
        ringnumber: 0           # 每個 interface 都要有不同的 ring number 作為 membership protocol 辨識之用
        bindnetaddr: 127.0.0.1  # cluster node 所綁定的 ip，設定為 Network ID
        mcastaddr: 226.94.1.1   # 指定 corosync 所使用的 multicast address
        mcastport: 5405         #  指定 corosync 所使用的 UDP port number(for mcast receives, and -1 for mcast sends)
    }
}

# AIS Availability Management Framework  service (暫時用不到)
amf {
    mode: disabled
}

# master voting 相關設定
quorum {
    provider: corosync_votequorum
    expected_votes: 2   # 執行投票選出 DC(Designated Coordinator) 的法定最低票數，若目前 cluster 中的 nodes 數量低於此值，
                        # 則此 cluster 就不票選 DC 出來，避免多個 cluster 同時存在時卻又選不出 DC 所造成的 split brain 問題
                        # 通常放棄票選 DC 後會進行 STONITH 的相關程序(例如：關掉電源)
    two_node: 1         # cluster 中只有兩個 node 時使用，預設為 0
}

# 執行 AIS 功能時所使用的身分
aisexec {
    user:   root
    group:  root
}

# Log 相關設定
logging {
    fileline: off
    to_stderr: yes
    to_logfile: no
    to_syslog: yes
    # logfile: /path/to/logfile # 當 to_logfile 為 yes 時，設定 log file 的路徑
    syslog_facility: daemon
    debug: off                  # 是否開啟 debug 模式
    # tags: none                # enter|leave|trace1|trace2|trace3|... 
    timestamp: on               # 在 Log 訊息中是否放入 timestamp 資訊
    logger_subsys {
        subsys: AMF
        debug: off
        tags: enter|leave|trace1|trace2|trace3|trace4|trace6
        # logfile_priority: info        # alert / crit / debug / emerg / err / info / notice / warning
        # syslog_facility: daemon       # daemon / local0 / local1 / local2 / local3 / local4 / local5 / local6 / local7
        # syslog_priority: info         # alert / crit / debug / emerg / err / info / notice / warning
    }
}
```


---------------------------------


crm 常用指令
============

``` bash
# 查詢 crm 的使用方式
$ sudo crm help

# 查詢 crm onfigure 的使用方式
$ sudo crm configure help

# 查詢 crm configure colocation 的使用方式
$ sudo crm configure colocation help

# 查詢 crm configure rsc_template 的使用方式
$ sudo crm configure rsc_template help

# 目前 cluster 狀態(nodes / resources)
$sudo crm status

# 設定 configure > property > stonith-enable 為 false
$ sudo crm configure property stonith-enabled=false

# 列出 RA(Resource Agent) 支援的 provider 類別
$ sudo crm ra classes

# 列出 RA(Resource Agent) > list > stonith 支援的 provider 清單
$ sudo crm ra list stonith

# 查詢 provier 詳細資料 & 使用方式
$ sudo crm ra info ocf:heartbeat:IPaddr2
```


---------------------------------


參考資料
========

- [Pacemaker - Clusters from Scratch(2015/03/05)](http://clusterlabs.org/doc/en-US/Pacemaker/1.1-pcs/html/Clusters_from_Scratch/index.html)

- [Pacemaker - Configuration Explained(2015/03/05)](http://clusterlabs.org/doc/en-US/Pacemaker/1.1-pcs/html/Pacemaker_Explained/index.html)

- [corosync+pacemaker实现web集群高可用 - nmshuishui的博客 - 51CTO技术博客](http://nmshuishui.blog.51cto.com/1850554/1399811)

- [linuxlasse.net :: Pacemaker and Corosync HA - 2 Node setup](http://linuxlasse.net/linux/howtos/Pacemaker_and_Corosync_HA_-_2_Node_setup)

- [Linux Cluster Part 1 - Install Corosync and Pacemaker on CentOS 6 - GeekPeek.Net](http://geekpeek.net/linux-cluster-corosync-pacemaker/)

- [Linux Cluster Part 2 - Adding and Deleting Cluster Resources - GeekPeek.Net](http://geekpeek.net/linux-cluster-resources/)

- [high availability - corosync + pacemaker + nginx - Server Fault](http://serverfault.com/questions/485315/corosync-pacemaker-nginx)

- [Resource Agents - Linux-HA](http://www.linux-ha.org/wiki/Resource_Agents)

- [Resource Agents - ocf_heartbeat_nginx](http://linux-ha.org/doc/man-pages/re-ra-nginx.html)

- [Pacemaker: fix timeout warnings « ID's blog](http://zeldor.biz/2011/03/pacemaker-fix-timeout-warnings/)

- [Suse Doc: 高可用性指南 - 設定叢集資源](https://www.suse.com/zh-tw/documentation/sle_ha/book_sleha/data/sec_ha_config_crm_resources.html)


### corosync.conf 相關

- [Ubuntu Manpage: corosync.conf - corosync executive configuration file](http://manpages.ubuntu.com/manpages/trusty/man5/corosync.conf.5.html)

- [Ubuntu Manpage: votequorum - Votequorum Configuration Overview](http://manpages.ubuntu.com/manpages/trusty/man5/votequorum.5.html)

- [Ubuntu Manpage: amf.conf - corosync AMF configuration file](http://manpages.ubuntu.com/manpages/precise/en/man5/amf.conf.5.html)
