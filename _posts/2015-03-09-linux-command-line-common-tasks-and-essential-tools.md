---
layout: post
title:  "學習 Linux Command Line 筆記(3) - 一般任務與必要工具程式"
description: "學習 Linux Command Line 中一般任務與必要工具程式，"
date:   2015-03-09 10:55:00
published: true
comments: true
categories: [linux]
tags: [Linux, Bash]
---

以下只記錄一下原本不會、比較不熟的，或是認為比較重要的 Linux Command Line 學習紀錄

套件管理(Package Management)
============================

### 低階套件安裝命令

| 型態 | 命令 |
|------|------|
| Debian(Ubuntu) | dpkg --install *package_file* |
| Red Hat | rpm -i *package_file* |


### 移除套件

| 型態 | 命令 |
|------|------|
| Debian(Ubuntu) | apt-get remove *package_name* |
| Red Hat | yum erase *package_name* |


### 低階套件更新命令

| 型態 | 命令 |
|------|------|
| Debian(Ubuntu) | dpkg --install *package_file* |
| Red Hat | rpm -U *package_file* |

### 顯示已安裝套件資訊

| 型態 | 命令 |
|------|------|
| Debian(Ubuntu) | apt-cache show *package_name* |
| Red Hat | yum info *package_name* |

### 尋找套件安裝的檔案

尋找指定的檔案是由哪個套件負責：

| 型態 | 命令 |
|------|------|
| Debian(Ubuntu) | dpkg --search *file_name* |
| Red Hat | rpm -qf *file_name* |

----------------------------

儲存媒體(Storage Media)
=======================

### 掛載 ISO Image

``` bash
$ mkdir /mnt/iso_image
$ mount -t iso9660 -o loop image.iso /mnt/iso_image
```

----------------------------

檔案搜尋(Search For Files)
==========================

### locate

- 在 CentOS 中沒有 locate，必須安裝 `mlocate` 套件才可以使用

- locate 資料庫必須透過 `updatedb` 指令建立

### find

``` bash
# 計算出"權限不為 0600 的檔案" or "權限不為 0700 的目錄"的數量加總
$ find / \( -type f -not -perm 0600 \) -or \( -type d -not -perm 0700 \) | wc -l

# 尋找當前目錄中檔名含有 error 的檔案並刪除
$ find ./ -type f -name *error* -delete
```

#### 使用者定義動作

``` bash
# 尋找當前目錄中檔名為 ls 開頭的檔案並使用 ls 一個一個列出
$ find ./ -type f -name 'ls*' -exec ls -l '{}' ';'

# 尋找當前目錄中檔名為 ls 開頭的檔案並使用 ls 一次全部列出
$ find ./ -type f -name 'ls*' -exec ls -l '{}' +

# 與上面的指令相同效果
$ find ./ -type f -name 'ls*' | xargs ls -l
```

----------------------------

打包與備份(Archiving And Backup)
================================

``` bash
# 僅解開指定的檔案
$ tar -xf ../playground2.tar --wildcards './playground/dir-*/file-A'

# 將目錄中檔名為 file-A 的檔案一次打包
$ find playground/ -name 'file-A' -exec tar rf playground.tar '{}' '+'

# 透過 ssh 將遠端主機中的 /playground 資料夾打包、取回並解開至當前目錄
$ ssh root@remote_host_address 'tar -cf - /playground' | tar xf -
```


文字處理(Text Processing)
=========================

### sort (排序)

``` bash
$ ls
sort1.txt  sort2.txt  sort3.txt

# 將所有檔案內容排序並輸出至新檔案
$ sort sort* > final_sort.txt
```

``` bash
# 根據目錄大小排序，並顯示前十筆
$ du -s /usr/share/* | sort -nr | head

# 根據輸出的第五個欄位進行排序後再輸出
$ ls -l /usr/bin | sort -nr -k 5 | head

$ cat distros.txt
SUSE 10.2 12/07/2006
Fedora 10 11/25/2008
SUSE 11.0 06/19/2008
Ubuntu 8.04 04/24/2008
....

# 從欄位 1 開始，結束於欄位 1；欄位 2 使用數字排序
$ sort --key=1,1 --key=2n distros.txt

# 使用第 3 個欄位的第 7 個字元作為排序鍵值(年份) -> 月份 -> 日
$ sort -k 3.7nbr -k 3.1nbr -k 3.4nbr distros.txt
 
# 以 ':' 為分隔符號，以第三個欄位進行數字排序
$ sort -t ":" -k 3n /etc/passwd | head
```

### cut (移除檔案每行的某段)

``` bash
$ cat -A distros.txt
SUSE^I10.2^I12/07/2006$
Fedora^I10^I11/25/2008$
SUSE^I11.0^I06/19/2008$
...

# 取第三個欄位值(預設分隔為 tab)
$ cut -f 3 distros.txt
12/07/2006
11/25/2008
06/19/2008
...

# 取第三個欄位值中的第 7-10 個字元
$ cut -f 3 distros.txt | cut -c 7-10
2006
2008
2008
...

# 以 : 為分隔，取第一個欄位並輸出前十筆資料
$ cut -d ':' -f 1 /etc/passwd | head
root
bin
daemon
...

# 透過 expand 將 tab 轉換為固定數量的空格，並取得指定位置的字串
$ expand distros.txt | cut -c 23-
2006
2008
2008
```

### paste (合併檔案行列)

