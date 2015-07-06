---
layout: post
title:  "[Ansible] Role 使用介紹"
description: "在這邊文章中，介紹 Role 以及如何使用"
date:   2015-04-12 11:14:00
published: true
comments: true
categories: [ansible]
tags: [Ansible, Linux]
---

Ansible Role 是什麼 ?
=====================

Ansible Role 是一種分類 & 重用的概念，透過將 vars, tasks, files, templates, handler ... 等等根據不同的目的(例如：web server、db server)，規劃後至於獨立目錄中，後續便可以利用 include 的概念來使用。

若同樣是 include 的概念，那 role 跟 include 之間不一樣的地方又是在哪裡呢? (include 的介紹可參考[前一篇文章](https://godleon.github.io/blog/2015/05/24/ansible-how-to-use-include))

答案是：<font color='red'>**role 的 include 機制是自動的!**</font>

我們只要事前將 role 的 vars / tasks / files / handler .... 等等事先定義好並按照特定的結構(下面會提到)放好，Ansible 就會很聰明的幫我們 include 完成，不需要再自己一個一個指定 include。

透過這樣的方式，管理者可以透過設定 role 的方式將所需要安裝設定的功能分門別類，拆成細項來管理並撰寫相對應的 script，讓原本可能很龐大的設定工作可以細分成多個不同的部分來分別設定，不僅僅可以讓自己重複利用特定的設定，也可以共享給其他人一同使用。

要設計一個 role，必須先知道怎麼將 files / templates / tasks / handlers / vars .... 等等拆開設定，必須使用以下 project 的結構：

```
site.yml
webservers.yml
fooservers.yml
roles/
   common/
     files/
     templates/
     tasks/
     handlers/
     vars/
     defaults/
     meta/
   webservers/
     files/
     templates/
     tasks/
     handlers/
     vars/
     defaults/
     meta/
```

> 以上是完整的 project 結構，實際上不一定要定義如此完整，如果沒有的部分可以不用，例如：若是沒有 template 的部分，就可以不需要 template 目錄。


此外，ansible 會針對 role(<font color='red'>**x**</font>) 進行以下處理：

1. 若檔案 **roles/<font color='red'>x</font>/tasks/main.yml** 存在，則會被自動加入到 playbook 中的 task list 中

2. 若檔案 **roles/<font color='red'>x</font>/handlers/main.yml** 存在，則會被自動加入到 playbook 中的 handler list 中

3. 若檔案 **roles/<font color='red'>x</font>/vars/main.yml** 存在，則會被自動加入到 playbook 中的 variables list 中

4. 若檔案 **roles/<font color='red'>x</font>/meta/main.yml** 存在，，任何與指定 role 相依的其他 role 設定皆會被自動加入

5. 在 **roles/<font color='red'>x</font>/files/** 目錄中的 copy tasks 或是 script tasks，在 playbook 中使用時不需要指定絕對(absolutely) or 相對(relatively)路徑

6. 在 **roles/<font color='red'>x</font>/templates/** 目錄中的 template tasks，在 playbook 中使用時不需要指定絕對(absolutely) or 相對(relatively)路徑

7. 在 **roles/<font color='red'>x</font>/tasks/** 目錄中的 tasks，在 playbook 中使用時不需要指定絕對(absolutely) or 相對(relatively)路徑

8. 定義在 **roles/<font color='red'>x</font>/defaults/main.yml** 中的變數將會是使用該 role 時所取得的預設變數，

-------------------------------------------

Role 的變數宣告
===============

使用 Role 的好處在於利用到 Ansible 的自動化特性，自動將分類好的資訊放入 playbook 並執行，其中變數的處理便是如此。

以下以 role <font color='red'>**common**</font> 為範例：

### 預設值

假設 Role(**common**) 的設定中若包含了預設的變數，可以定義在 **roles/common/defaults/main.yml**：

``` yaml
---
# file: roles/x/defaults/main.yml
# if not overridden in inventory or as a parameter, this is the value that will be used
http_port: 80
```

預設變數是可以透過 inventory 或是參數將其取代成其他值。

### 不可改變的值

若是有特定變數是不准透過 inventory 改變的，則是要放在 **roles/x/vars/main.yml** 中：

``` yaml
---
# file: roles/x/vars/main.yml
# this will absolutely be used in this role
http_port: 80
```

如此一來就不能透過 inventory 的方式改變變數值；但若是真的要改變變數值，還是可以在使用 role 時傳入參數去取代：

``` yaml
roles:
   - { name: apache, http_port: 8080 }
```

-------------------------------------------

Role 相依性
===========



Roles + Playbook = reuse

roles, which is a way of collecting playbooks together so they could potentially be reused

Ansible Galaxy


-------------------------------------------


參考資料
========

- [Playbook Roles and Include Statements — Ansible Documentation](http://docs.ansible.com/playbooks_roles.html)

- [Tags — Ansible Documentation](http://docs.ansible.com/playbooks_tags.html)

- [Access value through index number from ansible gather facts variable - Stack Overflow](http://stackoverflow.com/questions/24320800/access-value-through-index-number-from-ansible-gather-facts-variable)

- [http://techieroop.com/variable-of-variable-in-ansible-playbook/#.VWE_vUbwnTw](http://techieroop.com/variable-of-variable-in-ansible-playbook/#.VWE_vUbwnTw)

- [Creating Ansible Roles from Scratch: Part 1 | Azavea Labs](http://www.azavea.com/blogs/labs/2014/10/creating-ansible-roles-from-scratch-part-1/)
