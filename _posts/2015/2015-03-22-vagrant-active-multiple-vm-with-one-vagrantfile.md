---
layout: post
title:  "[Vagrant] 同時啟用多台 VM (Multi-Machine)"
description: "介紹 Vagrant 如何使用單一個 Vagrantfile 啟動多台 VM(Introduce how to active multiple virtual machine with one Vagrantfile in Vagrant)"
date:   2015-03-22 13:50:00
published: true
comments: true
categories: [vagrant]
tags: [Vagrant, Virtualization]
---

因為最近有測試 Ansible & OpenStack 的需求，所以想看看 vagrant 有沒有比較方便可以啟動多台 VM 的方法

發現原來 vagrant 本身就有支援同時啟動多台 VM 的功能

以下直接用個 Vagrantfile 範例說明：

``` ruby
Vagrant.configure("2") do |config|
  
  # 此處的 shell provision 會套用到所有的 VM
  config.vm.provision "shell", inline: <<-SHELL
	sudo apt-get -y update
	sudo apt-get -y dist-upgrade
	# vim 安裝 & 設定
	sudo apt-get -y install vim
	[[ $(grep tabstop /etc/vim/vimrc | wc -l) -eq 0 ]] && sudo echo "set tabstop=4" >> /etc/vim/vimrc
	[[ $(grep shiftwidth /etc/vim/vimrc | wc -l) -eq 0 ]] && sudo echo "set shiftwidth=4" >> /etc/vim/vimrc
	[[ $(grep "set nu" /etc/vim/vimrc | wc -l) -eq 0 ]] && sudo echo "set nu" >> /etc/vim/vimrc
  SHELL
  
  # shared folder 設定
  config.vm.synced_folder "../data", "/vagrant_data"

  # VM 1 設定
  config.vm.define "ansible-control-node" do |c|
    c.vm.box = "ubuntu/trusty64"
	c.vm.hostname = "ansible-control-node"
	c.vm.network "private_network", ip: "192.168.30.5"
	
	# VM 1 shell provision 設定
	c.vm.provision "shell", inline: <<-SHELL
		sudo apt-get -y install software-properties-common
		sudo apt-add-repository -y ppa:ansible/ansible
		sudo apt-get update
		sudo apt-get -y install ansible
		echo "Hello, I am ansible control node"
	  SHELL
  end

  # VM 2 設定
  config.vm.define "ansible-managed-node1" do |m1|
    m1.vm.box = "ubuntu/trusty64"
	m1.vm.hostname = "ansible-managed-node1"
	# m1 的網路設定
	m1.vm.network "private_network", ip: "192.168.30.10"
	m1.vm.network "forwarded_port", guest: 80, host: 8080
	m1.vm.network "forwarded_port", guest: 443, host: 8443
	
	# VM 2 shell provision 設定
	m1.vm.provision "shell", inline: <<-SHELL
		echo "Hello, I am ansible managed node1"
	  SHELL
  end 
  
end
```

透過以上的 Vagrantfile，就可以直接啟動兩台 VM，若要透過 vagrant ssh 連進 VM，要加上 vm 的名稱，例如：

``` bash
$ vagrant ssh ansible-control-node
```

or 

``` bash
$ vagrant ssh ansible-managed-node1
```

從設定檔可以看出，透過加入一些 shell provision 的設定，就可以方便的建立一個馬上可以用的測試環境；如果測試完了，直接 **vagrant destroy --force** 就可以把所有 VM 一起移除囉!


參考資料
========

- [Multi-Machine - Vagrant Documentation](https://docs.vagrantup.com/v2/multi-machine/index.html)