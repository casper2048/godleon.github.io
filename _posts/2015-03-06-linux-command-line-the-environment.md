---
layout: post
title:  "學習 Linux Command Line 筆記(2) - 組態與環境"
description: "學習 Linux Command Line 筆記 - 組態與環境"
date:   2015-03-06 14:55:00
published: true
comments: true
categories: [linux]
tags: [Linux, Bash]
---

以下只記錄一下原本不會、比較不熟的，或是認為比較重要的 Linux Command Line 學習紀錄

環境(The Environment)
=====================

shell session 有兩種：

1. `login shell session`：啟動 console 終端機時所產生的 session。

2. `non-login shell session`：通常是在 GUI 的環境中啟動終端機所產生的 session。

登入 shell session 會讀取一個 or 多啟動檔：

### login shell session 的啟動檔

| 檔案 | 內容 |
|------|------|
| **/etc/profile** | 套用給所有使用者的全域性組態 script |
| **~/.bash_profile** | 使用者個人的啟動檔。可用來擴充 or 複寫全域設定 |
| **~/.bash_login** | 若找不到 ~/.bash_profile，bash 會試著讀取此檔案 |
| **~/.profile** | 若 ~/.bash_profile & ~/.bash_login 都找不到，bsah 會試著讀取此檔案(這是在 Debian/Ubuntu 分支才有的設定) |

### non-login shell session 的啟動檔

| 檔案 | 內容 |
|------|------|
| **/etc/bash.bashrc** | 套用給所有使用者的全域性組態 script |
| **~/.bashrc** | 使用者個人啟動檔。可用來擴充 or 複寫全域設定 |

---------------------------------------

vi 簡介(A Gentle Introduction To vi)
====================================

### 搜尋

使用 `/` 可進行搜尋，重複搜尋可使用 `n`

### 搜尋 & 取代

``` bash
# 將檔案中所有的 Line 更換為 line
:%s/Line/line/g
```

| 項目 | 意義 |
|------|------|
| **:** | 以冒號起始 ex 命令 |
| **%** | `%` 為第一行到最後一行的簡寫。也可以寫成 `1,5`(第 1~5 行) or `1,$`(第 1 ~ 最後一行) |
| **s** | 操作方式, s 表示搜尋 & 取代 |
| **/Line/line/** | 搜尋模式 & 替代的文字 |
| **g** | 全域性。若省略，則只會取代每一行的第一個被搜尋到的字串。若補上 `c` 則會要求使用者確認 |

### 編輯多檔

``` bash
$ vi file1 file2 file3
```

- `:n`：移到下一個檔案

- `:N`：移到前一個檔案

- `:buffers`：檢視目前編輯的檔案清單

- `:buffer 2`：移到清單中第二個檔案

---------------------------------------

客製化提示符號(Customizing The Prompt)
======================================

主要就是 `PS1` 的設定了，可以參考以下連結：

- [鳥哥的 Linux 私房菜 -- 學習 bash shell](http://linux.vbird.org/linux_basic/0320bash.php)

- [凍仁的筆記: 多彩的 Bash 提示字元 ($PS1) on Debian 5](http://note.drx.tw/2010/04/bash-on-debian-lenny.html)

- [Bash prompt PS1 設定 與 產生器 - Tsung's Blog](http://blog.longwin.com.tw/2012/11/bash-prompt-set-generator-2012/)

---------------------------------------

參考資料
========

- [The Linux Command Line by William E. Shotts, Jr.](http://linuxcommand.org/tlcl.php)