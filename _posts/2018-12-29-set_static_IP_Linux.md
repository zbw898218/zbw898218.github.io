---
layout: post
title: 虚拟机Linux的静态IP设置
date: 2018-12-29
categories: blog
tags: [Linux]
description: Linux的静态IP设置
---

在日常编程学习中，必须要使用linux系统，所以配置一个静态的IP会很有必要。所以整理了一下，在设置静态IP过程中要注意的一些事项。

<!--more-->


##1.修改配置文件

1.路径：/etc/sysconfig/network-scripts/ifcfg-enoXXX
2.信息修改

	#网络类型
	TYPE=Ethernet
	#设置ip模式，static：静态 
	BOOTPROTO=static
	DEFROUTE=yes
	PEERDNS=yes
	PEERROUTES=yes
	IPV4_FAILURE_FATAL=no
	IPV6INIT=yes
	IPV6_AUTOCONF=yes
	IPV6_DEFROUTE=yes
	IPV6_PEERDNS=yes
	IPV6_PEERROUTES=yes
	IPV6_FAILURE_FATAL=no
	#名称
	NAME=eno16777736
	UUID=xxxxxx-xxxx-xxxx-xxx-xxxx
	#设备名称	
	DEVICE=eno16777736
	#设置开机自启动
	ONBOOT=yes
	#设置静态ip地址
	IPADDR=192.168.204.10
	#子网掩码
	NETMASK=255.255.255.0
	#网管	
	GATEWAY=192.168.204.2
	#DNS ,还可以设置几个：8.8.8.8 等等	
	DNS1=192.168.204.2

编辑完成，：wq 保存退出！
##重启网络服务并测试

	重启服务
	#第一种
	service network restart
	#第二种
	/etc/init.d/network restart

	ping IP 测试

	ping 192.168.204.1

	