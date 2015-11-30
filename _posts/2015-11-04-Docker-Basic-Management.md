---
layout: post
title:  "Docker 學習筆記 - 基本管理"
description: "此文章記錄學習 docker 的過程，此篇為基礎管理的部份，包含資料管理、基礎網路應用、container 之間互連的方式 ... 等"
date: 2015-11-04 10:36:00
published: true
comments: true
categories: [docker]
tags: [Docker, Container, Virtualization]
---

資料管理
========

### Data Volumes

掛載主機上的指定目錄(/tmp/webapp)到 container 的指定目錄(/webapp)上：

```bash
# -P: 開放 container 所有開啟的 port
$ sudo docker run -d -P --name web -v /tmp/webapp:/webapp training/webapp python app.py

# 若要掛載 data volume 為 read-only，加入 :ro 即可
$ sudo docker run -d -P --name web -v /tmp/webapp:/webapp:ro training/webapp python app.py
```

也可以掛載單一檔案：(**因為檔案 inode 會被修改的關係，但建議還是掛載目錄為主**)

```bash
$ sudo docker run --rm -ti -v ~/.bash_history:/root/.bash_history ubuntu /bin/bash
```


### Data Volume Container

Data Volume Container 的功能其實就只是把自身的 Data Volume 提供出來給其他 container 使用。

首先，以 ubnutu image 在背景先啟動一個 Data Volume Container，名稱為 **shared_data**：

```bash
$ sudo docker run -itd -v /shared_data --name shared_data ubuntu
```

接著啟動其他容器，並指定掛載 **shared_data** 中的 data volume：

```bash
$ sudo docker run -idt --volumes-from shared_data --name worker1 ubuntu
$ sudo docker run -idt --volumes-from shared_data --name worker2 ubuntu
```

在每個 container 內分別對 shared_data 目錄新增檔案 & 驗證

```bash
$ sudo docker exec -ti worker2 touch /shared_data/data_from_worker2
$ sudo docker exec -ti worker1 touch /shared_data/data_from_worker1

# 驗證 shared data
$ sudo docker exec -ti shared_data ls -l /shared_data
total 0
-rw-r--r-- 1 root root 0 Nov  3 14:00 data_from_worker1
-rw-r--r-- 1 root root 0 Nov  3 14:00 data_from_worker2
```

> 刪除掛載 data volume 的 container，並不會刪除 data volume；必須等到最後一個掛載 data volume 的 container 被移除後，才會消失。

> 必須注意的是，若要正確移除 data volume，就必須使用 <font color='red'>**docker rm -v**</font> 的方式移除最後一個 container 才行。

### 備份 Data Volume

備份 Data Volume Container 的 data volume 到目前所在目錄中，並命名為 **backup.tgz**：

```bash
$ sudo docker run --rm --volumes-from shared_data -v $(pwd):/backup --name backup_worker ubuntu tar -czvp -f /backup/backup.tgz /shared_data
```

驗證備份檔案：

```bash
$ $ tar -ztv -f backup.tgz
drwxr-xr-x root/root         0 2015-11-03 14:00 shared_data/
-rw-r--r-- root/root         0 2015-11-03 14:00 shared_data/data_from_worker1
-rw-r--r-- root/root         0 2015-11-03 14:00 shared_data/data_from_worker2
```

### 還原 Data Volume

有了備份檔 backup.tgz 後，可以用以下方式還原：

```bash
# 建立新的 Data Volume Container
$ sudo docker run -itd -v /shared_data --name shared_data_2 ubuntu /bin/bash

# 確認新的 Data Volume Container 目前沒有資料
$ sudo docker exec -ti shared_data_2 ls -al /shared_data
total 8
drwxr-xr-x  2 root root 4096 Nov  3 14:16 .
drwxr-xr-x 39 root root 4096 Nov  3 14:16 ..

# 新建立一個 container 作為還原之用，並執行還原工作
$ sudo docker run --rm --volumes-from shared_data_2 -v $(pwd):/restore ubuntu tar -xzv -f /restore/backup.tgz
shared_data/
shared_data/data_from_worker1
shared_data/data_from_worker2

# 確認新的 Data Volume Container 已經有還原後的資料
$ sudo docker exec -ti shared_data_2 ls -al /shared_data
total 8
drwxr-xr-x  2 root root 4096 Nov  3 14:00 .
drwxr-xr-x 39 root root 4096 Nov  3 14:16 ..
-rw-r--r--  1 root root    0 Nov  3 14:00 data_from_worker1
-rw-r--r--  1 root root    0 Nov  3 14:00 data_from_worker2
```

