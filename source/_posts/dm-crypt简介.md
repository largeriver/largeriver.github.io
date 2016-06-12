---
title: dm-crypt简介
date: 2016-06-12 17:09:52
categories:
- security

tags: 
- security
- crypt
---

# 简介
dm-crypt是一个透明磁盘加密子系统，建立在2.6版本内核的device-mapper特性之上的,使用了内核密码应用编程接口实现了透明的加密。
与早期的cryptoloop不同，dm-crypt被设计为支持更高级的操作，如XTS、LRW和ESSIV，以便避免水印攻击。 
dm-crypt 作为一个设备映射器目标(device-mapper target)来实现,可以被堆叠在其他设备映射器的顶端。因此，它可以用来加密整个磁盘、分区、raid卷、逻辑卷以及文件。

# 设备映射器 device-mapper
设备映射器是设计用来为在实际的块设备之上添加虚拟层提供一种通用灵活的方法, 用于将一个块设备映射到另一个块设备。除了提供加密功能，设备映射器还为LVM、软RAID提供支持，并为系统添加了一些诸如文件系统快照之类的附加特性。
设备映射器相当于一个过滤装置，它从它自己提供的虚拟块设备中获得数据并对它们进行处理，然后才把它们传递给另一个块设备。
当设备映射器被用于数据加密时，它会在/dev/mapper/目录下创建一个新的块设备。对于用户来说，这个虚拟设备和系统上的其它任何块设备在使用时没有区别。设备映射器（更确切地说是设备映射器的dm-crypt模块）使用对称加密算法，如AES，对输入该虚拟设备的所有数据进行加密。加密后的数据被传输到另一个块设备加以保存。


# 参考文献
* https://en.wikipedia.org/wiki/Dm-crypt
* [EncryptedFilesystemHowto](http://wiki.ubuntu.org.cn/index.php?title=EncryptedFilesystemHowto&variant=zh-cn#.E5.8A.A0.E5.AF.86.E6.96.87.E4.BB.B6.E7.B3.BB.E7.BB.9F)

