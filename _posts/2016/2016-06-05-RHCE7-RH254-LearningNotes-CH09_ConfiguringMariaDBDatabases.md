---
layout: post
title:  "[RHCE7] RH254 Chapter 9 Configuring MariaDB Databases Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 9 Configuring MariaDB Databases 留下的內容"
date: 2016-06-05 04:10:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH254]
---

9.1 Installing MariaDB
======================

### 安裝 MariaDB

- 安裝 MariaDB：`sudo yum -y groupinstall mariadb mariadb-client`

- 主要設定檔位於 `/etc/my.cnf`，也可把設定檔放在 `/etc/my.cnf.d` 目錄下

### 加強安全性

增加安全性：`sudo mysql_secure_installation`
- 設定 root 密碼
- 移除 localhost 之外的 root 權限
- 移除 anonymous-user 帳號
- 移除 test 資料庫

### Networking

- default port = `3306`

設定檔 `/etc/my.cnf`：

- `bind-address`：決定開放的程度，`::` 表示開放所有 ipv4 & ipv6 位址，`空白` or `0.0.0.0` 表示開放所有 ipv4 位址

- `skip-network`：若設定為 `1`，只有 localhost 可以連

開啟防火牆：

```bash
$ sudo firewall-cmd --permanent add-service=mysql
$ sudo firewall-cmd --reload
```

----------------------------------------------------

9.3 Managing Database Users and Access Rights
=============================================

Account 設定範例：(**指定從哪來的使用者**)

| Account | Description |
|---------|-------------|
| `mobius@'localhost'` | 使用者 mobius 可從 localhost 連資料庫 |
| `mobius@'192.168.1.5'` | 使用者 mobius 可從 192.168.1.5 連資料庫 |
| `mobius@'192.168.1.%'` | 使用者 mobius 可從 192.168.1.0/24 連資料庫 |
| `mobius@'%'` | 使用者 mobius 可從任何地方連資料庫 |
| `mobius@'2000:472:18::a21'` | 使用者 mobius 可從 2000:472:18::a21 連資料庫 |

Grant 設定範例：(**設定使用者在哪個資料庫的那些 table，有多大的權限**)

| Grant | Description |
|-------|-------------|
| GRANT SELECT ON database.table TO username@hostname | 有資料庫 **database** 中 **table** table 的 **select** 權限 |
| GRANT SELECT ON database.* TO username@hostname | 有資料庫 **database** 中 **所有 table** 的 **select** 權限 |
| GRANT SELECT ON *.* TO username@hostname | 有所有資料庫的 **select** 權限 |
| GRANT CREATE, ALTER, DROP ON database.* TO username@hostname | 有資料庫 **database** 中 **所有 table** 的 **create/alter/drop table** 權限 |
| GRANT ALL PRIVILEGES ON *.* TO username@hostname | 可對任何資料庫的任何 table 進行任何操作(幾乎等同管理者) |

> 使用 grant 設定使用者權限之前，若使用者原本不存在，要透過 `CREATE USER` 的語法先建立使用者帳號

> 若 `GRANT` 換成 `REVOKE` 關鍵字，則表示移除指定權限

> 權限設定完後，最後要輸入 `flush privileges;` 才會將權限設定寫入資料庫

最後，要確認目前的設定，可使用 `shwo grants for username@hostname;` 進行查詢，例如：`shwo grants for mary@'%';`

----------------------------------------------------

9.4 Creating and Restoring MariaDB Backups
==========================================

## 9.4.1 Creating a backup

MariaDB 支援的備份方式有兩種：

### Logical Backups

- 把資料 dump 出來，可攜性高

- 僅有 database schema & data，不會帶上更多的額外設定(例如：log & 權限設定)

- 備份速度較慢，因為要存取整個完整的 database

- 可在 server online 時執行備份

### Physical Backups

- database 的原生拷貝 (從 **/var/lib/mysql** 目錄中，以一個資料庫一個檔案的方式把 database 備份出來)

- 不同版本的 database 可能會不相容，可攜性低

- 備份的資料相對較小，速度快

- 備份資料可包含 log & configuration

- 需要在 server offline 或是 table 都被 lock 住的前提下才可以執行備份

## 9.4.2 Performing a logical backup

備份範例：

```bash
# 使用 root 身分備份
# 備份 inventory 資料庫到 /backup/inventory.dump
$ mysqldump -u root -p inventory > /backup/inventory.dump

# 使用 root 身分備份
# 備份所有資料庫到 /backup/mariadb.dump
$ mysqldump -u root -p all-databases > /backup/mariadb.dump
```

其他參數：

- `--add-drop-tables`：在 create table 前加上 drop table

