---
layout: post
title:  "Docker 學習筆記 - Hello Dockerfile"
description: "此文章記錄學習 docker 的過程，此篇文章介紹 Dockerfile 的入門寫法 ... 等"
date: 2015-11-27 10:35:00
published: true
comments: true
categories: [docker]
tags: [Docker, Container, Virtualization]
---

撰寫第一個 Dockerfile
====================

1、建立 Dockerfile & app.py
---------------------------

Dockerfile 固定就需要將檔案命名為 <font color='red'>**Dockerfile**</font>

```bash
FROM ubuntu:latest

MAINTAINER Leon Tseng <godleon@gmail.com>

RUN apt-get update
RUN apt-get -y install python3
RUN apt-get -y install python3-pip
RUN apt-get clean all

RUN pip3 install flask

RUN mkdir -p /flask
COPY app.py /flask

EXPOSE 5000

CMD ["/usr/bin/python3", "/flask/app.py"]
```

另外建立一個檔案 <font color='red'>**app.py**</font>：

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000, debug=True)
```

2、產生 image 檔
----------------

透過以下命令將 Dockerfile 建立成 image，並命名為 **python:flask**

```bash
$ docker build -t python:flask .
```

3、產生 docker instance
-----------------------

```bash
$ docker run --name python_flask -td -p 5000:5000 python:flask

$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
933ec051c7a7        python:flask        "/usr/bin/python3 /fl"   21 seconds ago      Up 20 seconds       0.0.0.0:5000->5000/tcp   python_flask
```

4、host 與 container 互傳資料
-----------------------------

這部分其實跟上面的 Dockerfile 沒甚麼關係，只是剛好可以一併提到，若是建立 container 時沒有使用 <font color='red'>**-v**</font> 參數掛載 host 目錄，還是可以用以下的命令可以將資料從 container 中複製出來：

```bash
# 其中 python_flask 是 container 名稱
$ docker cp python_flask:/flask/app.py .
```

當然也可以把 host 資料放進 container 中：

```bash
$ touch file_in_host

$ docker cp file_in_host python_flask:/flask

$ docker exec python_flask ls /flask
app.py
file_in_host
```

-----------------------------------------------------------

優化 Dockerfile 的建議
=====================

1. 為了實現 decoupled & scalable application，讓每個 container 只處理一個 process，並透過 container linking 或是其他 container-networking 相關的技術將 container 所提供的服務串起來。

2. container 不會一直存在，一般都是短暫存在且會 stop & restart

3. 為了需求變更而去變更 image 不是好習慣，應該透過 runtime configuration & docker data volume 的方式來滿足需求變更的情況

4. 使用 <font color='red'>**.dockerignore**</font> 的設計，可以避免不需要 or 敏感的資料在 docker image building 的過程中被加進去

5. 儘量使用來自 Docker Hub 的官方版 image，因為這些 image 直接由開發該 project 的人員維護，是最簡潔穩定的版本

6. 減少 image 的 layer 數量才有辦法運用到 image cache 的優勢

以上面的到的範例來說，可以進行以下簡單優化：

### 1、減少 image 的 layer 數量

盡量減少 RUN 的命令數量，因此原本的內容：

```
FROM ubuntu:latest
...(略)
RUN apt-get -y install python3
RUN apt-get -y install python3-pip
...(略)
```

可以改成下面的方式：

```
FROM ubuntu:latest
...(略)
RUN apt-get -y install python3
RUN apt-get -y install python3-pip
...(略)
```

### 2、盡量使用 Docker Hub 的官方版 Image

原本的使用 ubuntu:latest 為基底來安裝：

```bash
FROM ubuntu:latest
...(略)
RUN apt-get -y install python3
```

直接改用已經裝好 python 的 image：

```bash
FROM python:latest
...(略)
```

這樣連 python3 都不用自己裝了....(因此儘量利用已經存在的官方版 image 來設計 Dockerfile 會是個較為推荐的作法)

-----------------------------------------------------------

參考資料
=======

- [Best practices for writing Dockerfiles](http://docs.docker.com/engine/articles/dockerfile_best-practices/)

- [Dockerfile reference](http://docs.docker.com/engine/reference/builder/)

- [Dockerfile reference - .dockerignore file](http://docs.docker.com/engine/reference/builder/#dockerignore-file)

- [撰寫一份符合需求的 Dockerfile](http://cepave.com/how-to-write-dockerfile/)
