---
layout: post
title:  "[RHCE7] RH134 Chapter 13. Accessing Network Storage with Network File System (NFS) 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 13. Accessing Network Storage with Network File System (NFS) 留下的內容"
date: 2016-05-13 04:45:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH134]
---

11.1 Mounting Network Storage with NFS
======================================

## 11.1.1 Manualling mouting and unmouting NFS shares

### NFS 4.0 之前：

- NFS 啟動時會向 rpcbind 註冊

- NFS 的 port 是 rpcbind(TCP 111) 所配發的

- rpcbind restart，NFS 也要跟著 restart

- 查詢遠端主機開放的目錄：`sudo showmount -e [remote_host_name or IP]`

- 掛載：`sudo mount -t nfs [remote_host_name or IP]:/content /mnt`


### NFS 4.0：

- 固定使用 TCP port 2049

- 無法使用 **showmount**，因此必須預先知道開放的目錄

- 同時掛載所有開放的目錄：`mount -t nfs [remote_host_name or IP]:/ /mnt` (確定路徑也可以 mount 特定目錄)

> NFSv2, NFSv3, NFSv4 預設是同時開啟的

RHEL 7 預設會使用 NFSv4，若不支援才會往下降；NFSv4 使用 TCP，舊版本則會用 TCP or UDP

掛載 NFS 方式：

- 手動下命令

- 加到 `/etc/fstab` 中：`[remote_host_name]:/content  /mnt  nfs   default,sync,sec=xxx   0 0`

> 網路磁碟機建議使用 sync 選項，確保資料完整性 (另外了解 soft & hard 的差異)

> 使用 fsck 檢查的設備，必須是 umount or readonly 的狀態所，所出來的結果才是正確的

## 11.1.2 Security methods

- `none`：client 匿名存取(送來的身份會被忽略)，身份全部都會轉成 **nfsnobody**

- `sys`：client 存取時會把身份送進來(例如：UID 1000)，NFS server 會找到對應的身份 & 權限 (**<font color='red'>預設值</font>**，但其實不安全)

- `krb5`：使用 kerberos ticket，但通訊時以明碼傳遞

- `krb5i`：同上，但加上完整性驗證，可判斷傳輸的資料是否被修改

- `krb5p`：同上，但傳輸的資料會被加密

> 若要讓 NFS 可以向 Kerberos 進行驗證，也同時需要有 `nfs-secure` 服務(在 `nfs-utils` 套件中)

> 使用 `klist` 可以查詢目前所擁有的 kerberos ticket

-------------------------------------------------------------------------

11.2 Automounting Network Storage with NFS
==========================================

需要安裝套件 `autofs`，會在系統中提供一個 autofs system service

**優點：**

- 使用者不需要 root 權限來執行 mount & unmount 指令

- automounter 的設定對所有使用者皆有效，主要是著重在存取權限的調整

- 不會與 NFS server 一直持續保持連線，節省網路 & 系統資源

- 與原本 mount 所使用的設定參數相同

- 提供 direct & indirect 兩種目錄 mapping 模式，以提供 mount point 一定程度的彈性

- indirect 模式下，目錄會被自動生成，降低手動的需求

- automounter 其實可以用在 NFS 之外的其他檔案系統上

## direct-map

目錄 `/etc/auto.master.d`，檔名不拘，副檔名必須是 `.autofs`

`/etc/sysconfig/autofs` 中有 `TIMEOUT` 參數可以用，表示離開目錄多久會自動 unmount

自動掛載光碟機：`/mnt/cdrom   -ro,iso9660   :/dev/sr0`

## indirect-map

掛載的目錄都會在同一個目錄下

可用萬用字元


------------------------------------------------------------------------
