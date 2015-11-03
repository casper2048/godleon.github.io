---
layout: post
title:  "Docker 學習筆記 - 基礎概念"
description: "此文章記錄學習 docker 的過程，此篇為基礎概念的部份，包含 image, container, 以及 registry ... 等"
date: 2015-11-03 15:27:00
published: true
comments: true
categories: [docker]
tags: [Docker, Container, Virtualization]
---

安裝 docker
===========

docker 需要 3.13 版以上的 linux kernel

- [docker Installation on Ubuntu](https://docs.docker.com/installation/ubuntulinux/)

### docker group

docker engine 安裝之後，會在系統中新增一個名為 **docker** 的 group，只有在這個 group 內的 user 才可以存取 docker daemon 所使用的 Unix socket。

因此我們可以將特定使用者加入 docker group，如此一來一般 user 所產生的 container 才可以透過 docker daemon 的 Unix socket 連上網路。

若沒這麼作，就只能透過 sudo 的方式以 root 的身分所產生的 container 才能上網。

可透過以下指令將 user 加入 docker group：

```bash
# 以下假設 user name = ubuntu
sudo usermod -aG docker ubuntu
```

### UFW (Uncomplicated Fiewwall)

若啟用了 UFW，預設會將 traffic forwarding 的功能關閉，要讓 container 可以上網就必須調整設定，可參考一下網頁：

- [Installation on Ubuntu - Enable UFW forwarding](https://docs.docker.com/installation/ubuntulinux/#enable-ufw-forwarding)

### 設定 dokcer container 所使用的 DNS

```bash
# 加入 DNS 設定
$ sudo sh -c "echo 'DOCKER_OPTS=\"--dns 8.8.8.8 --dns 8.8.4.4\"' >> /etc/default/docker"

# 重新啟動 docker daemon
$ sudo restart docker
```

--------------------------------------------

Image
=====

docker pull 指令相關預設值如下：

- registry：<font color='red'>**registry.hub.docker.com**</font>

- tag: <font color='red'>**latest**</font>

docker pull 指令格式：
> dokcer pull [registry/]NAME[:tag]


```bash
# 從 public repository 中取得最新的 ubuntu image
$ sudo docker pull ubuntu

# 上面的指令與下面相同(沒有指定 tag 預設就是 latest)
$ sudo docker pull ubuntu:latest

# 其實完整指令如下
$ sudo docker pull registry.hub.docker.com/ubuntu:latest
```

### 執行 docker container

```bash
# 使用指定映像檔(ubuntu)建立一個 container，並執行 bash 程式
# -i: Keep STDIN open even if not attached
# -t: Allocate a pseudo-TTY
$ sudo docker run -t -i ubuntu /bin/bash
```

### 查看映像檔資訊

```bash
# 檢視目前映像檔清單
$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu              latest              1d073211c498        8 days ago          187.9 MB
hello-world         latest              0a6ba66e537a        2 weeks ago         960 B

# 檢視 image id 為 1d07 的映像檔的詳細資料
$ sudo docker inspect 1d07

# 透過 -f 參數檢視指定的屬性
$ sudo docker inspect -f {{".Architecture"}} 1d07
$ sudo docker inspect -f {{".Architecture"}} ubuntu:latest
```

### 自定 tag

```bash
# 自訂 tag，多加上一組 new_tag 至 ubuntu:latest 映像檔中
$ sudo docker tag ubuntu:latest ubuntu:new_tag
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu              latest              1d073211c498        8 days ago          187.9 MB
ubuntu              new_tag             1d073211c498        8 days ago          187.9 MB
hello-world         latest              0a6ba66e537a        2 weeks ago         960 B
```

### 搜尋映像檔

```bash
$ sudo docker search pernona
NAME                                    DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
percona                                 Percona Server is a fork of the MySQL rela...   55        [OK]
paulczar/percona-galera                                                                 11                   [OK]
klevo/percona                           Percona MySQL server supporting replicatio...   3                    [OK]
internavenue/centos-percona                                                             1                    [OK]
nicescale/percona-mysql                 Percona MySQL for NiceScale, support conne...   1                    [OK]
```

### 刪除映像檔

```bash
# 刪除指定 image，但因為有多組 tag 指到同一個 image，因此只會移除 tag
$ docker rmi ubuntu:new_tag
Untagged: ubuntu:new_tag

$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu              latest              1d073211c498        8 days ago          187.9 MB
hello-world         latest              0a6ba66e537a        2 weeks ago         960 B

# 只剩一個 tag 指向 image，因此這次就會將 image 移除
$ docker rmi ubuntu:latest
Untagged: ubuntu:latest
Deleted: 1d073211c498fd5022699b46a936b4e4bdacb04f637ad64d3475f558783f5c3e
Deleted: 5a4526e952f0aa24f3fcc1b6971f7744eb5465d572a48d47c492cb6bbf9cbcda
Deleted: 99fcaefe76ef1aa4077b90a413af57fd17d19dce4e50d7964a273aae67055235
Deleted: c63fb41c2213f511f12f294dd729b9903a64d88f098c20d2350905ac1fdbcbba

$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
hello-world         latest              0a6ba66e537a        2 weeks ago         960 B
```

### 查詢(移除)目前的 container 資訊

```bash
# 使用 ubnutu:latest image 建立一個 container
$ sudo docker run ubuntu echo 'Hello, Docker!'

$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS               NAMES
d14d5f4426fb        ubuntu              "echo 'Hello, Docker!"   5 seconds ago       Exited (0) 5 seconds ago                       drunk_visvesvaraya
7de10caa6429        hello-world         "/hello"                 7 hours ago         Exited (0) 7 hours ago                         compassionate_pare
12dcafa7169f        hello-world         "/hello"                 7 hours ago         Exited (0) 7 hours ago                         ecstatic_carson

# 移除指定的 container
$ sudo docker rm d14
```

### 建立映像檔

#### 1. 以現有的 image 為基礎產生新的 image

```bash
# 以下以 ubuntu:latest 來作實驗
$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu              latest              1d073211c498        10 days ago         187.9 MB
hello-world         latest              0a6ba66e537a        2 weeks ago         960 B

# 使用 image 啟動一個 container 並進入 shell
$ sudo docker run -ti ubuntu:latest /bin/bash

# 在 container 中新增一個檔案
root@06ca958f3dc1:/# touch test
root@06ca958f3dc1:/# exit

# 透過 docker commit 建立一個新的 image
$ sudo docker commit -m "Add a new file" -a "Docker greenhand" 06ca test

# 可看到新的 image 已經產生
$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
test                latest              4b70ef66946f        7 seconds ago       187.9 MB
......
```

#### 2. 匯入外部預先建立好的 image

```bash
$ sudo docker import https://download.openvz.org/template/precreated/centos-7-x86_64.tar.gz centos:7.0

$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
centos              7.0                 fc0749668baf        About a minute ago   564.3 MB
.....
```

### 儲存 & 載入 image

以下透過 ssh 直接把 image 匯出並透過 ssh 匯入到遠端的 server

```bash
$ sudo docker save centos:7.0 | ssh USER@remote.server.ip docker load
```

--------------------------------------------

Container
=========

### 建立 container

> **docker run** = docker create + docker start

### 背景執行 container

#### 1、透過 -d 參數讓 container 在背景執行

```
$ sudo docker run -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 5; done"
f8fbc2b5ca4290c7dfc3d2be9661a62e2b67cc888f2ed83e1e94bcc2d965bea1

$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
f8fbc2b5ca42        ubuntu              "/bin/sh -c 'while tr"   9 seconds ago       Up 8 seconds                            adoring_pasteur

$ sudo docker logs f8f
hello world
hello world
....

# 停止執行 container
$ sudo docker stop f8f
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                       PORTS               NAMES
f8fbc2b5ca42        ubuntu              "/bin/sh -c 'while tr"   2 minutes ago       Exited (137) 4 seconds ago                       adoring_pasteur
```

#### 2、使用 docker exec 下指令到背景執行中的 container

```bash
$ sudo docker run -itd ubuntu
febde594d3276abd557e13bcbfa03ffa87a918099885dce1b49b924ebd6bdb92

$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
febde594d327        ubuntu              "/bin/bash"         7 seconds ago       Up 6 seconds                            gloomy_turing

$ sudo docker exec -ti feb pwd
/

$ sudo docker exec -ti feb ls -al
total 72
drwxr-xr-x  32 root root 4096 Nov  2 14:32 .
drwxr-xr-x  32 root root 4096 Nov  2 14:32 ..
-rwxr-xr-x   1 root root    0 Nov  2 14:32 .dockerenv
-rwxr-xr-x   1 root root    0 Nov  2 14:32 .dockerinit
drwxr-xr-x   2 root root 4096 Oct 21 04:46 bin
.....

$ sudo docker exec -ti feb whoami
root
```

### 刪除容器

```bash
$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
febde594d327        ubuntu              "/bin/bash"         4 minutes ago       Up 4 minutes                            gloomy_turing

# 刪除執行中的 container(未加 -f 無法刪除)
$ sudo docker rm feb
Error response from daemon: Cannot destroy container feb: Conflict, You cannot remove a running container. Stop the container before attempting removal or use -f
Error: failed to remove containers: [feb]

# 強制刪除執行中的 container
$ sudo docker rm -f feb
feb
```

### 匯出 & 匯入 container

這裡介紹如何匯出 container 並透過 ssh 匯入到遠端的 server 上：

```bash
# 背景執行一個 container
$ sudo docker run -itd ubuntu
a2ca9da2cfe618c1945a31fd0e8d7954e8fad87e88b1a6e29b245cb00e07afa9

# export container & impoer to remote server by ssh
$ sudo docker export a2c | ssh USER@remote.server.ip sudo docker import - test/ubuntu:1.0

# 檢視 remote server
[REMOTE_SERVER]$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
test/ubuntu         1.0                 bf5319399929        About a minute ago   187.7 MB
```

container 匯入後其實不會執行，只是會變成一個 image 供使用而已。

--------------------------------------------

Repository
==========

### 建立 private registry

使用官方提供的 <font color='red'>**registry:2**</font> 安裝 private registry，registry 預設使用的 port 為 tcp 5000

```bash
# port forward 到外部 server 的 port 5000
$ sudo docker run -d --restart=always -p 5000:5000 --name registry registry:2

# 檢視 container status
$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS                    NAMES
e988dc62ce42        registry:2          "/bin/registry /etc/d"   3 minutes ago       Up 3 minutes               0.0.0.0:5000->5000/tcp   registry

# 從 docker hub 下載 ubuntu image
$ sudo docker pull ubuntu

# 對此 image 新增 tag 資訊，指定上面已經建立好的 registry
$ sudo docker tag ubuntu localhost:5000/ubuntu

$ sudo docker push localhost:5000/ubuntu
The push refers to a repository [localhost:5000/ubuntu] (len: 1)
1d073211c498: Image successfully pushed
5a4526e952f0: Image successfully pushed
99fcaefe76ef: Image successfully pushed
c63fb41c2213: Image successfully pushed
latest: digest: sha256:30550d609f2de731f61ad154ee50b267bec31ca373eaf424b8adbc836dc17f70 size: 7731
```

接著我們就可以從 private registry 下載 image 囉：

```bash
$ sudo docker pull localhost:5000/ubuntu
```

最後移除 private registry container

```bash
$ sudo docker stop registry

$ sudo docker rm -v registry
```

--------------------------------------------

參考資料
========

### 基本概念

- [初探Docker - Docker 跟 LXC 以及一般Hypervisor有何差別？ - 阿貝好威的實驗室](http://lab.howie.tw/2014/08/docker-docker-lxc-hypervisor.html)

- [Flockport - LXC vs Docker](https://www.flockport.com/lxc-vs-docker/)

- [Flockport - LXC vs LXD vs Docker - Making sense of the rapidly evolving container ecosystem](https://www.flockport.com/lxc-vs-lxd-vs-docker-making-sense-of-the-rapidly-evolving-container-ecosystem/)


### 實作相關

- [UbuntuUpdates - PPA: Docker](http://www.ubuntuupdates.org/ppa/docker)

- [Marek Goldmann | Connecting Docker containers on multiple hosts](https://goldmann.pl/blog/2014/01/21/connecting-docker-containers-on-multiple-hosts/)

- [Installation Docker Engine on Ubuntu](https://docs.docker.com/installation/ubuntulinux/)

- [Deploying a registry server](https://docs.docker.com/registry/deploying/)
