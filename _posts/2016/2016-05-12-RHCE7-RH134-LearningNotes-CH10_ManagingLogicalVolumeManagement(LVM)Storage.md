---
layout: post
title:  "[RHCE7] RH134 Chapter 10. Managing Logical Volume Management(LVM) Storage 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 10. Managing Logical Volume Management(LVM) Storage 留下的內容"
date: 2016-05-12 04:45:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH134]
---

10.1 Logical Volume Management Concepts
=======================================

可以從下圖很清楚看出 Physical Drivers / Partitions / PV(Physical Volume) / VG(Volume Group) / LV(Logical Volume) 的相互關係

![LVM](http://mo.morsi.org/blog/files/lvm1.png)

此外，還有 PE(Physical Extent) & LE(Logical Extent) 與上述其他概念的組成關係：

![PE_LE_1](http://s3.51cto.com/wyfs02/M00/72/35/wKioL1XetQ7wtmzfAAJ6MCQFC90143.jpg)

![PE_LE_2](http://www.linux-tips-and-tricks.de/images/stories/lvm.jpg)

----------------------------------------------------------------

10.2 Managing Logical Volumes
=============================

## 10.2.1 Create PV

1. 在 /dev/sda 中建立一個 partition，使用所有空間，partition type = 8e

2. 第一個硬碟建立 PV：`sudo pvcreate /dev/sda1`

3. 第二個硬碟建立 PV：`sudo pvcreate /dev/sdb`(沒有 partition 的狀況下)

### 其他相關指令

- `pvscan`：目前有哪些 PV

- `pvdisplay [pv_name]`：顯示 PV 的詳細資訊


## 10.2.2 Create VG

1. 建立 Volume Group：`sudo vgcreate -s 16M vg0 /dev/sda1 /dev/sdb` (**<font color='red'>設定 PE 大小為 16M</font>**)
> 相同 VG 中的 PE 都一樣大

2. 檢查 PV：`sudo pvscan` (顯示 PV 被使用中)

3. 檢查 VG：`sudo vgscan`

4. 檢視 VG 詳細資料：`sudo vgdisplay [vg_name]`

## 10.2.3 Create LV

1. 建立 LV(從 vg0 切割 200M 空間，給 LV 使用，並命名為 lv-1)：`sudo lvcreate -L 200M -n lv-1 vg0`

2. 建立 LV(使用 PE 個數指定大小)：`sudo lvcreate -l 13 -n lv-2 vg0`

3. 檢查 LV：`sudo lvscan`

4. 檢視 LV 詳細資料：`sudo lvdisplay /dev/vg0/lv-1` & `sudo lvdisplay /dev/vg0/lv-2`

> 要使用 LV 前要記得進行 format 的動作

----------------------------------------------------------------

10.3 Extending Logical Volumes
==============================

## 10.3.1 Extending and reducing a volume group

1. 建立新的 PV：`sudo pvcreate /dev/sdc`

2. 將新的 PV(/dev/sdc) 加到 VG 中：`sudo vgextend vg0 /dev/sdc`

> 若要把 PV 移出 VG 可以使用類似的指令：`sudo vgreduce vg0 /dev/sda1`

## 10.3.2 Extend a logical volume and XFS file system

目標：**將 LV 容量加大到 300M**

1. LV 放大可以 online，縮小需要 offline：`sudo lvextend -L +100M /dev/vg0/lv-1` or `sudo lvextend -L 300M /dev/vg0/lv-1`

2. 也可以用指定 PE 的個數來放大：`sudo lvextend -l +7 /dev/vg0/lv-2` or `sudo lvextend -l 19 /dev/vg0/lv-2`

3. 檢查 LV 狀態：`sudo lvscan`

4. 放大 XFS 檔案系統：`sudo xfs_growfs /dev/vg0/lv-1`

5. 放大 EXT4 檔案系統：`sudo resize2fs /dev/vg0/lv-2`

### 如何縮小 LV

**<font color='red'>【註】</font>** XFS 檔案系統僅能放大，不能縮小(上面的 lv-1 已經無法縮小)

目標：縮小 EXT4 到 200M

1. umount EXT4 partition：`sudo umount /lv-2`

2. 檢查 LV：`sudo fsck -f /dev/vg0/lv-2`

3. 縮小 EXT4 檔案系統：`sudo resize2fs /dev/vg0/lv-2 200M`

4. 縮小 LV：`sudo lvreduce -L 200M /dev/vg0/lv-2`

5. 重新掛載磁區：`sudo mount -a`

### 如何縮小 VG & 移除 PV(硬碟)

目標：移除第一個 PV(/dev/sda1)

1. 移動指定 PV 中的檔案到其他 PV 上：`pvscan` -> `sudo pvmove /dev/sda1 /dev/sdc`(也可以不指定目的裝置)

2. 移除 VG 中的 PV：`sudo vgreduce vg0 /dev/sda1` -> `pvscan`

3. 徹底移除 PV：`sudo pvremove /dev/sda1`

最後就可以把電腦關機並移除硬碟 /dev/sda 囉!


### 移除 LV

目標：**移除 /dev/vg0/lv-1**

1. 卸載檔案系統：`sudo umount /lv-1`
2. 移除 LV：`sudo lvremove /dev/vg0/lv-1` -> `sudo lvscan`

----------------------------------------------------------------

10.4 Snapshot (補充)
====================

假設：有個 20T 的 MySQL 資料庫，需要備份

1. 停掉 MySQL 服務

2. 建立 Snapshop：`sudo lvcreate -L 100M -s -n backup /dev/vg0/lv-2` -> 'sudo lvscan'

3. 啟動 MySQL 服務

4. 檢查 LV 之間的關聯：`sudo lvs`

> 只有在 PE 中的資料異動前，才會有資料複製的動作發生

## 老師設計的實驗

1. reset server

2. 幫 server 加上 200M/300M/500M IDE 硬碟共三顆

3. 開機，按 F12 選擇 virtio(4) 開機
