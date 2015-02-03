---
layout: post
title:  "[Docker] 快速建立 IPython Notebook 環境"
date:   2014-11-23 06:30:00
comments: true
categories: [docker]
tags: [Docker, Python]
---

上周參加讀書會有朋友提供一個 DockerHub 連結，表示目前已經有人將 IPython Notebook & SciPy Stack 包再一起發布在 DockerHub 上，可自行取用....

DockerHub 連結如下：
[IPython notebook server with the full SciPy stack + more](https://registry.hub.docker.com/u/ipython/scipyserver/)

使用方式真的很簡單，在安裝好 Docker 環境的電腦，執行以下命令：

``` bash
# 其中 MakeAPassword 是可以自行決定的登入密碼
$ docker run -d -p 443:8888 -e "PASSWORD=MakeAPassword" ipython/scipyserver
```

Docker 執行起來後，使用者會位於虛擬環境中的 `notebooks` 目錄，因此為了要保留在 notebook 上所輸入的資料，我們可以將外部的目錄與 notebooks 進行連結，這樣 docker 停止運作後就可以將原本的資料保留住：

``` bash
$ docker run -d -p 443:8888 -e "PASSWORD=MakeAPassword" -v /external/dir/path:/notebooks ipython/scipyserver
```

第一次執行會將相關的 image 下載回來，會花點時間，之後都會很快了!

執行完畢後只要連結到 **https://your_docker_server_ip**，並輸入 `MakeAPassword` 即可登入 IPython Notebook 了!