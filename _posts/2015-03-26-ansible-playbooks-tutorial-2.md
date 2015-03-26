---
layout: post
title:  "[Ansible] playbook 使用介紹(vars & handler & template)"
description: "在這邊文章中，介紹如何使用 playbook，搭配其中 var / handler / template 等功能，快速建立起支援 SSL 的 nginx web server"
date:   2015-03-26 16:35:00
published: true
comments: true
categories: [ansible]
tags: [Ansible, Linux]
---

Playbook
========

這次我們要透過 playbook 安裝一個支援 SSL 的 nginx web server，以下是 playbook 的內容：

``` yaml
---
- name: Configure web server with nginx and ssl
  # 指定安裝的 Group(webservers)
  hosts: webservers
  # 以 root 的權限安裝
  sudo: true
  # 預先定義變數
  vars:
    key_file: /etc/nginx/ssl/nginx.key
    cert_file: /etc/nginx/ssl/nginx.crt
    conf_file: /etc/nginx/sites-available/default
    server_name: localhost
  tasks:
    # 使用 apt module 安裝 nginx web server
    - name: Install nginx
      apt: >
        name=nginx
        update_cache=yes
    # 透過 file module 建立存放 SSL 憑證的目錄	
    - name: create directories for ssl certificates
      file: >
        path=/etc/nginx/ssl
        state=directory
    # 透過 copy module 複製 SSL 金鑰檔案到指定位置(使用 vars & template)
    - name: copy SSL key
      copy: > 
        src=/home/vagrant/ansible/files/ssl/nginx.key
        dest={{ key_file }}
        owner=root
        mode=0600
    # 透過 copy module 複製 SSL 憑證檔案到指定位置(使用 vars & template)
    # 並使用 notify 關鍵字呼叫 handler 重新啟動 nginx
    - name: copy SSL certificate
      copy: >
        src=/home/vagrant/ansible/files/ssl/nginx.crt
        dest={{ cert_file }}
      notify: restart nginx
    # 使用 template module 直接指定 jinja2 template 並套用上面定義的變數作為來源檔案
    # 並複製到指定的位置
    - name: copy nginx config file
      template: >
        src=/home/vagrant/ansible/templates/nginx.conf.j2
        dest={{ conf_file }}
      notify: restart nginx
    # 使用 file module 建立 symbolic link
    # 並使用 notify 關鍵字呼叫 handler 重新啟動 nginx
    - name: enable configuration
      file: >
        dest=/etc/nginx/sites-enabled/default
        src={{ conf_file }}
        state=link
      notify: restart nginx
  # handler 定義 (呼叫需要使用 handler name)
  handlers:
    # 使用 service module 重新啟動 nginx
    - name: restart nginx
      service: >
        name=nginx
        state=restarted
```

-------------------------------------------

變數(vars)
==========

在 playbook 中定義了四個變數，分別是：

1. **key_file**

2. **cert_file**

3. **conf_file**

4. **server_name**。

這四個變數，可以用在 playbook 內，也可以用在 template 內，透過類似以下的方式就可以使用：

``` bash
{{ var_name }}
```

-------------------------------------------

SSL certificates
================

由於是透過 ansible 進行安裝設定，因此所有的重要檔案都會統一從 control node 產生，包含產生 SSL 憑證檔案

在 control node 上執行以下指令，產生設定 https 網站所需要的 SSL certificate：

``` bash
# 建立存放 ssl certificate 的目錄
$ mkdir /home/vagrant/ansible/ssl

# 透過 openssl 建立自用的 ssl certificate
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /home/vagrant/ansible/ssl/nginx.key -out /home/vagrant/ansible/ssl/nginx.crt
```

-------------------------------------------

Template (Jinja2)
=================

Ansible 使用 Jinja2 template engine，搭配變數套用成各式文件檔，在這邊則是套用成 nginx 的設定檔

以下是  /home/vagrant/ansible/templates/nginx.conf.j2 的內容：

```
server {
	listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;
	listen 443 ssl;
	root /usr/share/nginx/html;
	index index.html index.htm;
	server_name {{ server_name }};
	ssl_certificate {{ cert_file }};
	ssl_certificate_key {{ key_file }};
	location / {
			try_files $uri $uri/ =404;
	}
}
```

在 template 中的 <font color='blue'>**{{ server_name }}**</font>、<font color='blue'>**{{ cert_file }}**</font>、<font color='blue'>**{{ key_file }}**</font> 三個分別用左右兩個大括號包起來的部分，都會被 playbook 中定義的 vars 給取代。

-------------------------------------------

Handler
=======

這個 playbook 中有包含了 **handler** 的設定，它的概念就像是 task，只是它必須透過 <font color='red'>**notofy**</font> module 去呼叫，而呼叫時必須使用 handler name。

在上面的 playbook 中，handler 的工作是使用 service module 重新啟動 nginx。

-------------------------------------------

執行結果
========

了解了 playbook 所有的部分後，接著

``` bash
$ ansible-playbook web-ssl.yml

PLAY [Configure web server with nginx and ssl] ********************************

GATHERING FACTS ***************************************************************
ok: [192.168.30.10]

TASK: [Install nginx] *********************************************************
changed: [192.168.30.10]

TASK: [create directories for ssl certificates] *******************************
changed: [192.168.30.10]

TASK: [copy SSL key] **********************************************************
changed: [192.168.30.10]

TASK: [copy SSL certificate] **************************************************
changed: [192.168.30.10]

TASK: [copy nginx config file] ************************************************
changed: [192.168.30.10]

TASK: [enable configuration] **************************************************
ok: [192.168.30.10]

NOTIFIED: [restart nginx] *****************************************************
changed: [192.168.30.10]

PLAY RECAP ********************************************************************
192.168.30.10              : ok=8    changed=6    unreachable=0    failed=0
```

同樣也是順利執行完成，最後我們可以連結到 [https://192.168.30.10](https://192.168.30.10) 檢視結果，如果有出現以下畫面，表示 https nginx web server 安裝成功囉!

![Ansible - install nginx & openssl](https://lh3.googleusercontent.com/-VBes_uGITog/VROj3rWViUI/AAAAAAAAKws/ArSxsdrgOZo/w692-h307-no/ansible-web-ssl.png)

-------------------------------------------

參考資料
========

- [Ansible Documentation — Ansible Documentation](http://docs.ansible.com/index.html)

- [FREE Ansible Up & Running Preview](http://www.ansible.com/blog/free-ansible-book)