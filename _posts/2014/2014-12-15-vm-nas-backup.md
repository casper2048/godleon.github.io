---
layout: post
title:  "VM & NAS 備份設定記錄"
date:   2014-12-15 11:35:00
comments: true
categories: [virtualization]
tags: [Virtualization]
---

為了避免設定步驟 & 條件忘記......所以筆記一下...

Backup Server
=============

OS：Debian 7.7.0

### NFS Server

#### 安裝 NFS Serevr 套件

``` bash
root~# apt-get -y install nfs-kernel-server
```

#### 預設停用 NFSv4，使用 NFSv3 (目前 vSphere 5.5 僅支援 NFSv3)

修改 **/etc/default/nfs-kernel-server**

``` bash
RPCMOUNTDOPTS="--manage-gids --no-nfs-version 4"
```

### 設定掛載路徑

修改 **/etc/exports**

``` bash
/path/to/backup    xxx.xxx.xxx.xxx(rw,sync,no_root_squash,no_subtree_check)
/path2/to/open    *(rw,sync,no_root_squash,no_subtree_check)
```


外部備份設備
===========

### 4TB 硬碟處理方式

因為目前買的硬碟超過 2TB，所以要用 GPT 格式來分割，才可以使用超過 2TB 的空間，相關分割方式可參考以下連結：

- [肥佳洛的學習網 » 2TB 單碟解決方式](http://figaro.neo-info.net/?p=313)

- [Linux如何使用超過2TB以上的裝置(HDD,iSCSI)空間(將partition切成GPT格式)](http://dreamtails.pixnet.net/blog/post/28175881)

### rsync

- [rsync 取代 cp , 有進度狀態, 又可保留權限 @ CC :: 隨意窩 Xuite日誌](http://blog.xuite.net/csiewap/cc/44769207)

- [資料備份同步工具簡介— rsync](http://newsletter.ascc.sinica.edu.tw/news/read_news.php?nid=1742)

- [使用 rsync 實做重複資料刪除技術](http://www.ringline.com.tw/support/techpapers/storage/135--rsync-.html)

- [Rsync and sparse files | Gergap's Weblog](https://gergap.wordpress.com/2013/08/10/rsync-and-sparse-files/)


### Disk Clone

- [Linux 下使用dd的應用](http://nathan-inlinux.blogspot.tw/2013/05/linux-dd.html)


Shell Script
============

### 字串轉大小寫
 - [Converting string to lower case in Bash shell scripting - Stack Overflow](http://stackoverflow.com/questions/2264428/converting-string-to-lower-case-in-bash-shell-scripting)

### string.contains() 功能

在 ESXi Host 上執行正確
- [command line - How to determine if a string is a substring of another in bash? - Ask Ubuntu](http://askubuntu.com/questions/299710/how-to-determine-if-a-string-is-a-substring-of-another-in-bash)

在 ESXi Host 上執行錯誤
- [Shell Script: Check if a string contains a substring](http://notepad2.blogspot.tw/2012/09/shell-script-check-if-string-contains.html)

### string.replace() 功能

- [unix - Replace a string in shell script - Stack Overflow](http://stackoverflow.com/questions/3306007/replace-a-string-in-shell-script)

### 文字模式發信

- [使用 ssmtp 於 shell 透過 Gmail 寄信 - Tsung's Blog](http://blog.longwin.com.tw/2009/08/ssmtp-shell-gmail-send-mail-2009/)

- [shell script - Attach files for sending mail which are the result set of find command - Unix & Linux Stack Exchange](http://unix.stackexchange.com/questions/138061/attach-files-for-sending-mail-which-are-the-result-set-of-find-command)

- [如何從命令列透過mutt及gmail寄信 - iT邦幫忙::IT知識分享社群](http://ithelp.ithome.com.tw/question/10054431)

- [在Linux上利用mutt指令寄附加檔Mail @ PeGaSuS - 佩格索斯 :: 痞客邦 PIXNET ::](http://pegasus923.pixnet.net/blog/post/33608243)

### date 用法

- [[Linux] 在 shell script 產生想要的日期格式 - Knuckles_note板 - Disp BBS](https://disp.cc/b/11-4X7n)

- [Shell Script：運用 date 指令取得日期時間(Linux) @ 狐的窩 :: 痞客邦 PIXNET ::](http://mark528.pixnet.net/blog/post/7267328)

### awk & sed

- [awk 工具](http://dywang.csie.cyut.edu.tw/moodle23/dywang/linuxProgram/node27.html)

- [sed 工具](http://dywang.csie.cyut.edu.tw/moodle23/dywang/linuxProgram/node26.html)

- [怎么用awk取文本的最后一行？ - Linux论坛 - 51CTO技术论坛_中国领先的IT技术社区](http://bbs.51cto.com/thread-961661-1.html)

### 其他

- [Linux/Unix 程式設計](http://dywang.csie.cyut.edu.tw/moodle23/dywang/linuxProgram/linuxProgram.html)


ESXi Host 執行 cron Job
=======================

- [Add cron Job to VMware ESX/ESXi | VMware | Jules.FM](http://www.jules.fm/Logbook/files/add_cron_job_vmware.html)


在 Windows 中讀取 Linux 分割區
============================

其他工具試起來都有些小問題，這個是目前測試過最穩定的....

- [Access to Ext 2/3/4, HFS and ReiserFS from Windows](http://www.diskinternals.com/linux-reader/)


龍哥提供的：

- [FSproxy: Welcome to FSproxy Homepage!](http://archive.siejak.pl/fsproxy/wikka8979.html?wakka=HomePage)

- [morgantseng: Portable Captain Nemo Pro 5.1.0.0](http://morgantseng.blogspot.tw/2012/09/portable-captain-nemo-pro-5100.html)


NAS 資料同步
===========

- [分散式檔案系統 (Distributed File System)](http://192.192.75.66/elearn/winserver/8.htm)

- [利用微軟DFS簡化網路檔案管理 | iThome](http://www.ithome.com.tw/node/60103)

- [Windows 2008 DFS(分散式檔案系統)實作及設定(無AD架構) - Jason的電腦健身房- 點部落](http://www.dotblogs.com.tw/dotjason/archive/2009/10/09/10983.aspx)

- [Windows 2008 R2 DFS(分散式檔案系統)HA架構實作及進階設定(含AD架構) - Jason的電腦健身房- 點部落](http://www.dotblogs.com.tw/dotjason/archive/2010/06/17/15934.aspx)


SAMBA 設定 & 檔案打包備份
========================

- [鳥哥的 Linux 私房菜 -- SAMBA 伺服器](http://linux.vbird.org/linux_server/0370samba.php)

- [鳥哥的 Linux 私房菜 -- Linux 的檔案壓縮與打包](http://linux.vbird.org/linux_basic/0240tarcompress.php)

- [使用tar備份資料(Linux) @ James Lin's weblog :: 痞客邦 PIXNET ::](http://sclin0323.pixnet.net/blog/post/24993462)