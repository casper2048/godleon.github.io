---
layout: post
title:  "[Vagrant] 同時客製化多個 VM"
description: "介紹 Vagrant 如何客製化 VM，包含 VM 數量 / cpu 核心數 / memory 大小 / 硬碟數量&大小 / 網卡數量 ... 等等"
date:   2015-05-22 19:55:00
published: true
comments: true
categories: [vagrant]
tags: [Vagrant, Virtualization]
---

從開始接觸 [vagrant](https://www.vagrantup.com/) 到現在，都只是用來快速開啟簡單的 VM 使用

用久了總會有比較多的需求，像是....

1. 一次產生多個 VM

2. 可以設定不同的 cpu 數量 & memory 大小

3. 硬碟數量可以超過 1 個，並個別指定其大小

4. 可以設定多張網路卡

於是今天花了點時間研究，拼拼湊湊之後終於完成以上幾個需求了!

以下是 vagrantfile 的內容：

``` ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

domain = "example.com"

# VM 設定 (使用 json 格式定義)
VMs = [
    {
        :name => "node01", 
        :cpu => "1", 
        :mem => "1024", 
        :private_ip1 => "192.168.121.101", 
        :private_ip2 => "192.168.122.101",
        :private_ip3 => "192.168.123.101",         
        :hostname => "node01.#{domain}", 
        :disk2_name => "vm_disks/node01_disk2",
        :disk2 => "vm_disks/node01_disk2.vdi",
        :disk3_name => "vm_disks/node01_disk3",
        :disk3 => "vm_disks/node01_disk3.vdi"
    }, 
    {
        :name => "node02", 
        :cpu => "2", 
        :mem => "2024", 
        :private_ip1 => "192.168.121.102", 
        :private_ip2 => "192.168.122.102", 
        :hostname => "node02.#{domain}", 
        :disk2_name => "vm_disks/node02_disk2", 
        :disk2 => "vm_disks/node02_disk2.vdi",
        :disk3_name => "vm_disks/node02_disk3",
        :disk3 => "vm_disks/node02_disk3.vdi"
    }
]

Vagrant.configure(2) do |config|

  VMs.each do |opts|
    config.vm.define opts[:name] do |node|
      
      node.vm.box = "ubuntu/trusty64"
      node.vm.hostname = opts[:hostname]
      
      # 設定 VM 網卡(private_network = host-only)
      node.vm.network "private_network", ip: opts[:private_ip1]     if opts[:private_ip1]
      node.vm.network "private_network", ip: opts[:private_ip2]     if opts[:private_ip2]
      node.vm.network "private_network", ip: opts[:private_ip3]     if opts[:private_ip3]

      # 設定 VM 開機時執行的 shell command
      node.vm.provision "shell", inline: "echo hello from #{opts[:name]}"
      
      node.vm.provider :virtualbox do |vb|
        # 設定 VM cpu & memory
        vb.customize ["modifyvm", :id, "--memory", opts[:mem]]
        vb.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
        
        # 判斷 VHD 是否存在，不存在則建立
        unless File.exist?(opts[:disk2])
          vb.customize ["createhd",  "--filename", opts[:disk2_name], "--size", 2048]
        end
        # 將 VHD 與 VM disk controller 綁定
        vb.customize ["storageattach", :id, "--storagectl", "SATAController", "--port", "1", "--type", "hdd", "--medium", opts[:disk2]]
        
        unless File.exist?(opts[:disk3])
          vb.customize ["createhd",  "--filename", opts[:disk3_name], "--size", 2048]
        end
        vb.customize ["storageattach", :id, "--storagectl", "SATAController", "--port", "2", "--type", "hdd", "--medium", opts[:disk3]]
      end
    end
  end
  
end
```


參考資料
========

- [Tips & Tricks - Vagrantfile - Vagrant Documentation](http://docs.vagrantup.com/v2/vagrantfile/tips.html)

- [Multi-machine Vagrantfile with Shorter, Cleaner Syntax Using JSON and Loops](http://thornelabs.net/2014/11/13/multi-machine-vagrantfile-with-shorter-cleaner-syntax-using-json-and-loops.html)

- [Add a second disk to system using vagrant](https://gist.github.com/leifg/4713995)

- [ruby - Create two disks for multiple environment machines in Vagrant - Stack Overflow](http://stackoverflow.com/questions/27877929/create-two-disks-for-multiple-environment-machines-in-vagrant)

- [My (improved) ‘Vagrantfile’ | vStone Blog](http://vstone.eu/my-improved-vagrantfile/)

- [networking - Vagrant: how to configure multiple NICs within a Vagrantfile? - Stack Overflow](http://stackoverflow.com/questions/23957752/vagrant-how-to-configure-multiple-nics-within-a-vagrantfile)