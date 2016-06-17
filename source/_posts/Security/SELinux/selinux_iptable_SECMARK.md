---
title: "SELinux: 利用iptable 的 SECMARK 机制来为Android网络通信进行MAC访问控制"
date: 2015-05-18 16:19:53
categories:
- security

tags:
- SELinux
- iptable
---



为对AF_INET socket实施访问控制，你需要使用某种方法对数据包添加标签，并且通过网络栈在数据包中传递这些标签。这种做法并不常用，当SELinux出现时，主流Linux甚至不支持该方式。不过现在，SELinux中已经整合了两种数据包标记方式.
<!-- more -->
# SECMARK
SECMARK 基于iptable 配置中的报文特征来设置数据包标签，然后执行send/recv策略检查。策略的组成是：sender 域、receiver域 以及数据包类型。为启用SECMARK机制，你需要在kernel中开启相应的支持，并且配置iptable SECMARK或者CONNSECMARK规则。这些报文标签仅存在于本机的网络协议栈，并不会传播到其他主机。

SEC处理过程：
![SECMARK处理过程](/img/selinux_iptable_SECMARK_01.png)


# Labeled XFRM or NetLabel
基于发送者来标记保温的两种机制。其报文标签可以传播到远程系统，然后在接收时执行安全检查。其安全策略包含sender 域、receiver域 以及数据包类型。
## Labeled XFRM
该机制将报文标签保存在一个IPSEC安全关联中，由接收端从SA中推断报文标签。
## NetLabel
该机制将安全标签保存在通过某个IP选项（例如：CIPSO）保存在每个报文中。如果需要在Android上支持NetLabel机制，开发者需要开启相关内核选项，并一直netlabelctl

# Android支持
在Android上启用SECMARK机制非常容易，因为我们已经将内核配置选项增加到了android-base.cfg中，iptables SECMARK支持已经被包含其中。

## 内核支持
你需要检查并启用下列Android 内核配置项：

+ CONFIG_NETWORK_SECMARK=y
+ CONFIG_IP_NF_SECURITY=y
+ CONFIG_NETFILTER_XT_TARGET_CONNSECMARK=y
+ CONFIG_NETFILTER_XT_TARGET_SECMARK=y
+ CONFIG_NF_CONNTRACK_SECMARK=y

目前，这些配置型属于android-base.cfg的一部分。不过我在MSM8939 Android5.0.2中的源码树中没有找到相关配置。

## 设置规则
iptable SECMARK规则可以用来对报文进行标记，例如：

	iptables -t security -A INPUT -p tcp --dport 4591 -j SECMARK --selctx u:object_r:http_packet_t:s0"
	iptables -t mangle -A OUTPUT -p tcp --sport 4591 -j SECMARK --selctx  u:object_r:http_packet_t:s0

你还需要为你希望收发任何报文增加规则，否则收发将被拒绝：

	#容许收发未标记的报文
	allow domain unlabeled:packet { send recv};
	#容许platform_app  域的进程收发标记为http_packet_t的报文
	allow platform_app http_packet_t:packet { recv send };



# 参考链接

1. [Add selinux network script to policy](https://bitbucket.org/seandroid/external-sepolicy/commits/70d4fc2243721a54cd177959e05cf81b54c4e226)
2. [SELinux Networking Support](http://selinuxproject.org/page/NB_Networking)
