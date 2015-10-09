---
layout: post
title:  "[OpenStack] 使用 Rally 進行 Benchmark"
description: "此文章介紹如何安裝 & 使用 Rally，並對 OpenStack 進行 Benchmark 的工作"
date: 2015-10-09 11:30:00
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Container, Docker, Rally]
---

在此環境中，我們以 docker 作為安裝測試的環境：

安裝 docker
==========

```bash
$ sudo wget -qO- https://get.docker.com/ | sh
```

安裝完後可用以下指令驗證是否安裝成功：

```bash
$ sudo docker run hello-world
```

-----------------------------------

在 docker 中安裝 Rally
=====================

```bash
$ sudo mkdir /rally_container

# 將目錄權限設定給 Rally user(uid:65500)
$ sudo chown 65500 /rally_container

# 將上面的目錄掛載到 /home/rally 家目錄
$ sudo docker run -it -v /rally_container:/home/rally rallyforge/rally /bin/bash
```

-----------------------------------

設定執行環境
==========

> 以下指令皆在 docker container 內執行

命名為 **<font color='red'>admin-openrc.sh</font>**：

```bash
#!/bin/bash

export OS_VOLUME_API_VERSION=2
export OS_AUTH_URL=http://<YOUR_KEYSTONE_ENDPOINT_IP>:5000/v2.0
export OS_TENANT_NAME="admin"
export OS_USERNAME="admin"
export OS_PASSWORD="YOUR_ADMIN_PASSWORD"
```

```bash
# 匯入環境設定檔 for OpenStack
root@9ac5$ source admin-openrc.sh

# 建立 rally DB
root@9ac5$ rally-manage db recreate

# 產生 OpenStack deployment configuration 給 rally
root@9ac5$ rally deployment create --fromenv --name=existing

# 檢查 OpenStack 的服務是否正常
root@9ac5$ rally deployment check
```

-----------------------------------

開始第一個 Rally 測試
===================

測試之前要先確認 tenant quota 的問題，避免設計出因為 quota 而 fail 的 task template：

```bash
$ nova quota-defaults
+-----------------------------+-------+
| Quota                       | Limit |
+-----------------------------+-------+
| instances                   | 10    |
| cores                       | 20    |
| ram                         | 51200 |
| floating_ips                | 10    |
| fixed_ips                   | -1    |
| metadata_items              | 128   |
| injected_files              | 5     |
| injected_file_content_bytes | 10240 |
| injected_file_path_bytes    | 255   |
| key_pairs                   | 100   |
| security_groups             | 10    |
| security_group_rules        | 20    |
+-----------------------------+-------+
```

從上面可以看出，每個 tenant 的 instance quota 只有 10，vCore 可用數量只有 20，因此在設計 task template 時就必須注意一下，否則還沒測試到硬體極限
，反而因為 quota 限制造成工作無法正常執行，那就不是我們想要的結果了!

在 Rally 中可以測試的 [plugin](https://github.com/openstack/rally/tree/master/samples/tasks/scenarios) 很多，不論是 Nova / Cinder / Glance / Keystone / Neutron .... 等等都有很多 plugin 可以進行測試。

每個 scenario 也都不太一樣，使用者可以根據自己的測試需要挑選合適的 plugin 來進行測試。

以下測試最基本的 NOVA start & delete VM，我們挑選 <font color='red'>**boot-and-delete**</font> 來進行第一個測試，這個 scenario 執行以下工作：

1. 建立 instance (ubuntu m1.medium)

2. 開機

3. 關機

4. 刪除 instance

以下是我們所使用的 task scenario：

```yaml
{% set flavor_name = flavor_name or "m1.medium" %}
{% set image_name = "ubuntu-trusty-server-amd64-qcow2" %}
{% set instance_count = 5 %}

---
NovaServers.boot_and_delete_server:
  - args:
      flavor:
          name: "{{ flavor_name }}"
      image:
          name: {{ image_name }}
      force_delete: false
    runner:
      type: "constant"
      times: {{ instance_count }}
      concurrency: {{ instance_count }}
    context:
      users:
        tenants: 3
        users_per_tenant: 2
      network:
        start_cidr: "100.1.0.0/16"
```

程式執行完後，可使用以下指令匯出測試報告：

```bash
$ rally task report --out output.html
```

-----------------------------------

其他進階使用方式
===============

當然以上是最基本的使用方式，以下介紹幾種不同變化：

### 加上 SLA 條件

```yaml
{% set flavor_name = flavor_name or "m1.medium" %}
{% set image_name = "ubuntu-trusty-server-amd64-qcow2" %}
{% set instance_count = 5 %}

---
NovaServers.boot_and_delete_server:
  - args:
      flavor:
          name: "{{ flavor_name }}"
      image:
          name: {{ image_name }}
      force_delete: false
    runner:
      type: "constant"
      times: {{ instance_count }}
      concurrency: {{ instance_count }}
    context:
      users:
        tenants: 3
        users_per_tenant: 2
      network:
        start_cidr: "100.1.0.0/16"
    sla:
      max_avg_duration: 600
      max_seconds_per_iteration: 480
      failure_rate:
        max: 0
```

### 同時執行多個 task

```yaml
{% set flavor_name = flavor_name or "m1.medium" %}
{% set image_name = "ubuntu-trusty-server-amd64-qcow2" %}
{% set instance_count = 5 %}

---
NovaServers.boot_and_delete_server:
  - args:
      flavor:
          name: "{{ flavor_name }}"
      image:
          name: {{ image_name }}
      force_delete: false
    runner:
      type: "constant"
      times: {{ instance_count }}
      concurrency: {{ instance_count }}
    context:
      users:
        tenants: 3
        users_per_tenant: 2
      network:
        start_cidr: "100.1.0.0/16"

NovaServers.boot_and_bounce_server:
  - args:
      flavor:
          name: "{{ flavor_name }}"
      image:
          name: "{{ image_name }}"
      force_delete: false
      actions:
        - hard_reboot: 1
        - soft_reboot: 1
        - stop_start: 1
        - rescue_unrescue: 1
      runner:
        type: "constant"
        times: {{ instance_count }}
        concurrency: {{ instance_count }}
      context:
        users:
          tenants: 3
          users_per_tenant: 2
        network:
          start_cidr: "100.1.0.0/16"
```

### 使用 [Jinja2](http://jinja.pocoo.org/docs/dev/) 語法
