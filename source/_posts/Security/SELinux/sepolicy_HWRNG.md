---
title: "为什么不容许应用访问HWRNG(硬件随机数产生器)设备?"
date: 20160529 15:22:33
categories:
- security

tags:
- SELinux
- sepolicy
---

为什么不容许应用访问HWRNG(硬件随机数产生器)设备?简短的说：使用/dev/random比直接使用HWRNG更安全。
<!--more-->
# 问题
HWRNG设备是**[硬件随机数产生器（hardware random number generator）](https://en.wikipedia.org/wiki/Hardware_random_number_generator)**的缩写，在我的android手机上，其设备名为 /dev/hw_random.
>crw------- root     root              u:object_r:hw_random_device:s0 hw_random

在domain.te中，仅仅容许ueventd和system_server来访问HWRNG。
>\# Only init, ueventd and system_server should be able to access HW RNG
>neverallow { domain -init -system_server -ueventd -unconfineddomain } hw_random_device:chr_file *;

为什么不容许其他进程直接访问硬件随机数设备呢？

# 原因
简短的说：使用/dev/random比直接使用HWRNG更安全。
在android的early-boot的阶段，init进程从/dev/hw_random中将随机字节拷贝到/dev/random中。这时候，/dev/random包含了HWRNG中所有熵信息加上自己的熵信息，这样远比仅仅使用HWRNG更安全。
**该规则保证所有的进程使用我们手头可用的最随机的数据**。归根结底，对HWRNG本身，目前并没有强安全保证；其实现细节也各不相同，或好或坏；也没有公开的源代码