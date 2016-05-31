---
layout: post
title:  "[RHCE7] RH254 Chapter 7 Providing Remote Block Storage Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 7 Providing Remote Block Storage 留下的內容"
date: 2016-05-31 11:40:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH254]
---

7.1 iSCSI Concepts
==================

詳細的 iSCSI 介紹可參考 => [鳥哥的 Linux 私房菜 -- 網路磁碟裝置：iSCSI伺服器](http://linux.vbird.org/linux_server/0460iscsi.php)

-----------------------------------------------

7.2 Providing iSCSI Targets
===========================

## 7.2.1 iSCSI target overview

- iSCSI target 以 LUN 的方式提供外部存取

- LUN 可以以 raw 方式存取，或是格式化成 client 支援的檔案系統

- 通常 LUN 同時間只能給一個 client 存取，除非是使用像是 **GFS2** 這種 cluster-capable 的檔案系統才可以同時間有多個 client 存取

- iSCSI target 還可以透過 ACL 方式限制 LUN 的存取

## 7.2.2 iSCSI target configuration

首先先來個前置作業，先分別建立一個 LVM logic volume(`/dev/iSCSI_vg/disk1_lv`) & partition(`/dev/vdb1`)：

```bash
$ sudo lvscan
  ACTIVE            '/dev/iSCSI_vg/disk1_lv' [100.00 MiB] inherit

$ sudo fdisk -l /dev/vdb
.......
   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048     2099199     1048576   83  Linux
/dev/vdb2         2099200     4196351     1048576   8e  Linux LVM
```


RHEL7 中提供了 `targetcli` 套件工具來協助設定 iSCSI target，除了支援 **cd**, **ls**, **pwd**, **set** 等一般指另外，還支援 **TAB completion**

```bash
$ sudo yum -y install targetcli
```

建立 block device：

```bash
# 進入互動模式
$ sudo targetcli
Warning: Could not load preferences file /root/.targetcli/prefs.bin.
targetcli shell version 2.1.fb34
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/> ls
o- / ..................... [...]
  o- backstores .......... [...]
  | o- block ............. [Storage Objects: 0] # 在 server 上定義好的 block device
  | o- fileio ............ [Storage Objects: 0] # 特定檔案(類似 disk image 的概念)
  | o- pscsi ............. [Storage Objects: 0] # 實體的 SCSI 裝置(可透過 passthrough 的模式使用)
  | o- ramdisk ........... [Storage Objects: 0] # 使用 ram 空間作為存取裝置(斷電資料就沒了)
  o- iscsi ............... [Targets: 0]
  o- loopback ............ [Targets: 0]
/>
/> cd backstores/
# 建立 block
/backstores> block/ create block1 /dev/iSCSI_vg/disk1_lv
Created block storage object block1 using /dev/iSCSI_vg/disk1_lv.
# 建立 block
/backstores> block/ create block2 /dev/vdb1
Created block storage object block2 using /dev/vdb1.
# 建立 file
/backstores> fileio/ create file1 /root/disk1_file 100M
Created fileio file1 with size 104857600
/backstores> ls
o- backstores ..................................................... [...]
  o- block ........................................................ [Storage Objects: 2]
  | o- block1 ..................................................... [/dev/iSCSI_vg/disk1_lv (100.0MiB) write-thru deactivated]
  | o- block2 ..................................................... [/dev/vdb1 (1.0GiB) write-thru deactivated]
  o- fileio ....................................................... [Storage Objects: 1]
  | o- file1 ...................................................... [/root/disk1_file (100.0MiB) write-back deactivated]
  o- pscsi ........................................................ [Storage Objects: 0]
  o- ramdisk ...................................................... [Storage Objects: 0]
```

設定 IQN：(**基本上 IQN 可以自訂為有意義的名稱，但一般都會戴上自己所屬的 domain name**)

```bash
/> cd /iscsi
# 建立 IQN
/iscsi> create iqn.2016-05.com.example.remotedisk1
Created target iqn.2016-05.com.example.remotedisk1.
Created TPG 1.  # 會自動建立預設的 TPG
/iscsi> ls
o- iscsi ................................................... [Targets: 1]
  o- iqn.2016-05.com.example.remotedisk1 ................... [TPGs: 1]
    o- tpg1 ................................................ [no-gen-acls, no-auth]
      o- acls .............................................. [ACLs: 0]
      o- luns .............................................. [LUNs: 0]
      o- portals ........................................... [Portals: 0]
```

設定 ACL 規則：(僅允許 `iqn.2016-05.com.example:desktop0` 連入)

```bash
/iscsi> cd iqn.2016-05.com.example.remotedisk1/tpg1
# 建立新的 ACL 規則
/iscsi/iqn.20...otedisk1/tpg1> acls/ create iqn.2016-05.com.example:desktop0
Created Node ACL for iqn.2016-05.com.example:desktop0
/iscsi/iqn.20...otedisk1/tpg1> ls
o- tpg1 .............................................. [no-gen-acls, no-auth]
  o- acls ............................................ [ACLs: 1]
  | o- iqn.2016-05.com.example:desktop0 .............. [Mapped LUNs: 0]
  o- luns ............................................ [LUNs: 0]
  o- portals ......................................... [Portals: 0]
```

使用一開始建立好的三個 block device，建立 LUN：

```bash
/iscsi/iqn.20...otedisk1/tpg1> luns/ create /backstores/block/block1
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2016-05.com.example:desktop0
/iscsi/iqn.20...otedisk1/tpg1> luns/ create /backstores/block/block2
Created LUN 1.
Created LUN 1->1 mapping in node ACL iqn.2016-05.com.example:desktop0
/iscsi/iqn.20...otedisk1/tpg1> luns/ create /backstores/fileio/file1
Created LUN 2.
Created LUN 2->2 mapping in node ACL iqn.2016-05.com.example:desktop0
/iscsi/iqn.20...otedisk1/tpg1> ls
o- tpg1 ....................................................... [no-gen-acls, no-auth]
  o- acls ..................................................... [ACLs: 1]
  | o- iqn.2016-05.com.example:desktop0 ....................... [Mapped LUNs: 3]
  |   o- mapped_lun0 .......................................... [lun0 block/block1 (rw)]
  |   o- mapped_lun1 .......................................... [lun1 block/block2 (rw)]
  |   o- mapped_lun2 .......................................... [lun2 fileio/file1 (rw)]
  o- luns ..................................................... [LUNs: 3]
  | o- lun0 ................................................... [block/block1 (/dev/iSCSI_vg/disk1_lv)]
  | o- lun1 ................................................... [block/block2 (/dev/vdb1)]
  | o- lun2 ................................................... [fileio/file1 (/root/disk1_file)]
  o- portals .................................................. [Portals: 0]
```

設定監聽用 portal：(不須特別指定 port，預設會使用 `iSCSI 3260`)

```bash
/iscsi/iqn.20...otedisk1/tpg1> portals/ create 172.25.0.11
Using default IP port 3260
Created network portal 172.25.0.11:3260.
/iscsi/iqn.20...otedisk1/tpg1> ls
o- tpg1 .......................................................... [no-gen-acls, no-auth]
  o- acls ........................................................ [ACLs: 1]
  | o- iqn.2016-05.com.example:desktop0 .......................... [Mapped LUNs: 3]
  |   o- mapped_lun0 ............................................. [lun0 block/block1 (rw)]
  |   o- mapped_lun1 ............................................. [lun1 block/block2 (rw)]
  |   o- mapped_lun2 ............................................. [lun2 fileio/file1 (rw)]
  o- luns ........................................................ [LUNs: 3]
  | o- lun0 ...................................................... [block/block1 (/dev/iSCSI_vg/disk1_lv)]
  | o- lun1 ...................................................... [block/block2 (/dev/vdb1)]
  | o- lun2 ...................................................... [fileio/file1 (/root/disk1_file)]
  o- portals ..................................................... [Portals: 1]
    o- 172.25.0.11:3260 .......................................... [OK]
```

> 若沒指定 portal，則會使用 `0.0.0.0` 作為 portal，這表示可以透過 iSCSI target 上所有的網路介面進行存取

最後檢視所有設定：

```bash
/iscsi/iqn.20...otedisk1/tpg1> cd /
/> ls
o- / ......................................................... [...]
  o- backstores .............................................. [...]
  | o- block ................................................. [Storage Objects: 2]
  | | o- block1 .............................................. [/dev/iSCSI_vg/disk1_lv (100.0MiB) write-thru activated]
  | | o- block2 .............................................. [/dev/vdb1 (1.0GiB) write-thru activated]
  | o- fileio ................................................ [Storage Objects: 1]
  | | o- file1 ............................................... [/root/disk1_file (100.0MiB) write-back activated]
  | o- pscsi ................................................. [Storage Objects: 0]
  | o- ramdisk ............................................... [Storage Objects: 0]
  o- iscsi ................................................... [Targets: 1]
  | o- iqn.2016-05.com.example.remotedisk1 ................... [TPGs: 1]
  |   o- tpg1 ................................................ [no-gen-acls, no-auth]
  |     o- acls .............................................. [ACLs: 1]
  |     | o- iqn.2016-05.com.example:desktop0 ................ [Mapped LUNs: 3]
  |     |   o- mapped_lun0 ................................... [lun0 block/block1 (rw)]
  |     |   o- mapped_lun1 ................................... [lun1 block/block2 (rw)]
  |     |   o- mapped_lun2 ................................... [lun2 fileio/file1 (rw)]
  |     o- luns .............................................. [LUNs: 3]
  |     | o- lun0 ............................................ [block/block1 (/dev/iSCSI_vg/disk1_lv)]
  |     | o- lun1 ............................................ [block/block2 (/dev/vdb1)]
  |     | o- lun2 ............................................ [fileio/file1 (/root/disk1_file)]
  |     o- portals ........................................... [Portals: 1]
  |       o- 172.25.0.11:3260 ................................ [OK]
  o- loopback ................................................ [Targets: 0]
/> exit
Global pref auto_save_on_exit=true
Last 10 configs saved in /etc/target/backup.
Configuration saved to /etc/target/saveconfig.json
```

最後要開啟防火牆 & 啟動 iSCSI target 服務：

```bash
# 開啟防火牆
$ sudo firewall-cmd --permanent --add-port=3260/tcp
$ sudo firewall-cmd --reload

# 啟動 target.service
$ sudo systemctl start target.service
$ sudo systemctl status target.service
```

### Authentication

除了 ACL 之外，還可以設定密碼認證，但因為 iSCSI CHAP 所能提供的安全性並不是很高，因此建議還是從網路規劃 & 防火牆設定來著手

設定僅有指定來源才可以存取 iSCSI target，或是將 storage network 規劃成一個獨立私有的 VLAN 來使用

### Command-line mode

targetcli 除了互動模式外，也可以用 command line 模式來設定，例如：

```bash
$ sudo targetcli /backstores/block create block1 /dev/iSCSI_vg/disk1_lv

$ sudo targetcli /iscsi create iqn.2016-05.com.example.remotedisk1

$ sudo targetcli /iscsi/iqn.2016-05.com.example.remotedisk1/tpg1/portals create 172.25.0.11

$ sudo targetcli saveconfig
```

-----------------------------------------------

Practice: Providing iSCSI Targets
=================================

## 目標

1. 建立一個 200MB 的 LVM logical volume，完整路徑為 `/dev/iscsi_vg/disk1_lv`

2. 設定 iSCSI Target 並使用上面的 lv 作為 LUN

## 實作步驟

我們將實作部分分成幾個部分：

1. 分割硬碟

2. 設定 LVM

3. 設定 iSCSI target
  1. 建立 `backstores`
  2. 建立 `IQN` in `iscsi`
  3. 建立 `ACL` in `TPG`
  4. 建立 `LUN` in `TPG`
  5. 建立 `portal` in `TPG`

4. 開啟防火牆

5. 啟動 iSCSI target

### 1、分割硬碟

```bash
$ sudo fdisk -l /dev/vdb
....
   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048     2099199     1048576   8e  Linux LVM
```

### 2、設定 LVM

```bash
$ sudo pvcreate /dev/vdb1
  Physical volume "/dev/vdb1" successfully created

$ sudo vgcreate iscsi_vg /dev/vdb1
  Volume group "iscsi_vg" successfully created

$ sudo lvcreate -n disk1_lv -L 200M iscsi_vg
  Logical volume "disk1_lv" created

$ sudo lvscan
  ACTIVE            '/dev/iscsi_vg/disk1_lv' [200.00 MiB] inherit
```

安裝 targetcli 並設定 iSCSI target：

```bash
$ yum -y install targetcli
```

### 3、設定 iSCSI target

```bash
$ sudo targetcli
/> ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 0]
  o- loopback ......................................................................................................... [Targets: 0]
/> /backstores/block create server.disk1 /dev/iscsi_vg/disk1_lv
Created block storage object server.disk1 using /dev/iscsi_vg/disk1_lv.
/> /iscsi create iqn.2016-05.com.example.com.remotedisk1
Created target iqn.2016-05.com.example.com.remotedisk1.
Created TPG 1.
/> /iscsi/iqn.2016-05.com.example.com.remotedisk1/tpg1/acls create iqn.2016-05.com.example.desktop0
Created Node ACL for iqn.2016-05.com.example.desktop0
/> /iscsi/iqn.2016-05.com.example.com.remotedisk1/tpg1/luns create /backstores/block/server.disk1
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2016-05.com.example.desktop0
/> /iscsi/iqn.2016-05.com.example.com.remotedisk1/tpg1/portals create 172.25.0.11
Using default IP port 3260
Created network portal 172.25.0.11:3260.
/> ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 1]
  | | o- server.disk1 ..................................................... [/dev/iscsi_vg/disk1_lv (200.0MiB) write-thru activated]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2016-05.com.example.com.remotedisk1 ........................................................................... [TPGs: 1]
  |   o- tpg1 ............................................................................................... [no-gen-acls, no-auth]
  |     o- acls .......................................................................................................... [ACLs: 1]
  |     | o- iqn.2016-05.com.example.desktop0 ..................................................................... [Mapped LUNs: 1]
  |     |   o- mapped_lun0 .......................................................................... [lun0 block/server.disk1 (rw)]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 .................................................................... [block/server.disk1 (/dev/iscsi_vg/disk1_lv)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 172.25.0.11:3260 ................................................................................................. [OK]
  o- loopback ......................................................................................................... [Targets: 0]
/> exit
Global pref auto_save_on_exit=true
Last 10 configs saved in /etc/target/backup.
Configuration saved to /etc/target/saveconfig.json
```

### 4、開啟防火牆

```bash
$ sudo firewall-cmd --permanent --add-port=3260/tcp
$ sudo firewall-cmd --reload
```

### 5、啟動 iSCSI target

```bash
$ sudo systemctl enable target.service
$ sudo systemctl start target.service

$ sudo systemctl status target.service
target.service - Restore LIO kernel target configuration
   Loaded: loaded (/usr/lib/systemd/system/target.service; enabled)
   Active: active (exited) since Sun 2016-05-29 21:56:20 JST; 4s ago
  Process: 2713 ExecStart=/usr/bin/targetctl restore (code=exited, status=0/SUCCESS)
 Main PID: 2713 (code=exited, status=0/SUCCESS)

May 29 21:56:20 server0.example.com systemd[1]: Started Restore LIO kernel target configuration.
```

-----------------------------------------------

7.3 Accessing iSCSI Storage
===========================

iSCSI initiator 可用軟體模擬，也可以是硬體。

在 RHEL7 中，安裝了 `iscsi-initiator-utils` 套件後，就會有 `iscsi.service` & `iscsid.service` 兩個 system unit，並搭配 `/etc/iscsi/iscsid.conf` & `/etc/iscsi/initiatorname.iscsi` 兩個設定檔：

- `/etc/iscsi/initiatorname.iscsi`：檔案內容為 iSCSI initiator 的 IQN 字串

- `/etc/iscsi/iscsid.conf`：進行 new target discovery 時的設定，例如：iSCSI timeout、retry 次數、認證帳號密碼...等等。(修改此檔案需要重新啟動 iscsi.service)

在 iSCSI initiator 要連接 iSCSI target 使用之前，必須先被 discovered：

```bash
$ sudo iscsiadm -m discovery -t sendtargets -p 172.25.0.11
172.25.0.11:3260,1 iqn.2016-05.com.example.com.remotedisk1

# 被 discovery 到的 iSCSI targe 會記錄在 /var/lib/iscsi/nodes/ 目錄中
$ sudo cat /var/lib/iscsi/nodes/iqn.2016-05.com.example.com.remotedisk1/172.25.0.11,3260,1/default
# BEGIN RECORD 6.2.0.873-21
node.name = iqn.2016-05.com.example.com.remotedisk1
node.tpgt = 1
node.startup = automatic
node.leading_login = No
iface.iscsi_ifacename = default
iface.transport_name = tcp
......
node.discovery_address = 172.25.0.11
node.discovery_port = 3260
......
# END RECORD
```

接著要設定 iSCSI initiator IQN：

```bash
$ echo "InitiatorName=iqn.2016-05.com.example.desktop0" | sudo tee /etc/iscsi/initiatorname.iscsi

# 重新啟動 iscsi.service
$ sudo systemctl enable iscsi.service
$ sudo systemctl restart iscsi.service
```

與 iSCSI target 互動：

```bash
# 登入 iSCSI target
$ sudo iscsiadm -m node -T iqn.2016-05.com.example.com.remotedisk1 -l

# 查詢目前與 iSCSI target 連線中的 session 資訊
$ sudo iscsiadm -m session
tcp: [2] 172.25.0.11:3260,1 iqn.2016-05.com.example.com.remotedisk1 (non-flash)

# 登出 iSCSI target 連線 session
$ sudo iscsiadm -m node -T iqn.2016-05.com.example.com.remotedisk1 -u
Logging out of session [sid: 2, target: iqn.2016-05.com.example.com.remotedisk1, portal: 172.25.0.11,3260]
Logout of [sid: 2, target: iqn.2016-05.com.example.com.remotedisk1, portal: 172.25.0.11,3260] successful.

# 刪除 iSCSI target 資訊
$ sudo iscsiadm -m node -T iqn.2016-05.com.example.com.remotedisk1 -o delete
$ sudo iscsiadm -m node -T iqn.2016-05.com.example.com.remotedisk1
iscsiadm: No records found
```

-----------------------------------------------

Practice: Accessing iSCSI Storage
=================================

## 目標

1. discover 並連接遠端的 iSCSI target & LUN

2. 相關參數如下：
  - IP: `172.25.0.11`
  - IQN: `iqn.2016-05.com.example.remotedisk1`
  - IQN ACL: `iqn.2016-05.com.example.desktop0`

## 實作過程

```bash
# 安裝 iscsi-initiator-utils 套件
$ sudo yum -y install iscsi-initiator-utils

# 修改 initiator IQN
$ echo "InitiatorName=iqn.2016-05.com.example.desktop0" | sudo tee /etc/iscsi/initiatorname.iscsi

# discovery
$ sudo iscsiadm -m discovery -t st -p 172.25.0.11
172.25.0.11:3260,1 iqn.2016-05.com.example.remotedisk1

# login 到遠端的 iSCSI target
$ sudo iscsiadm -m node -T iqn.2016-05.com.example.remotedisk1 -l
Logging in to [iface: default, target: iqn.2016-05.com.example.remotedisk1, portal: 172.25.0.11,3260] (multiple)
Login to [iface: default, target: iqn.2016-05.com.example.remotedisk1, portal: 172.25.0.11,3260] successful.

# 查詢現有的 partition，多一個 sda
$ sudo lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  200M  0 disk
vda    253:0    0   10G  0 disk
└─vda1 253:1    0   10G  0 part /
vdb    253:16   0   10G  0 disk

# 檢視相關 log
$ sudo tail /var/log/messages
May 30 16:06:44 localhost kernel: scsi 2:0:0:0: alua: port group 00 state A non-preferred supports TOlUSNA
May 30 16:06:44 localhost kernel: scsi 2:0:0:0: alua: Attached
May 30 16:06:44 localhost kernel: scsi 2:0:0:0: Attached scsi generic sg0 type 0
May 30 16:06:44 localhost kernel: sd 2:0:0:0: [sda] 409600 512-byte logical blocks: (209 MB/200 MiB)
May 30 16:06:44 localhost kernel: sd 2:0:0:0: [sda] Write Protect is off
May 30 16:06:44 localhost kernel: sd 2:0:0:0: [sda] Write cache: enabled, read cache: enabled, supports DPO and FUA
May 30 16:06:44 localhost kernel: sda: unknown partition table
May 30 16:06:44 localhost kernel: sd 2:0:0:0: [sda] Attached SCSI disk
May 30 16:06:44 localhost iscsid: Could not set session1 priority. READ/WRITE throughout and latency could be affected.
May 30 16:06:44 localhost iscsid: Connection1:0 to [target: iqn.2016-05.com.example.remotedisk1, portal: 172.25.0.11,3260] through [iface: default] is operational now

# 檢視 target 詳細資訊
$ sudo iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 6.2.0.873-21
Target: iqn.2016-05.com.example.remotedisk1 (non-flash)
        Current Portal: 172.25.0.11:3260,1
        Persistent Portal: 172.25.0.11:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2016-05.com.example.desktop0
                Iface IPaddress: 172.25.0.10
                Iface HWaddress: <empty>
                Iface Netdev: <empty>
                SID: 1
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 2  State: running
                scsi2 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sda          State: running

$ cd /var/lib/iscsi/nodes
$ sudo ls -lR
.:
total 0
drw-------. 3 root root 31 May 31 05:04 iqn.2016-05.com.example.remotedisk1

./iqn.2016-05.com.example.remotedisk1:
total 0
drw-------. 2 root root 20 May 31 05:04 172.25.0.11,3260,1

./iqn.2016-05.com.example.remotedisk1/172.25.0.11,3260,1:
total 4
-rw-------. 1 root root 2043 May 31 05:04 default

# logout iSCSI target
$ sudo iscsiadm -m node -T iqn.2016-05.com.example.remotedisk1 -u
Logging out of session [sid: 1, target: iqn.2016-05.com.example.remotedisk1, portal: 172.25.0.11,3260]
Logout of [sid: 1, target: iqn.2016-05.com.example.remotedisk1, portal: 172.25.0.11,3260] successful.

# block device 已經消失
$ sudo lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    253:0    0  10G  0 disk
└─vda1 253:1    0  10G  0 part /
vdb    253:16   0  10G  0 disk

# delete iSCSI target 資訊，因此 /var/lib/iscsi/nodes 目錄中也不會有相關資料了
$ sudo iscsiadm -m node -T iqn.2016-05.com.example.remotedisk1 -o delete
$ sudo ls -lR
.:
total 0

# 重新啟動 iscsi.service
$ sudo systemctl restart iscsi.service
```

> 若要設定在 `/etc/fstab`，可加入設定 `/dev/sda  /mnt  ext4  -netdev  0 0` (使用 `-netdev` 會確保網路通了之後才會掛載 remote disk)

-----------------------------------------------

Lab: Providing Block-based Storage
==================================

## 目標

1. 在 server 上建立 `1GB` 的 iSCSI target，IQN 為 `iqn.2014-06.com.example.server0`

2. iSCSI initiator IQN 限定為 `iqn.2014-06.com.example.desktop0`

3. desktop 必須能夠**永久**的掛載 iSCSI target LUN 於 `/iscsidisk` 目錄，並將其格式化為 `XFS` 檔案系統

## 實作過程

### 設定 iSCSI target

建立磁碟分割、設定 LVM：

```bash
$ sudo fdisk -l /dev/vdb
.....
   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048     4196351     2097152   8e  Linux LVM

$ sudo pvcreate /dev/vdb1
  Physical volume "/dev/vdb1" successfully created

$ sudo vgcreate vg_iscsi /dev/vdb1
  Volume group "vg_iscsi" successfully created

$ sudo lvcreate -n lv_disk1 -L 1G vg_iscsi
  Logical volume "lv_disk1" created

$ sudo lvscan
  ACTIVE            '/dev/vg_iscsi/lv_disk1' [1.00 GiB] inherit
```

設定 iSCSI target：

```bash
$ sudo yum -y install targetcli

# 設定 iSCSI target
$ sudo targetcli
/> /backstores/block create server.disk1 /dev/vg_iscsi/lv_disk1
Created block storage object server.disk1 using /dev/vg_iscsi/lv_disk1.

/> /iscsi create iqn.2014-06.com.example.server0
Created target iqn.2014-06.com.example.server0.
Created TPG 1.

/> /iscsi/iqn.2014-06.com.example.server0/tpg1/acls create iqn.2014-06.com.example.desktop0
Created Node ACL for iqn.2014-06.com.example.desktop0

/> /iscsi/iqn.2014-06.com.example.server0/tpg1/luns create /backstores/block/server.disk1
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2014-06.com.example.desktop0

/> /iscsi/iqn.2014-06.com.example.server0/tpg1/portals create 172.25.0.11
Using default IP port 3260
Created network portal 172.25.0.11:3260.

/> exit
Global pref auto_save_on_exit=true
Last 10 configs saved in /etc/target/backup.
Configuration saved to /etc/target/saveconfig.json


# 設定防火牆
$ sudo firewall-cmd --permanent --add-port=3260/tcp
$ sudo firewall-cmd --reload
```


### 設定 iSCSI initiator

```bash
$ sudo yum -y install iscsi-initiator-utils

# 設定 iSCSI initiator IQN
$ echo "InitiatorName=iqn.2014-06.com.example.desktop0" | sudo tee /etc/iscsi/initiatorname.iscsi

# iSCSI target discovery
$ sudo iscsiadm -m discovery -t st -p 172.25.0.11
172.25.0.11:3260,1 iqn.2014-06.com.example.server0

# login iSCSI target
$ sudo iscsiadm -m node -T iqn.2014-06.com.example.server0 -l
Logging in to [iface: default, target: iqn.2014-06.com.example.server0, portal: 172.25.0.11,3260] (multiple)
Login to [iface: default, target: iqn.2014-06.com.example.server0, portal: 172.25.0.11,3260] successful.

$ sudo lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0   1G  0 disk
vda    253:0    0  10G  0 disk
 -vda1 253:1    0  10G  0 part /
vdb    253:16   0  10G  0 disk

# 將 remote disk 格式化為 XFS 並查詢 UUID
$ sudo mkfs.xfs /dev/sda
[student@desktop0 ~]$ sudo blkid
/dev/vda1: UUID="9bf6b9f7-92ad-441b-848e-0257cbb883d1" TYPE="xfs"
/dev/sda: UUID="fff253e8-2ee6-47e4-91d8-cd0fe521731d" TYPE="xfs"

# 設定永久掛載
# _netdev: 網路裝置，等到網路通了才會嘗試連結
# 2: 最後才進行 fsck
$ mkdir /iscsidisk
$ echo "UUID=fff253e8-2ee6-47e4-91d8-cd0fe521731d   /iscsidisk  xfs  _netdev  0 0" | sudo tee --append /etc/fstab
$ sudo mount -a
echo "Hello iSCSI remote disk" | sudo tee /iscsidisk/123

# 重開機後確認檔案是否存在
```
