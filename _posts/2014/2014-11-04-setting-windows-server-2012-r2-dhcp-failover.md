---
layout: post
title:  "[Windows]設定 Windows Server 2012 R2 DHCP Failover (容錯移轉)"
date:   2014-11-04 13:57:00
comments: true
categories: [windows]
tags: [Windows, TCP/IP]
---


之前設定了兩台 AD server，打算兩台都安裝 DHCP Server 的服務，並設定 dhcp failover，於是參考了以下文章：

[Windows Server 2012 DHCP Failover夥伴伺服器故障排除 | MIS的背影](http://blog.pmail.idv.tw/?p=7904)

但設定到最後一步總是發生錯誤，出現 **Error 20010** 的錯誤，錯誤訊息大概就是甚麼**選項不存在**之類的...

後來找了一下資料，才想起來之前設定 Wyse solution 的時候有加了 161 & 162 兩個自訂的 DHCP server option(伺服機選項)，兩台 DHCP server 選項內容不同，自然無法進行 failover 的設定。

最後將這兩個選項也設定到另外一台 DHCP server 後，failover 的功能就可以設定起來了!

參考資料
=======

- [Windows Server 2012 DHCP Failover夥伴伺服器故障排除 | MIS的背影](http://blog.pmail.idv.tw/?p=7904)

- [Server 2012 R2 DHCP Failover Error 20010](https://social.technet.microsoft.com/Forums/en-US/a60ed600-58d6-4f06-a8c5-d01104b1eaa7/server-2012-r2-dhcp-failover-error-20010?forum=winserver8gen)

- [Matt on ... Whatever: DHCP Failover Breaks with Custom Options](http://blog.rolpdog.com/2012/11/dhcp-failover-breaks-with-custom-options.html)