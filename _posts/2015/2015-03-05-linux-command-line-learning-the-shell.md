---
layout: post
title:  "學習 Linux Command Line 筆記(1) - 學習 Shell"
description: "學習 Linux Command Line 筆記 - 學習 Shell"
date:   2015-03-05 14:50:00
published: true
comments: true
categories: [linux]
tags: [Linux, Bash]
---

以下只記錄一下原本不會、比較不熟的，或是認為比較重要的 Linux Command Line 學習紀錄

重導向(Redirection)
===================

### 1、Standard Input(0)/Output(2)/Error(2)

將 stdout(1) & stderr(2) 同時輸出到指定檔案中：

``` bash
$ ls -l /bin/usr > ls-output.txt 2>&1

#也可以用以下這種新的方式
$ ls -l /bin/usr &> ls-output.txt
```

### 2、Pipelines

使用 **tee** 讀取 stdin 後並同時輸出至 stdout & file：

```
# ls.txt 檔案中會有 ls /usr/bin 的內容
$ ls /usr/bin | tee ls.txt | grep zip
```

---------------------------------------

Seeing The World As The Shell See It
====================================

### 1、Expansion

#### 1.1、Arithmetic Expansion (僅能處理整數)

``` bash
$ echo $((2+2))
4
$ echo $(((5**2)*3))
75
$ echo Five divided by two equal $((5/2))
Five divided by two equal 2
```

#### 1.2、Brace Expansion

``` bash
$ echo Front-{A,B,C}-Back
Front-A-Back Front-B-Back Front-C-Back

$ echo Number_{1..5}
Number_1 Number_2 Number_3 Number_4 Number_5

$ echo {01..15}
01 02 03 04 05 06 07 08 09 10 11 12 13 14 15

$ echo {001..15}
001 002 003 004 005 006 007 008 009 010 011 012 013 014 015

$ echo {Z..A}
Z Y X W V U T S R Q P O N M L K J I H G F E D C B A

$ echo a{A{1,2},B{3,4}}b
aA1b aA2b aB3b aB4b

# 使用 brace expansion 快速建立目錄
$ mkdir photos
$ cd photos/
$ mkdir {2007..2009}-{01..12}
$ ls
2007-01  2007-04  2007-07  2007-10  2008-01  2008-04  2008-07  2008-10  2009-01  2009-04  2009-07  2009-10
2007-02  2007-05  2007-08  2007-11  2008-02  2008-05  2008-08  2008-11  2009-02  2009-05  2009-08  2009-11
2007-03  2007-06  2007-09  2007-12  2008-03  2008-06  2008-09  2008-12  2009-03  2009-06  2009-09  2009-12
```

#### 1.3、Command Substitution

``` bash
$ ls -l $(which --skip-alias cp)
-rwxr-xr-x. 1 root root 151032 Jun 10  2014 /usr/bin/cp

$ ls -l `which --skip-alias cp`
-rwxr-xr-x. 1 root root 151032 Jun 10  2014 /usr/bin/cp

$ file $(ls -d /usr/bin/* | grep zip)
/usr/bin/bunzip2:      symbolic link to `bzip2'
/usr/bin/bzip2:        ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=0x80151d8bb82c3a04911bb01dd561ccc5921ebf67, stripped
/usr/bin/bzip2recover: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=0xa99dfb7461d200a2bcbffbc7cff1394e6b90a926, stripped
/usr/bin/gpg-zip:      POSIX shell script, ASCII text executable
/usr/bin/gunzip:       POSIX shell script, ASCII text executable
/usr/bin/gzip:         ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=0x383452f440145000382210de81b664cfca62308a, stripped
```

### 2、Quoting

``` bash
# shell 認為 echo 有很多參數
$ echo $(cal)
March 2015 Su Mo Tu We Th Fr Sa 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31

# shell 認為 echo 只有一個參數
$ echo "$(cal)"
     March 2015
Su Mo Tu We Th Fr Sa
 1  2  3  4  5  6  7
 8  9 10 11 12 13 14
15 16 17 18 19 20 21
22 23 24 25 26 27 28
29 30 31
```

---------------------------------------

權限(Permissions)
=================

查詢目前使用者身分：

``` bash
$ id
uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

### 參考資料

- [關於 SetUID、SetGID, Sticky Bit 的說明](http://linux.vbird.org/linux_basic/0220filemanager/0220filemanager.php#suid_sgid_sticky)

---------------------------------------

程序(Processes)
===============

``` bash
# 輸出系統資源使用狀態
$ vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 111532   1444 240828    0    0     2     1    7    9  0  0 100  0  0
 
# 每三秒輸出一次資源使用狀態
# vmstat 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 111532   1444 240824    0    0     2     1    7    9  0  0 100  0  0
 0  0      0 111524   1444 240824    0    0     0     0   26   46  0  0 100  0  0
 0  0      0 111524   1444 240824    0    0     0     0   27   45  0  0 100  0  0
 0  0      0 111524   1444 240824    0    0     0     0   27   46  0  0 100  0  0
 0  0      0 111524   1444 240824    0    0     0     0   25   45  0  0 100  0  0
.......
```

---------------------------------------

參考資料
========

- [The Linux Command Line by William E. Shotts, Jr.](http://linuxcommand.org/tlcl.php)