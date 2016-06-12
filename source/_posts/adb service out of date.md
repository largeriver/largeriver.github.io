---
title: 如何解决adb过时问题：adb service out of date
date: 2015-06-30 10:13:49
categories:
- android

tags:
- adb
- howto
---

参考文献： http://my.oschina.net/u/924450/blog/179519

# 问题描述

在我的工作环境下，很容易出现adb service out of date问题，影响远程调试。通常，下列情况会出现该问题：

* 重新插拔USB线
* adb remount
* adb root

# 解决过程

1. 确保adbd 进程被杀掉，然后重新插上USB线。看看能不能启动adbd

		>adb nodaemon server
		cannot bind 'tcp:5037'

2. 检查5037端口的占用情况，原来是进程4172

		>netstat -ano | findstr "5037"
		协议  本地地址                    外部地址             状态            PID
		TCP    127.0.0.1:5037         0.0.0.0:0              LISTENING       4172
		TCP    127.0.0.1:5037         127.0.0.1:54592        TIME_WAIT       0
		TCP    127.0.0.1:5037         127.0.0.1:54600        TIME_WAIT       0
		TCP    127.0.0.1:5037         127.0.0.1:54603        TIME_WAIT       0
		TCP    127.0.0.1:54592        127.0.0.1:5037         TIME_WAIT       0
		TCP    127.0.0.1:54598        127.0.0.1:5037         TIME_WAIT       0
		TCP    127.0.0.1:54599        127.0.0.1:5037         TIME_WAIT       0
		TCP    127.0.0.1:54602        127.0.0.1:5037         TIME_WAIT       0

3. **找到4172该进程，发现是搜狗拼音输入法的SogouPhoneService.exe。作为一个拼音输入法，你没事去监听5037端口干什么？处理办法：**
4. 杀无赦，
5. 然后删除对应的exe文件。
6. 然后在exe所在的文件夹，新建一个名字为SogouPhoneService.exe的空目录，以防搜狗自动恢复该exe文件。

4. 再次查看5037端口，这次是adb进程了

		>adb start-server
		>
		>netstat -ano | findstr 5037
		TCP    127.0.0.1:5037         0.0.0.0:0              LISTENING       7248

5.OVER

# 总结
adb service out of date的原因就是其他程序抢占了该端口。

除了adb service外，任何跑去监听5037端口的行为都是不能看做是善意的。
