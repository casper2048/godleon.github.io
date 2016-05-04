---
layout: post
title:  "[RHCE7] RH134 Chapter 5 Managing Priority pf Linux Processes 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 5 Managing Priority pf Linux Processes 留下的內容"
date: 2016-05-05 04:50:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH134]
---

<a name="ch5.1" />
5.1 Process Priority and "nice" Concepts
========================================

nice 值範圍 `-20` ~ `19`，沒指定預設就是 0，愈小表示 priority 愈高

> root 可以調大 or 調小 nice value，但一般使用者只能調大

```bash
# 加上參數 -l 可取得 nice value(NI)
[vagrant@server ~]$ ps -l
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
0 S  1000 11502 11501  0  80   0 - 28845 wait   pts/0    00:00:00 bash
0 R  1000 11572 11502  0  80   0 - 34343 -      pts/0    00:00:00 ps
```

> PR 20 = nice 0


------------------------------------------------------------

<a name="ch5.2" />
5.2 Using nice and renice to Influence Process Priority
=======================================================

★★★★★★★★★ Very Important! ★★★★★★★★★ (start)

```bash
# o format => 使用者自訂顯示欄位
$ ps axo pid,comm,nice --sort=-nice
 PID COMMAND          NI
   26 khugepaged       19
  476 alsactl          19
   25 ksmd              5
  537 rtkit-daemon      1
    1 systemd           0
......
   11 rcuos/0           0
   12 watchdog/0        -
......
29921 ps                0
  449 auditd           -4
  467 sedispatch       -4
  462 audispd          -8
 1583 pulseaudio      -11
    5 kworker/0:0H    -20
.......
```

★★★★★★★★★ Very Important! ★★★★★★★★★ (end)


啟動程式時指定 nice 值：

```bash
# nice 示範
$ nice -n 10 top
$ ps -al
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
0 S  1000 11626 11502  0  90  10 - 36537 poll_s pts/0    00:00:00 top
0 R  1000 11627 11604  0  80   0 - 34343 -      pts/1    00:00:00 ps

# renice 示範
$ renice -n 15 11604
11604 (process ID) old priority 0, new priority 15
$ ps -al
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
0 S  1000 11626 11502  0  90  10 - 36537 poll_s pts/0    00:00:00 top
0 R  1000 11630 11604  0  95  15 - 34343 -      pts/1    00:00:00 ps
```

------------------------------------------------------------

補充：Hash Algorithm 特性
=========================

1. 計算的資料來源沒有長度限制

2. 雜湊長度永遠固定

3. 雜湊演算法為單向運算

4. 不同的資料來源不會產出相同的雜湊值

```
$ sha1sum /etc/passwd
524dab4b600e8d87cb7aa5b21e4d57f8996a4374  /etc/passwd

$ echo "1234" | sha1sum
1be168ff837f043bde17c0314341c84271047b31  -
```