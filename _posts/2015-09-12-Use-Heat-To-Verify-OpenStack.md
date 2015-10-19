---
layout: post
title:  "[OpenStack] 使用 Heat 快速驗證 OpenStack 環境"
description: "此篇文章中說明如何使用 Heat，可以快速的驗證 OpenStack 環境以及相關服務是否可以正常運作"
date: 2015-09-12 10:35
published: true
comments: true
categories: [openstack]
tags: [OpenStack, Heat]
---

在這邊文章中，要透過 Heat 快速的建置出與 [[OpenStack] 簡單驗證 OpenStack 環境的 script](http://godleon.github.io/blog/2015/09/01/Simple-Script-For-Verify-OpenStack/) 這篇文章中相同的配置。

前置工作
=======

在這邊我們只有一個前置工作必須完成，就是透過 Neutron client 建立一個對外的 network，語法如下：

```bash
$ neutron net-create ext_net --router:external --provider:physical_network external --provider:network_type flat
```

當對外網路順利建立完成後，前置工作則告一段落。

------------------------------------------------

設定 Heat Template
==================

以下是用來配置虛擬環境用的 template：(檔名存為 **<font color='red'>heat_sample.yaml</font>**)

```yaml
heat_template_version: 2014-10-16

description: Heat example

parameters:
  ExtNetID:
    type: string
    description: ID of the external network

resources:
# ========== Security group ==========
  sg_default:
    type: "OS::Neutron::SecurityGroup"
    properties:
      name: sg_default
      description: "Ping and SSH"
      rules:
        - protocol: icmp
        - protocol: tcp
          port_range_min: 22
          port_range_max: 22

# ========== ssh keypair for authentication ==========
  ssh_keypair:
    type: "OS::Nova::KeyPair"
    properties:
      name: "demo-key"
      public_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCq37Y8IgAthTnRBVj1JSRgPjIKgrD8fvZC2/IdP/6xkJyxk/FnNUgZ2wUokqroIagx6QXaFWD+a6oGZYISlBbgrsF24JqovMsRiYxoiHcy//qXuZJC5PlC/F8An46yUsFjcQzaJPktaaJ0fxdh0p2VSFmgFJ/B+nLgC0Q852WeR2Tm83I00HNTaqDEpdlbCfrOZoLXfO2wzm8McAjXJaKoT0/fb/4JwzgpBMjmjcZdZrqzLYJI79bdQDI4nKr3K4p+mCDS86XU7c0E0EMBcstNpntNvMT1sVupmiTwMEs22ycJrheOAA6ZA3uK6PTHq3dYyp8tbl6pykTpY0bCwwPB user@ubuntu-trusty-64"

# ========== cloud images list ==========
  cirros-0.3.4-x86_64-qcow2:
    type: "OS::Glance::Image"
    properties:
      name: "cirros-0.3.4-x86_64-qcow2"
      container_format: "bare"
      disk_format: "qcow2"
      location: "http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img"
      is_public: "true"
  ubuntu-trusty-server-amd64-qcow2:
    type: "OS::Glance::Image"
    properties:
      name: "ubuntu-trusty-server-amd64-qcow2"
      container_format: "bare"
      disk_format: "qcow2"
      location: "https://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img"
      is_public: "true"
  CentOS-7-x86_64-GenericCloud-qcow2:
    type: "OS::Glance::Image"
    properties:
      name: "CentOS-7-x86_64-GenericCloud-qcow2"
      container_format: "bare"
      disk_format: "qcow2"
      location: "http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2c"
      is_public: "true"

# ========== Subnet of external network ==========
  ext-subnet:
    type: "OS::Neutron::Subnet"
    properties:
      name: "ext-subnet"
      cidr: "10.10.198.0/24"
      allocation_pools:
        - start: 10.10.198.101
          end: 10.10.198.200
      dns_nameservers: ["8.8.8.8", "8.8.4.4"]
      enable_dhcp: "false"
      gateway_ip: "10.10.198.254"
      network: { get_param: ExtNetID }

# ========== Subnet for test ==========
  test-net:
    type: "OS::Neutron::Net"
    properties:
      name: "test-net"
  test-subnet:
    type: "OS::Neutron::Subnet"
    properties:
      name: "test-subnet"
      cidr : "192.168.50.0/24"
      dns_nameservers: ["8.8.8.8", "8.8.4.4"]
      enable_dhcp: "true"
      gateway_ip: "192.168.50.1"
      network_id: { get_resource: test-net }
  test-router:
    type: "OS::Neutron::Router"
    properties:
      admin_state_up: "true"
      name: "test-router"
      external_gateway_info:
        network: { get_param: ExtNetID }
  test-router_if0:
    type: "OS::Neutron::RouterInterface"
    properties:
      router: { get_resource: test-router }
      subnet: { get_resource: test-subnet }
  test-net_port-1:
    type: "OS::Neutron::Port"
    properties:
      name: "test-net_port-1"
      admin_state_up: "true"
      network_id: { get_resource: test-net }
      security_groups:
        - { get_resource: sg_default }
  test-net_floating-ip:
    type: "OS::Neutron::FloatingIP"
    properties:
      floating_network: { get_param: ExtNetID }
      port_id: { get_resource: test-net_port-1 }

# ========== Subnet for staging ==========
  staging-net:
    type: "OS::Neutron::Net"
    properties:
      name: "staging-net"
  staging-subnet:
    type: "OS::Neutron::Subnet"
    properties:
      name: "staging-subnet"
      cidr : "192.168.50.0/24"
      dns_nameservers: ["8.8.8.8", "8.8.4.4"]
      enable_dhcp: "true"
      gateway_ip: "192.168.50.1"
      network_id: { get_resource: staging-net }
  staging-router:
    type: "OS::Neutron::Router"
    properties:
      admin_state_up: "true"
      name: "staging-router"
      external_gateway_info:
        network: { get_param: ExtNetID }
  staging-router_if0:
    type: "OS::Neutron::RouterInterface"
    properties:
      router: { get_resource: staging-router }
      subnet: { get_resource: staging-subnet }
  staging-net_port-1:
    type: "OS::Neutron::Port"
    properties:
      name: "staging-net_port-1"
      admin_state_up: "true"
      network_id: { get_resource: staging-net }
      security_groups:
        - { get_resource: sg_default }
  staging-net_floating-ip:
    type: "OS::Neutron::FloatingIP"
    properties:
      floating_network: { get_param: ExtNetID }
      port_id: { get_resource: staging-net_port-1 }

# ========== Subnet for production ==========
  production-net:
    type: "OS::Neutron::Net"
    properties:
      name: "production-net"
  production-subnet:
    type: "OS::Neutron::Subnet"
    properties:
      name: "production-subnet"
      cidr : "192.168.50.0/24"
      dns_nameservers: ["8.8.8.8", "8.8.4.4"]
      enable_dhcp: "true"
      gateway_ip: "192.168.50.1"
      network_id: { get_resource: production-net }
  production-router:
    type: "OS::Neutron::Router"
    properties:
      admin_state_up: "true"
      name: "production-router"
      external_gateway_info:
        network: { get_param: ExtNetID }
  production-router_if0:
    type: "OS::Neutron::RouterInterface"
    properties:
      router: { get_resource: production-router }
      subnet: { get_resource: production-subnet }
  production-net_port-1:
    type: "OS::Neutron::Port"
    properties:
      name: "production-net_port-1"
      admin_state_up: "true"
      network_id: { get_resource: production-net }
      security_groups:
        - { get_resource: sg_default }
  production-net_floating-ip:
    type: "OS::Neutron::FloatingIP"
    properties:
      floating_network: { get_param: ExtNetID }
      port_id: { get_resource: production-net_port-1 }

# ========== initial instances ==========
  cirros-instance:
    type: "OS::Nova::Server"
    properties:
      name: "cirros-instance"
      image: { get_resource: "cirros-0.3.4-x86_64-qcow2" }
      flavor: "m1.tiny"
      key_name: { get_resource: "ssh_keypair" }
      networks:
        - port: { get_resource: "test-net_port-1" }
      user_data_format: RAW
  ubuntu-instance:
    type: "OS::Nova::Server"
    properties:
      name: "ubuntu-instance"
      image: { get_resource: "ubuntu-trusty-server-amd64-qcow2" }
      flavor: "m1.medium"
      key_name: { get_resource: "ssh_keypair" }
      networks:
        - port: { get_resource: "staging-net_port-1" }
      user_data_format: RAW
  centos-instance:
    type: "OS::Nova::Server"
    properties:
      name: "centos-instance"
      image: { get_resource: "CentOS-7-x86_64-GenericCloud-qcow2" }
      flavor: "m1.medium"
      key_name: { get_resource: "ssh_keypair" }
      networks:
        - port: { get_resource: "production-net_port-1" }
      user_data_format: RAW
```
------------------------------------------------

Create & Delete Heat stack
=================

接著我們只要使用以下指令，就可以透過 Heat 將定義好的 virtual infrastructure(稱為 **<font color='red'>stack</font>**) 建置到 OpenStack 環境上：

```bash
$ heat stack-create -f heat_sample.yaml -P "ExtNetID=$(neutron net-list | awk '/ ext_net / { print $2 }')" quick_stack
```

當然，移除 stack 也很容易，使用以下指令即可完成：

```bash
$ heat stack-delete quick_stack
```

------------------------------------------------

參考資料
=======

- [heat - access created vm - permission denied (publickey) - Ask OpenStack: Q&A Site for OpenStack Users and Developers](https://ask.openstack.org/en/question/58473/heat-access-created-vm-permission-denied-publickey/)

- [OpenStack Resource Types — heat 5.0.0.0b4.dev135 documentation](http://docs.openstack.org/developer/heat/template_guide/openstack.html)

- [OpenStack Orchestration In Depth, Part II: Single Instance Deployments](https://developer.rackspace.com/blog/openstack-orchestration-in-depth-part-2-single-instance-deployments/)

- [OpenStack-Heat-Installation/Create-your-first-stack-with-Heat.rst at master · MarouenMechtri/OpenStack-Heat-Installation](https://github.com/MarouenMechtri/OpenStack-Heat-Installation/blob/master/Create-your-first-stack-with-Heat.rst)

- [Ubuntu, cloud-init, and OpenStack Heat · Scott's Weblog · The weblog of an IT pro specializing in virtualization, networking, open source, and cloud computing](http://blog.scottlowe.org/2015/04/23/ubuntu-openstack-heat-cloud-init/)
