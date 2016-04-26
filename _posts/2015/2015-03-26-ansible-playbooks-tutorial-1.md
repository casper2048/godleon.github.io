---
layout: post
title:  "[Ansible] playbook 入門"
description: "在這邊文章中，介紹如何使用 playbook 進行簡單的工作，快速的在遠端機器上安裝 nginx web server"
date:   2015-03-26 12:00:00
published: true
comments: true
categories: [ansible]
tags: [Ansible, Linux]
---

使用 playbook 前必須先知道的
============================

### Inventory File

Inventory 檔案不能有執行的權限(x)，否則 ansible 會回報以下錯誤：

> ERROR: problem running inventory/file/path --list ([Errno 8] Exec format error)

### SSH private key

透過 SSH private key，可以讓 Ansible control node 直接登入到 remote node 中，但同樣也有些限制在。

ssh private key 只能有擁有者可以存取的權限，若是其他人(group/others)也有存取的權限，ansible 會回報以下錯誤：(必須使用 -vvvv 參數才看的到)

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0777 for '/home/vagrant/ansible/private_key' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
bad permissions: ignore key: /home/vagrant/ansible/private_key
debug2: we did not send a packet, disable method
debug1: No more authentication methods to try.
Permission denied (publickey,password).
```

---------------------------------------------------

Invetory File
=============

在開始撰寫 playbook 之前，先了解 invetory file(**/etc/ansible/hosts**) 的內容：

``` ini
[webservers]
192.168.30.10
```

---------------------------------------------------

撰寫第一個 playbook
===================

每個 playbook 包含的一個或多個 play(劇情)，每個 play 都會包含了以下兩項：

1. 要進行設定的 hosts

2. 要在這些 hosts 上面執行的任務清單

了解以上內容後，建立第一個 playbook，檔名為 <font color='red'>**web-nossl.yml**</font>：

``` yaml
# 檔案開頭
---
- name: Configure webserver with nginx
  hosts: webservers
  # 取得 root 權限執行 task
  sudo: True
  tasks:
    # 複製 nginx 設定檔(從 control node 遠端複製到 managed node 上)
    - name: install nginx
      apt: name=nginx update_cache=yes
	  
    # 透過 copy module 進行檔案複製
    - name: copy nginx config file
      copy: >
		src=/home/vagrant/ansible/files/nginx.conf
		dest=/etc/nginx/sites-available/default

    # 透過 symbolic link 的方式將 default 設定檔放到 /etc/nginx/sites-enabled 目錄中
    - name: enable configuration
      file: >
        dest=/etc/nginx/sites-enabled/default
        src=/etc/nginx/sites-available/default
        state=link
		
    # 複製 nginx 首頁檔案(從 control node 遠端複製到 managed node 上)
    - name: copy index.html	
      copy: >
		src=/home/vagrant/ansible/files/index.html
		dest=/usr/share/nginx/html/index.html
		mode=0644
		
    # 重新啟動 nginx 服務
    - name: restart nginx
      service: name=nginx state=restarted
```

- 在 playbook 中，**name** 的功能類似程式碼中的註解，雖然不寫也沒關係，但寫清楚會讓人更容易了解這個 playbook 的目的何在。

- **host** 則是定義在 inventory file 中的項目，可參考前一段的設定

- **sudo** 則是指定 task 是否已 root 的身分執行

- **tasks** 則是撰寫所要執行的工作，基本上就是 module + arguments

其中有兩個檔案是放在 control node 的，分別是：

<font color='red'>**/home/vagrant/ansible/files/nginx.conf**</font>

``` bash
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        root /usr/share/nginx/html;
        index index.html index.htm;

        server_name localhost;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

<font color='red'>**/home/vagrant/ansible/files/index.html**</font>

``` html
<html>
  <head>
    <title>Welcome to Nginx, via Ansible!</title>
  </head>
  <body>
  <h1>Nginx, configured by Ansible</h1>
  <p>If you can see this, Ansible successfully installed Nginx.</p>
  </body>
</html>
```


## 撰寫 playbook 需要額外注意的地方

建立 yaml 的 playbook 在格式上有些需要注意的地方：

1. 不能用 tab，如果在 vi 裡面使用 tab，記得透過 **set expandtab** 功能把 tab 轉換為 space

2. 每一行設定最後方不能留空白 or tab，會被判定為格試錯誤


## 執行 playbook

最後透過以下方式執行 playbook，執行結果如下：

``` bash
$ ansible-playbook web-nossl.yml

PLAY [Configure webserver with nginx] *****************************************

GATHERING FACTS ***************************************************************
ok: [192.168.30.10]

TASK: [install nginx] *********************************************************
changed: [192.168.30.10]

TASK: [copy nginx config file] ************************************************
changed: [192.168.30.10]

TASK: [enable configuration] **************************************************
ok: [192.168.30.10]

TASK: [copy index.html] *******************************************************
changed: [192.168.30.10]

TASK: [restart nginx] *********************************************************
changed: [192.168.30.10]

PLAY RECAP ********************************************************************
192.168.30.10              : ok=6    changed=4    unreachable=0    failed=0
```

## 執行結果說明

由於 ansible playbook 執行的很順利，因此僅有出現了 ok & changed 狀態，以下說明這兩個狀態：

### ok

表示 task 的執行並沒有改變 remote host 任何狀態，例如：安裝一個已經存在的套件、複製一個已經存在的檔案。

### changed

表示 task 執行後有改變 remote host 的狀態，例如：安裝一個新套件、複製一個新設定檔。

## 檢視是否執行成功

此時我們可以連到 [http://192.168.30.10](http://192.168.30.10) 檢查 nginx 是否已經安裝設定好並啟動

若是成功，則可以看到下面的畫面：

![nginx index page by Ansible](https://lh3.googleusercontent.com/-0go8aIo6o5s/VROGFZR6WDI/AAAAAAAAKwI/fr_3ZoGTwJk/w497-h197-no/ansible-web-nossl.png)

---------------------------------------------------

參考資料
========

- [Ansible Documentation — Ansible Documentation](http://docs.ansible.com/index.html)

- [FREE Ansible Up & Running Preview](http://www.ansible.com/blog/free-ansible-book)