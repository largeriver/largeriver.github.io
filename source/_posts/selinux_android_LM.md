---
title: 【译】AndroidL和AndroidM的SEAndroid状态与展望
date: 2016-05-24 11:21:49
categories:
- android
- security

tags:
- SELinux
- security
---


原文是在Linux Sercurity Submit 2015 会议中的PPT[SELinux in Android Lollipop and Marshmallow](http://kernsec.org/files/lss2015/lss2015_selinuxinandroidlollipopandm_smalley.pdf)。

从Android4.4开始，SELinux被应用于保护Android的可信计算基(TCB).

# AndroidL
## 在AndroidL中，SELinux的状态是：

* 所有的系统和应用都是受约束的，包括root守护进程
* 只有kernel和init进程是“未约束”的。
* 即使这两个域也不能为所欲为。

## SELinux是如何保护TCB？

* 没有进程可以map低端内存，或者访问/deb/{k}/mem
* 只有init可以设置内核配置和策略
* 只有recovery可以修改/和/system目录，这两个目录分别代表rootfs和system分区
* 本地服务只能从/和/system上开始执行。
* 只有debuggerd可以执行ptrace调用
* 应用不能写大多数netlink 套接字
* 应用不能写大多数service 套接字
* 不容许读取和follow不可信的符号链接。


## SElinux如何保护系统服务？

以 负责提供密钥的安全存储的Android KeyStore 为例：

* SELinux 内核增强防护：
    * 没有进程可以ptract keystore
    * 没有其他进程可以打开/data/misc/keystore 文件
* SELinux 用户空间访问控制：
    *  敏感操作由selinux策略进行限制
    *  keystore 在客户请求时，检查相应的selinux 策略

## AndroidL CTS of SELinux
对于一个正常androidL及更新的版本，其必须要满足的基本原则是：

* 所有的域都处于enforcing状态
* 只有init运行在init域
* 只有kernel 线程运行在kernel域
* android核心服务运行在各自独立的域中
* 正常版本中，不应该存在进程运行在su域和recovery域
* 不支持policy booleans

# Android 6.0 Marshmallow (“M”)

## SELinux 新变化

* ioctl 白名单支持
* 增强的android多用户支持
* 增强的chrome沙箱
* Binder 锁定
* 策略加固
* CTS 强化

## Android 多用户

Android多用户机制，在Android4.2中引入平板中，从AndroidL中开始支持Phone。Android多用户机制是受控受限配置的基础，如Android for Work。

## SELinux和Android 多用户

* 目标：加强android多用户的透明隔离 而无需复杂的策略
* 将用户id映射为惟一的MLS级别（mls categories），指派给应用进程和文件
* MLS约束将防止不同级别间的通讯（binder中间层除外）
* 不同的用户的进程自动被赋予不同的级别
* SElinux防止跨级别的通信：发送信号，访问/proc/pid,打开应用数据文件，本地socet
* 无需额外的针对每个用户或者每个应用的策略配置。mls机制搞定一切

## Chrome沙箱

## BInder锁定(Locking down)

## 策略加固

* 强制init 在exec时进行域转换，所有的服务和辅助程序都运行在独立的域之中
* 锁定块设备的访问：防止对关键分区的直接访问；限制每个域只能访问必需的分区
* 移除非受限域：init和kernel 也不是非受限域了。
* 更多额度neverallow规则

## CTS 增强


# 将来

* Enable apps to opt into stronger protections：Sandboxing, isolation, file protection
* 调查新的运行时权限策略(AndroidM)
* 改进SELinux工具
* 未来的用户空间策略增强
* 内核自保护

References:

* Source code: https://bitbucket.org/seandroid
* Project page: http://seandroid.bitbucket.org
* ToDo list: https://bitbucket.org/seandroid/wiki/wiki/ToDo
