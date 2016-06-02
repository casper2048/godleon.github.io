---
layout: post
title:  "[Linux] 如何加速 X11 Forwarding"
description: "此文章介紹如何透過修改 ssh 連線加密演算法，來達到加速 X11 Forwarding 的效果"
date: 2016-06-02 08:30:00
published: false
comments: true
categories: [linux]
tags: [Linux, Ubuntu]
---

當你只有一台虛弱的 Linux 筆電，但有一台強大的 server 時，有沒有辦法把 workload 放到 server 上減輕一下負擔阿.....??

答案是有的，就是透過 X11 forwarding 的技術，把 server 的畫面傳到 Linux 筆電上來

> 當然 server 也要有 X11 桌面啦!

原本只要 `ssh -X user@remote.server.com` 就可以了，進到 server 後，就可以執行像是 `firefox &`、`google-chrome &` 把指定的程式呼叫出來並在 Linux 筆電端呈現!

但其實 ssh 連線是加密的，且預設的加密演算法 overhead 還不小，因此可以透過下列方式優化：

## server 端

```bash
# 加入各式各樣加密演算法
$ echo "Ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,blowfish-cbc,aes128-cbc,3des-cbc,cast128-cbc,arcfour,aes192-cbc,aes256-cbc" | sudo tee --append /etc/ssh/sshd_config

# 重新啟動 sshd 服務
$ sudo systemctl restart sshd.service
```

## client 端

直接連線：`ssh -XC -c blowfish-cbc,arcfour user@remote.server.com`

若參數不容易記憶，可以編輯 ：

```bash
Host remote.server.com
  Compression yes
  ForwardX11 yes
  Ciphers blowfish-cbc,arcfour
```

接著直接連線即可：`ssh user@remote.server.com`

References
==========

- [How to speed up X11 forwarding in SSH - Xmodulo](http://xmodulo.com/how-to-speed-up-x11-forwarding-in-ssh.html)

- [Steronius' Programmatically Tolerable Repository of Technical Goodies: SSH - no matching cipher found](http://steronius.blogspot.tw/2014/10/ssh-no-matching-cipher-found.html)