--------------------------------------------

網路基礎配置
===========

### 設定 port mapping

#### 1、不指定 port

```bash
# 透過 -P 參數，會從 host 中隨機取一個 port 出來使用
$ sudo docker run -d -P training/webapp python app.py
7ce8b8495f04f278e1444bf9b38fd55b6eec569bf9c2c8b9617772fcd4b4a683

# 可看出 port 的對映關係為 container:5000 <--> host:32771
$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
7ce8b8495f04        training/webapp     "python app.py"     8 seconds ago       Up 8 seconds        0.0.0.0:32771->5000/tcp   boring_perlman
```

#### 2、指定 port

可以指定一個 port：

```bash
$ sudo docker run -d -p 5000:5000 training/webapp python app.py
26ab249017364ee2f3482ad3f429707d06688c64746a278f682ed81baeee1472

$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
26ab24901736        training/webapp     "python app.py"     6 seconds ago       Up 5 seconds        0.0.0.0:5000->5000/tcp    boring_mestorf
```

也可以同時指定多個 port：

```bash
$ sudo docker run -d -p 5000:5000 -p 3000:80 training/webapp python app.py
719bc6b51ba2e46cdd17837ece2ab39873de4aeba0e735bbb08a4140a7cb2641

$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                          NAMES
719bc6b51ba2        training/webapp     "python app.py"     5 seconds ago       Up 4 seconds        0.0.0.0:5000->5000/tcp, 0.0.0.0:3000->80/tcp   elegant_poitras
```

#### 3、對應到特定位址

指定特定的 ip address & port：

```bash
$ sudo docker run --name web -d -p 127.0.0.1:5000:5000 training/webapp python app.py
45794faef43c8be5c378808c150d946ed4bb1fef4877ece19740d4805ee8a2e6

$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                      NAMES
45794faef43c        training/webapp     "python app.py"     5 seconds ago       Up 4 seconds        127.0.0.1:5000->5000/tcp   web
```

指定特定的 ip address，但不指定 port：

```bash
$ sudo docker run --name web -d -p 127.0.0.1::5000 training/webapp python app.py
214a0a32a2275517874f2c50ea7401e096f1b3872122f09c0437ab78d1590724

$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                       NAMES
214a0a32a227        training/webapp     "python app.py"     3 seconds ago       Up 2 seconds        127.0.0.1:32768->5000/tcp   web
```

也可指定不同協定，例如 udp：

```bash
$ sudo docker run --name web -d -p 127.0.0.1:5000:5000/udp training/webapp python app.py
66fcf5f6f9f6f7b0ea3b46cbf49b7bdb418a4c5e321bc8561c0d180087d2a9d6

$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                NAMES
66fcf5f6f9f6        training/webapp     "python app.py"     8 seconds ago       Up 7 seconds        5000/tcp, 127.0.0.1:5000->5000/udp   web
```

若要查詢 container port 的對應狀況，可用以下指令：

```bash
$ sudo docker port web 5000/udp
```

### 建立 container 間的連結

