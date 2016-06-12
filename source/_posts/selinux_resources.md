---
title: "SELinux 资源索引"
date: 2016-05-26 11:21:49
categories:
- security

tags:
- SELinux
---

# 网站

* [Android 官方文档](https://source.android.com/devices/tech/security/se-linux.html)
* [bitbucket](http://seandroid.bitbucket.org/index.html) 提供了SEAndroid未来的发展方向，例如EOps、MMAC、策略注入、可加载更新、中间策略语言


# Books

* **The SELinux Notebook>> 4th** :  NSA官方参考手册，包含了SEAndroid的内容
* **SELinux by Example: Using Security Enhanced Linux** : 比较详细的SELinux例子，有中文译本
* **[Implement SELinux as a LSM](https://www.nsa.gov/research/_files/publications/implementing_selinux.pdf)** ： Stephen Smalley，SELinux的内核实现细节

# 讨论组

seandroid-list@tycho.nsa.gov

# 工具箱

* [seal](https://github.com/seandroid-analytics/seal)
SEAndroid现场分析工具，目前非常原始。
* [setools4](https://github.com/TresysTechnology/setools)
图形化的sepolicy开源分析工具，非常强大，比较有特定的功能是：域转换分析、信息流分析、重复规则分析。二次开发也非常容易。
