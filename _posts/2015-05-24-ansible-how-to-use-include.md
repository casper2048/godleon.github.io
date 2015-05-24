---
layout: post
title:  "[Ansible] Include 使用介紹"
description: "在這邊文章中，介紹如何透過 Include，將可重複利用的東西拉出，並可用再多個 playbook 中"
date:   2015-05-24 07:00:00
published: true
comments: true
categories: [ansible]
tags: [Ansible, Linux]
---

前言
====

用了 ansible 一段時間後，您可能會遇到以下這些狀況：

1. playbook 越寫越多，越寫越大

2. 發現有許多重複性的設定

這時候也許就會考慮將容易重複利用的內容從 playbook 中拉出，獨立成一個個的小檔案，當有需要呼叫此內容時，再透過 <font color='red'>**includes**</font> 關鍵字將其包進來使用。


------------------------------


Include 使用範例說明
====================

這個部分以實際範例來說明 Include 的使用方式。

### 檔案結構

``` bash
./
├── handlers
│   └── handlers.yml
├── Inventory
├── task1.yml
├── tasks
│   └── foo.yml
```

### 檔案內容

#### Inventory

``` bash
[cluster]
node01 ansible_ssh_host=192.168.122.101
node02 ansible_ssh_host=192.168.122.102
```

#### tasks/foo.yml

``` yaml
---

- name: First Command for {{ usr_1 }}
  command: /bin/echo "First Command {{ usr_1 }}"
  
- name: Second Command for {{ usr_2 }}
  command: /bin/echo "Second Command {{ usr_2 }}"
```

#### handlers/handlers.yml

``` yaml
- name: restart apache
  service: name=apache2 state=restarted
```

#### task1.yml

``` yml
# 使用指令
# ansible-playbook task1.yml -i Inventory -K -v
---

- name: Test Include statement
  hosts: node01
  remote_user: vagrant
  sudo: false

  tasks:
    - name: say hi
      tags: foo
      shell: echo "hi....."
  
    # 以下為幾個不同的 include 方式，可以選擇自己喜歡的
    - include: tasks/foo.yml usr_1=Larry usr_2=Page
    
    - { include: tasks/foo.yml, usr_1: Machael, usr_2: Jordan }
    
    - include: tasks/foo.yml
      vars:
        usr_1: John
        usr_2: Lu
        
    - name: Install Apache service
      apt: name=apache2 state=present update_cache=yes
      notify: restart apache
      sudo: true
      
  
  # handler 的部分也可以 include
  handlers:
    - include: handlers/handlers.yml
```


------------------------------

參考資料
========

- [Playbook Roles and Include Statements — Ansible Documentation](http://docs.ansible.com/playbooks_roles.html)