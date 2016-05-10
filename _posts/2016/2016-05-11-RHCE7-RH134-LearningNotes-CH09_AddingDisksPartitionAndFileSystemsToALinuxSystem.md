---
layout: post
title:  "[RHCE7] RH134 Chapter 09. Adding Disks, Partitions, and File Systems to a Linux System 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 09. Adding Disks, Partitions, and File Systems to a Linux System 留下的內容"
date: 2016-05-10 04:45:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH134]
---


9.1 Adding Partitions, File Systems, and Persistent Mounts
===========================================================

## 9.1.1 Disk partitioning

Hard Disk 的分割結構：(從頭到尾)

1. MBR(512 bytes) = Boot Loader(446 byte) + Partition Table(64 bytes)

Partition Type：

- `Primary`：一個硬碟最多可以切 4 個 primary partition (系統要啟動必須裝載 Primary Partition)

- `Extended`：若要超過 4 個 partition，則要將 primary partition 來換成 extended partition

- `Logical`：在 extended partition 中的 partition (Logical Partition Table 存在 Extended Partition 的前面)，且**<font color='red'>代碼從 5 開始(/dev/sda5)</font>**

備份 /dev/sda1：`dd if=/dev/sda1 of=/tmp/sda1.dd`

還原 /dev/sda1：`dd if=/tmp/sda1.dd of=/dev/sda1`

### MBR Partition

- 支援 2^32 個定址空間

- 傳統硬碟最大容量為 2T：2^32 x 512 bytes / 1024 / 1024 /1024 / 1024 = 2T

- 超過 2T 的硬碟 -> Advanced Format(block size 從 512 bytes 變成 4K) = 最大可支援到 16T (過渡時期的解決方案)

- 備份 MBR：`dd if=/dev/sda of=/tmp/mbr.dd bs=512 count=1`

### GPT Partition

- 若 OS 裝在 GPT Partition 上，沒辦法在舊電腦上用(因為傳統的 int 13h 問題)；但開機後，OS level 可以認得超過 2T 的硬碟

- UEFI BIOS 上的 int 13h 改寫過，因此同時支援 MBR & GPT 兩種 partition

- 最多支援 128 個 partition，支援 2^64 個定址空間，因此最大可以支援到 8ZB(512 bytes block size)，若是使用 4K block size 則可支援到 64ZB

- GPT 為每一個 partition 提供 128 bits GUID 作為辨識之用

- GPT partition table 在硬碟的頭尾各有一個，並搭配 CRC checksum，因此有備援的機制


## 9.1.2 Managing MBR partitions with fdisk/gdisk

- 若沒有使用 `w` 儲存 partition 分割設定，就不會實際切割硬碟

```bash
# 查詢目前的 partition table
[vagrant@localhost ~]$ cat /proc/partitions
major minor  #blocks  name

   8        0   83886080 sda
   8        1     512000 sda1
   8        2   83373056 sda2
 253        0   52428800 dm-0
 253        1    1048576 dm-1
 253        2   29827072 dm-2
```

切割完 partition 後，記得執行 `sudo partprobe [device name]` 強制 kernel 重新讀取最新的 partition table

## 9.1.4 Creating file systems

硬碟分割完後需要進行格式化，一般會使用 `mkfs` 指令：

`sudo mkfs -t xfs /dev/vdb1`：格式化成 XFS，也可以寫成 `sudo mkfs.xfs /dev/vdb1`

## 9.1.4 Mounting file systems

```bash
$ sudo mkdir /XFS

$ sudo mkdir /EXT4

# 掛載 XFS partition
$ sudo mount /dev/vdb1 /XFS

# 掛載 EXT4 partition
$ sudo mount /dev/vdb2 /EXT4-t

# 顯示 partition type(-T) & human readable(-h)
$ df -T -h

# 查詢目前在 mount point 的 user 有哪些
$ sudo fuser -v /XFS
                     USER        PID ACCESS COMMAND
/mnt:                root     kernel mount /XFS
                     student    1663 ..c.. bash
                     root       1936 ..c.. sudo

# 踢掉目前在指定 mount point 的 user
$ sudo fuser -km /NFS
```

### /etc/fstab

若要讓 partition 開機後就固定掛載在指定目錄，就要使用 **/etc/fstab** 來達成：

```bash
[vagrant@server ~]$ cat /etc/fstab | grep '^[^#]'
# 1. partition = /dev/mapper/centos-root (也可以用 partition UUID)
# 2. mount point = /
# 3. partition type = xfs
# 4. mount options = defaults (rw,suid,dev,exec,auto,nouser,async)
# 5. 指定是否需要 dump = 0 (不需 dump)
# 6. 系統開機時進行 fsck 檢查的順序
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=cb6b4419-b7cc-4315-b5bd-5926d21e944e   /boot    xfs     defaults        0 0
/dev/mapper/centos-home /home                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0

/dev/vdb1   /XFS    xfs   defaults    1   2
/dev/vdb2   /EXT4   ext4  defaults    1   2
```

編輯完 `/etc/fstab` 後，必須執行 `sudo mount -a` 檢查是否 partition 可以正確掛載

> 建議 partition 透過 UUID 指定，可使用 `blkid [partition_name]` 來取得 partition UUID 的資訊(但似乎只對已經 mount 到特定目錄的 partition 有效)

也可以使用 LABEL 的方式設定在 /etc/fstab 中：

```bash
# 為 XFS partition 命名
$ sudo xfs_admin -L XFS /dev/vdb1
# 為 EXT4 partition 命名
$ sudo e2label /dev/vdb2 EXT4

# 查詢結果
$ sudo blkid

$ sudo mount -L XFS /XFS
$ sudo mount -L EXT4 /EXT4

# /etc/fstab 的設定可以改成如下
LABEL=XFS   /XFS    xfs   defaults    1   2
LABEL=EXT4  /EXT4   ext4  defaults    1   2
```

> Label 還是有可能重複，因此原廠建議用上面的 UUID 掛載，是最保險沒問題的

--------------------------------------------------------------------

## 9.2 Managing Swap Space

以下以切割 /dev/vdb 為例：

1. 使用 `fdisk`(82) or `gdisk`(8200) 進行硬碟分割

2. 將 partition 格式化為 swap：`sudo mkswap /dev/vdb1`

3. 掛載 swap partition：`sudo swapon /dev/vdb1`

若是要在 **/etc/fstab** 中掛載 fstab，在 dump & fsck 的欄位都要給 0，標準設定方式如下：

`UUID=e02aae2f-70a9-4b1b-a1e1-9016b0acc3e4  swap  swap  defaults  0 0`

### 其他 swap 相關指令

- `sudo swapon -a`：檢查 `/etc/fstab` 中的 swap partition 設定並掛載

- `swap -s`：檢視目前系統中 swap partition 的資訊以及使用優先權