``` bash
$ cat distros-dates.txt
11/25/2008
10/30/2008
06/19/2008
...

$ cat distros-versions.txt
Fedora  10
Ubuntu  8.10
SUSE    11.0
...

# 合併多個檔案並輸出
$ paste distros-dates.txt distros-versions.txt
11/25/2008      Fedora  10
10/30/2008      Ubuntu  8.10
06/19/2008      SUSE    11.0
```

### join (以共同欄位結合兩個檔案行列)

``` bash
$ cat distros-key-names.txt
11/25/2008      Fedora
10/30/2008      Ubuntu
06/19/2008      SUSE
...

$ cat distros-key-vernums.txt
11/25/2008      10
10/30/2008      8.10
06/19/2008      11.0
... 

# 將兩檔案透過共同欄位進行合併
$ join distros-key-names.txt distros-key-vernums.txt
11/25/2008 Fedora 10
10/30/2008 Ubuntu 8.10
06/19/2008 SUSE 11.0
```

### diff (逐行比較檔案)

``` bash
$ cat fileA.txt
Line A
Line B
Line C
Line D

$ cat fileB.txt
Line B
Line X
Line D
Line K

# 本文格式輸出
$ diff -c fileA.txt fileB.txt
*** fileA.txt   2015-03-08 09:52:50.449806257 +0000
--- fileB.txt   2015-03-08 09:53:17.259213595 +0000
***************
*** 1,4 ****
- Line A
  Line B
! Line C
  Line D
--- 1,4 ----
  Line B
! Line X
  Line D
+ Line K

# 統一格式輸出
$ diff -u fileA.txt fileB.txt
--- fileA.txt   2015-03-08 09:52:50.449806257 +0000
+++ fileB.txt   2015-03-08 09:53:17.259213595 +0000
@@ -1,4 +1,4 @@
-Line A
 Line B
-Line C
+Line X
 Line D
+Line K
```

### patch (將 diff 套用至原始檔案)

透過 diff & patch 更新檔案內容：

``` bash
$ cat file1.txt
Line1
Line2
Line3

$ cat file2.txt
Line1
Line2 modified
Line3
Line4

# 使用 diff 產生 patch 檔
$ diff -Naur file1.txt file2.txt > patchfile.txt

# 透過 patch 將 patch 檔案更新至 file1.txt
[root@localhost test_patch]# patch < patchfile.txt
patching file file1.txt

[root@localhost test_patch]# cat file1.txt
Line1
Line2 modified
Line3
Line4
```

### tr (轉譯或刪除字元)

``` bash
$ echo "lowercase letters" | tr a-z A-Z
LOWERCASE LETTERS

$ echo "lowercase letters" | tr [:lower:] A
AAAAAAAAA AAAAAAA

# 去除重複字元
[root@localhost test_patch]# echo "aaabbbccc" | tr -s ab
abccc

# 重複字元沒有相連就無效
[root@localhost test_patch]# echo "abcabcabc" | tr -s ab
abcabcabc
```

### sed (過濾與轉換文字的串流編輯器)

``` bash
$ sed -n '1,3p' distros.txt
SUSE    10.2    12/07/2006
Fedora  10      11/25/2008
SUSE    11.0    06/19/2008

# 列出包含 SUSE 字眼的行列
$ sed -n '/SUSE/p' distros.txt
SUSE    10.2    12/07/2006
SUSE    11.0    06/19/2008
SUSE    10.3    10/04/2007
SUSE    10.1    05/11/2006

# 列出沒有 SUSE 字眼的行列
$ sed -n '/SUSE/!p' distros.txt
Fedora  10      11/25/2008
Ubuntu  8.04    04/24/2008
....

# 將 MM/DD/YYYY 的日期格式更換為 YYYY-MM-DD
$ sed 's/\([0-9]\{2\}\)\/\([0-9]\{2\}\)\/\([0-9]\{4\}\)$/\3-\1-\2/' distros.txt
SUSE    10.2    2006-12-07
Fedora  10      2008-11-25
SUSE    11.0    2008-06-19
....
```

### groff

透過 sed & groff 的搭配，可以產生類似下方的效果：

```
 +------------------------------+
 |  Linux Distribution Report   |
 +------------------------------+
 | Name    Version    Released  |
 +------------------------------+
 |Fedora     5       2006-03-20 |
 |Fedora     6       2006-10-24 |
 |Fedora     7       2007-05-31 |
 |Fedora     8       2007-11-08 |
 |Fedora     9       2008-05-13 |
 |Fedora    10       2008-11-25 |
 |SUSE      10.1     2006-05-11 |
 |SUSE      10.2     2006-12-07 |
 |SUSE      10.3     2007-10-04 |
 |SUSE      11.0     2008-06-19 |
 |Ubuntu     6.06    2006-06-01 |
 |Ubuntu     6.10    2006-10-26 |
 |Ubuntu     7.04    2007-04-19 |
 |Ubuntu     7.10    2007-10-18 |
 |Ubuntu     8.04    2008-04-24 |
 |Ubuntu     8.10    2008-10-30 |
 +------------------------------+
```

### dirname && basename

``` bash
# 取得完整路徑中不含程式的路徑部分
$ dirname /var/spool/mail
/var/spool

# 取得完整路經中的程式名稱
$ basename /var/spool/mail
mail
```

----------------------------

參考資料
========

- [The Linux Command Line by William E. Shotts, Jr.](http://linuxcommand.org/tlcl.php)