```bash
# 建立 db container
$ sudo docker run --name db -d training/postgres
a134586d0eac577584fc11c49e54006866521527568bc0a0be59ee2f5e40c969

# 建立 web container，並 link 到 db container (使用 --link 參數，參數值的格式 "container_name:alias")
$ sudo docker run --name web -d -P --link db:db_link training/webapp python app.py
3fb2b15517bf7b323b5bd5027679957dc0ed51ae20e3d40fbe9012db6fd2b4a8

$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                     NAMES
3fb2b15517bf        training/webapp     "python app.py"          5 seconds ago       Up 4 seconds        0.0.0.0:32775->5000/tcp   web
a134586d0eac        training/postgres   "su postgres -c '/usr"   14 seconds ago      Up 14 seconds       5432/tcp                  db

# 查詢　web & db 之間的連結
$ sudo docker inspect -f "{{ .HostConfig.Links }}" web
[/db:/web/db_link]

# 連結之後，可取得 db container 相關的環境變數
$ sudo docker exec web env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=3fb2b15517bf
DB_LINK_PORT=tcp://172.17.0.33:5432
DB_LINK_PORT_5432_TCP=tcp://172.17.0.33:5432
DB_LINK_PORT_5432_TCP_ADDR=172.17.0.33
DB_LINK_PORT_5432_TCP_PORT=5432
DB_LINK_PORT_5432_TCP_PROTO=tcp
DB_LINK_NAME=/web/db_link
DB_LINK_ENV_PG_VERSION=9.3
HOME=/root

# 同時也新增了 DNS 的資訊到 /etc/hosts 中
$ sudo docker exec web cat /etc/hosts
172.17.0.34	3fb2b15517bf
127.0.0.1	localhost
...... (略)
172.17.0.33	db_link a134586d0eac db
172.17.0.33	db
172.17.0.33	db.bridge
172.17.0.34	web
172.17.0.34	web.bridge


# 從 web container 嘗試 ping db container
$ sudo docker exec web ping -c 3 db
PING db_link (172.17.0.33) 56(84) bytes of data.
64 bytes from db_link (172.17.0.33): icmp_seq=1 ttl=64 time=0.037 ms
64 bytes from db_link (172.17.0.33): icmp_seq=2 ttl=64 time=0.042 ms
64 bytes from db_link (172.17.0.33): icmp_seq=3 ttl=64 time=0.038 ms
... (略)
```

--------------------------------------------

建立 wordpress + mysql 應用 (使用 link)
======================================

首先，先建立 mysql container

```bash
# 建立 mysql container，名稱為 mysqpwp
# 設定 root password = wpdocker
$ docker run --name mysqlwp -d -e MYSQL_ROOT_PASSWORD=wpdocker mysql
```

接著建立 wordpress container，並使用 link 與 mysql container 連結

```bash
# 建立 wordpress container，名稱為 wordpress
# 將 container:80 對映到 host:1080
$ docker run --name wordpress -d --link mysqlwp:mysql -p 1080:80 wordpress

# 查詢 wordpress 與 mysql 的連結關係
$ docker inspect -f "{{.HostConfig.Links}}" wordpress
[/mysqlwp:/wordpress/mysql]

# 查詢在 wordpress 上的環境變數(可以發現多了 MYSQL 的部分)
$ docker exec wordpress env
.....(略)
MYSQL_PORT=tcp://172.17.0.2:3306
MYSQL_PORT_3306_TCP=tcp://172.17.0.2:3306
MYSQL_PORT_3306_TCP_ADDR=172.17.0.2
MYSQL_PORT_3306_TCP_PORT=3306
MYSQL_PORT_3306_TCP_PROTO=tcp
MYSQL_NAME=/wordpress/mysql
MYSQL_ENV_MYSQL_ROOT_PASSWORD=wpdocker
MYSQL_ENV_MYSQL_MAJOR=5.7
MYSQL_ENV_MYSQL_VERSION=5.7.9-1debian8
.....(略)
```


--------------------------------------------


關於使用 --link 需要注意的事項
==========================

從上面的範例可以看出，**--link** 在串接兩個 container 時的確很好用，但這僅限於在同一個實體機器上，若是在 multiple hosts 的狀況下，就需要其他的 solution 來協助處理跨 host 的 container 溝通問題了!


--------------------------------------------

參考資料
========

- [博客來-Docker入門與實戰](http://www.books.com.tw/products/0010676115)