- `--no-data`：只備份 schema

- `--lock-all-tables`：設定備份時資料庫為唯讀狀態

- `--add-drop-database`：在 create database 之前加上 drop database

----------------------------------------------------

Practice: Restoring a MariaDB Database from Backup
==================================================

## 目標

- 從備份檔 `/home/student/inventory.dump` 中還原資料庫 `inventory`

- 資料庫的帳號為 root 加上空白密碼

## 實作過程

```bash
# 建立資料庫
$ mysql -u root
MariaDB [(none)]> create database inventory;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> exit;
Bye

# 還原資料庫
$ mysql -u root inventory < /home/student/inventory.dump
[student@server0 ~]$ mysql -u root
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 9
Server version: 5.5.35-MariaDB MariaDB Server

Copyright (c) 2000, 2013, Oracle, Monty Program Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use inventory;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [inventory]> show tables;
+---------------------+
| Tables_in_inventory |
+---------------------+
| category            |
| manufacturer        |
| product             |
+---------------------+
3 rows in set (0.00 sec)

MariaDB [inventory]> select * from catagory;
MariaDB [inventory]> select * from category;
+----+------------+
| id | name       |
+----+------------+
|  1 | Networking |
|  2 | Servers    |
|  3 | Ssd        |
+----+------------+
3 rows in set (0.00 sec)
```

----------------------------------------------------

Lab: Configuring MariaDB Databases
==================================

## 目標

1、安裝 & 啟動 MariaDB 服務

1、還原 `legacy` 資料庫，備份檔位於 `/home/student/mariadb.dump`

2、建立使用者帳號

| User | Password | Privileges |
|------|----------|------------|
| mary | mary_password | 在資料庫 legacy 上的所有 table 有 select 權限 |
| legacy | legacy_password | 在資料庫 legacy 上的所有 table 有 insert/update/delete 權限 |
| report | report_password | 在資料庫 legacy 上的所有 table 有 select 權限 |

3、新增資料到 `manufacturer` 表格

| Name | Seller | Phone number |
|------|--------|--------------|
| HP | Joe Doe | +1 (432) 754-3509 |
| Dell | Luke Skywalker | +1 (431) 219-4589 |
| Lenovo | Darth Vader | +1 (327) 647-6784 |

## 實作過程

```bash
# 安裝套件
$ sudo yum -y groupinstall mariadb mariadb-client

# 啟動 MariaDB 服務 & 防火牆
$ sudo systemctl enable mariadb.service
$ sudo systemctl restart mariadb.service
$ sudo firewall-cmd --permanent --add-service=mysql
$ sudo firewall-cmd --reload

# 建立 legacy 資料庫並還原備份資料
$ mysql -u root
MariaDB [(none)]> create database legacy;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> exit
Bye
[student@server0 ~]$ mysql -u root legacy < /home/student/mariadb.dump

# 設定使用者存取資料庫相關權限 (注意! 一定要先 CREATE USER!)
$ mysql -u root
MariaDB [(none)]> create user mary@'%' identified by 'mary_password';
MariaDB [(none)]> create user legacy@'%' identified by 'legacy_password';
MariaDB [(none)]> create user report@'%' identified by 'report_password';
MariaDB [(none)]> grant select on legacy.* to mary@'%';
MariaDB [(none)]> grant select,insert,update,delete on legacy.* to legacy@'%';
MariaDB [(none)]> grant select on legacy.* to report@'%';
MariaDB [(none)]> flush privileges;

# 新增資料
MariaDB [(none)]> use legacy;
MariaDB [legacy]> describe manufacturer;
+--------------+--------------+------+-----+---------+----------------+
| Field        | Type         | Null | Key | Default | Extra          |
+--------------+--------------+------+-----+---------+----------------+
| id           | int(11)      | NO   | PRI | NULL    | auto_increment |
| name         | varchar(100) | NO   |     | NULL    |                |
| seller       | varchar(100) | YES  |     | NULL    |                |
| phone_number | varchar(17)  | YES  |     | NULL    |                |
+--------------+--------------+------+-----+---------+----------------+
4 rows in set (0.00 sec)
MariaDB [legacy]> insert into manufacturer(name, seller, phone_number) values('HP', 'Joe Doe', '+1 (432) 754-3509');
MariaDB [legacy]> insert into manufacturer(name, seller, phone_number) values('Dell', 'Luke Skywalker', '+1 (431) 219-4589');
MariaDB [legacy]> insert into manufacturer(name, seller, phone_number) values('Lenovo', 'Darth Vader', '+1 (327) 647-6784');
MariaDB [legacy]> exit
```
