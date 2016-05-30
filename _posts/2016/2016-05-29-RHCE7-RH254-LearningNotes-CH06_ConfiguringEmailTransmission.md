---
layout: post
title:  "[RHCE7] RH254 Chapter 6 Configuring Email Transmission Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 6 Configuring Email Transmission 留下的內容"
date: 2016-05-29 15:20:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH254]
---

6.1 Configuring a Send-only Email Configuration
===============================================

## 6.1.1 Email architecture and null clients

RHEL7 Postfix 提供了 `/usr/sbin/sendmail` 作為內部發送通知訊息之用。

`null client` 的工作僅將所有 email 轉送到其他的 mail relay，本章重點在於如何把機器設定為 Postfix null client，藉此透過 `sendmail` & `SMTP` 協定將訊息送到外面的 mail server

## 6.2.2 Transmission of an email message

1. mail client 透過 `SMTP` 傳送郵件

2. 內部的寄信需求可能不需要認證就可以被接受(relay 的部分則大多會有一些規則上 & 防火牆上的限制)

3. 外部的 relay server 會根據目的地 domain 的 `MX` 紀錄，將信轉過去

4. 收信者的 mail server 可能會支援 `POP3` or `IMAP` 等協定來將信件取回，也可能提供 web interface 供使用

## 6.2.3 Postfix

Postfix 是 RHEL7 預設的 mail server，模組化設計，主要設定檔位於 `/etc/postfix/main.cf` 中 (設定檔都集中在 `/etc/postfix` 目錄下)

以下是比較重要的 Postfix 設定：

| Setting | Purpose |
|---------|---------|
| `inet_interfaces` | 設定監聽流入 & 流出訊息的網路介面。<br />`loopback-only`: 監聽 127.0.0.1 & ::1<br />`all`: 監聽所有網路介面<br />Default: `inet_interfaces = localhost`<br />**【註】**若有多個 hostname or ip 需要監聽，則用空白隔開 |
| `myorigin` | 將本地端發出去的 mail domain 改為此台主機<br />Default:`myorigin = $myhostname` |
| `relayhost` | 將訊息轉到指定的外部 mail relay<br />Default:`relayhost = ` |
| `mydestination` | 設定 mail 的終點，到指定的地方後就會直接進入 local mailbox<br />Default:`mydestination = $myhostname, localhost.$mydomain, localhost` |
| `local_transport` | 定義到 **mydestination** 的 mail 要怎麼處理<br />`local:$myhostname`：使用 local mail agent 把 mail 存放到 /var/spool/mail 中 |
| `mynetworks` | 允許從這台 mail server 進行 relay 的來源清單(用空白隔開)<br />Default:`mynetworks = 127.0.0.0/8 [::1]/128` |

要變更 Postfix 的行為，使用者可以透過手動修改 `/etc/postfix/main.cf` 或是透過 `postconf` 指令：

```bash
# 顯示所有設定
$ postconf
.....
unverified_sender_reject_reason =
unverified_sender_tempfail_action = $reject_tempfail_action
verp_delimiter_filter = -=+
virtual_alias_domains = $virtual_alias_maps
virtual_alias_expansion_limit = 1000
virtual_alias_maps = $virtual_maps
virtual_alias_recursion_limit = 1000
....

# 僅查詢指定的設定
$ postconf inet_interfaces myorigin
inet_interfaces = localhost
myorigin = $myhostname

# 使用 "-e" 參數修改設定
$ sudo postconf -e 'myorigin = example.com'
$ postconf myorigin
myorigin = example.com

# 需要 reload or restart postfix.service 才會生效
$ sudo systemctl reload postfix.service
```

> 建議直接編輯 `/etc/postfix/main.cf`，因為參數選項太多不容易記憶，詳細參數說明可參考 `postconf(5)`

## 6.2.4 Postfix null client configuration

要把 Postfix 設定成一個標準的 null client，要達到下面幾個要求：

1. sendmail 是用來將所有 email 轉送到外部的 mail relay 之用

2. local Postfix 不接受處理本地端傳送任何郵件訊息，只會負責 relay 出去

3. 使用者可以透過 null client 收發信件

統整之後，一共需要以下設定：

| Directive | Null Client(serverX.example.com) |
|-----------|--------------------------------------|
| `inet_interfaces` | inet_interfaces=loopback-only |
| `myorigin` | myorigin=desktopX.example.com |
| `relayhost` | relayhost=[smtpX.example.com] |
| `mydestination` | mydestination=  |
| `local_transport` | local_transport=error: local delivery disabled |
| `mynetworks` | mynetworks=127.0.0.0/8, [::1]/128 |

因此我們可以透過以下指令設定 null client：

```bash
# 設定要 relay mail 的主機資訊
$ sudo postconf -e "relayhost=[smtpX.example.com]"

# 設定監聽介面
$ sudo postconf -e "inet_interfaces = loopback-only"

# 設定來自 localhsot 的郵件訊息都由 relay 處理
$ sudo postconf -e "mynetworks=127.0.0.0/8 [::1]/128"

# 修改寄信出去所表示的 domain name
$ sudo postconf -e "myorigin=desktopX.example.com"

# 設定 null client 不是郵件的終點位置，避免郵件訊息傳到本機帳號
$ sudo postconf -e "mydestination="

# 停止 local mail delivery
$ sudo postconf -e "local_transport=error: local delivery disabled"
```

---------------------------------------------------------

Practice: Configuring Send-only Email Service
=============================================

## 目標

1. 將 serverX 設定成 null client

2. 將所有的 mail relay 到 `smtpX.example.com`

> desktopX & serverX 都已經執行 lab 相關指令

## 實作過程

1、設定 serverX

```bash
$ sudo postconf -e "relayhost=[smtp0.example.com]"
$ sudo postconf -e "inet_interfaces=loopback-only"
$ sudo postconf -e "mynetworks=127.0.0.0/8 [::1]/128"
$ sudo postconf -e "myorigin=desktop0.example.com"
$ sudo postconf -e "mydestination="
$ sudo postconf -e "local_transport=error: local delivery disabled"

[student@server0 ~]$ sudo systemctl restart postfix.service
```
