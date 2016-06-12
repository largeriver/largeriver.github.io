---
title: 安全杂记
date: 2016-06-12 16:49:24

categories:
- security

tags: 
- security
---

# 安全目的

安全边界问题：安全的核心是安全边界。
安全的核心是保护信息资产，其实现方式是信息流控制。

## 信息资产
信息资产assets的价值在于提供给组织的竞争优势（advantage）和效用（utils）:
* 组织外，scarcity
* 组织内，shareability


## 安全三要素：

* 安全策略
* 安全机制
* 安全保障

## CIA原则
任何类型的安全控制共有的三个主要的**网路安全原则**被称为 CIA原则, 保密性 完整性 和可用性:

1. 保密性 confidentiality: 指的是若未经授权访问 则信息不向用户 流程或设备公开 
2. 完整性 integrity: 指的是信息没有受到未经授权的 修改或损坏 
3. 可用性 accountability: 指的是能够被访问 


## 软件安全框架的设计原则：
* Policy as code instead of data。策略是代码而非数据。
* Policy-agnostic  [æɡ'nɑːstɪk] security  infrastructure  策略不可知的基础设置。

## 安全设计原则的分类：

* prevention
* mitigation
* detection and recovery



---------------
# 安全机制

## 访问控制

访问控制的一些基本概念包括: 

1. 身份信息: 用户提供的身份验证信息 例如用户名 
2. 验证: 身份信息通过例如密码等机制得以确认 
3. 授权: 通过特定的指标来确定某一用户是否有权执行某些操作 
3. 可追究性: 监控和记录某一用户的操作过程

## 密码学


### 密码学原则
加密机制不是秘密，唯一的秘密是**密钥**，随机选择，保持机密性。某些参数有利于此原则：

* 保存密钥比保持算法更容易
* 改变密钥比改变算法更容易
* 标准化：容易部署，公开验证。

### 密码学的安全特性：

* encryption =》confidentiality，integration
* authentication =》confidentiality
* identification =》accountability

### 密文分组模式：

* ECB：每一块独立加密
* CBC cipher block chain：前一块密文与明文XOR后，再加密。	串行加密，并行解密

### 消息认证码
由单向散列函数构成消息认证码的方法是： HMAC keyed-hash message authentication code

### 加密与软件保护的区别：

* 密码学的基本假设是密钥始终是保密的。所有的优缺点或多或少的遵循或者违背了此原则。
* 软件保护，不管采用何种保护手段，代码和信息最终都要释放到系统中才能运行。因此，软件保护的设计原则是：在期望的时间期限内，保存秘密。

## 其他密码学技术
[dm-crypt]() 基于 device mapper机制

## “空气间隙”（air-gapping）
【空气间隙网络】与公共网络中隔离,无论是其实现逻辑还是物理实体。
很多有安全意识的组织和机构都采取了各种各样的措施,以阻止可能发生的敏感数据泄漏事件。负责保存和处理敏感数据的电子计算机通常都会在一个空气间隙网络中进行操作。这些网络在物理实体上是不会与非关键网络（主要是公共网络 ）连接的。

## 软件保护



### 软件保护和密码学之间的不同之处

* 在密码学中，有一个基本假设：攻击者永远拿不到密钥。所有成功、失败的案例中，都有坚持、违反这一基本假设的因素。
* 在软件保护中，代码混淆也好，软件水印也好，防篡改也好，它们终归会被攻击者击破。只要软件必须在攻击者那里运行，他最终就能够获得你的代码。

安全是带时效的，只要抵御攻击的时间能够达到期望即可。这可以通过攻击潜力来衡量。
保护资产，例如，使得关键算法、转账等行为发生在本地或者甚至远程的安全容器之类，也是一种安全保护策略。例如，apple pay，客户端只是view，真正的control逻辑、数据存储都在安全的服务端。为了防止程序绕过指纹验证机制，指纹的检测被放在安全容器：trustzone、SE之中；交易数据经过安全容器打包后，提交给服务器。
传统TPM比起trustzone和SE，能控制的资源更少，与用户交互的手段更少。

基于硬件的保护技术则试图改变这一情况：通过给数据、代码或者可执行文件提供一个安全的环境。


有没有一个软件保护技术能使破解它的代价远大于实施它所要付出的努力？
任何一个软件保护模型都必须能以某种方式表示应用和破解有关保护措施时的开销。


# 软件安全

内存安全memory-safe:

* time safe？
* spatial safe


编程语言如果是type-safe，一定也是memory safe的。
任何支持指针算术运算和数字到指针的转换的语言，既不是内存安全，也不是类型安全的。



sound: no miss alarm，but maybe false alarm
correctness：not false alarm，but maybe miss alarm


程序静态分析：

* flow sensitive
* path sensitive
* context sensitive


基于密码的密钥推导函数：
bcrypt 计算困难
scrypt 内存困难


## 如何对抗stack smashing attack？

* ASLR
* Nx 防止从数据区执行
* 金丝雀，GCC中的实现称之为 stack smashing protection
* fortify source

---------------
# 安全保证

## Information theoretical security
目前学术界不再使用无条件安全(Unconditional security), 而是使用Information theoretical security这一术语，其安全性是高于现行的基于计算复杂性密码体系。Shannon证明若通信双方能够分享无限长的密钥，同时结合一次一密就可以达到 Information theoretical security。

而量子密码或者说量子密钥分发，就是利用量子力学的基本原理，用单个光子或者其他微观粒子的量子态做载体，来实现通信双方分享无限长的密钥，若有人去窃听，势必会改变其量子状态，这样通信双方就可以发现窃听。若没有窃听，则用该密钥就可以实现 Information theoretical security。

## 攻击潜力的评估因素：

* 耗时
* 是否是该TOE领域的专家
* 是否是该TOE内部设计实现人员
* 攻击窗口
* 是否需要借助专业的软硬件

## TCSEC评估等级

* A1：verified Design
* B3：security domains：reference monitor（security kernel）
* B2：structured protection
* B1：labeled security：MAC
* C2：controlled access
* D： minimal protection

---------------
# SELinux
## SELINUX:

* TE => BIBA model
* MLS =》 BLP model
* RABC

## Flask的主要部件：

* OM 对象管理器，策略实施
* SS 安全服务器，策略标记与决策

## SELinux内核实现的主要部件：

*　SS 策略标记与决策
*　AVC	访问向量缓冲
* the hooks	策略实施
* the selinuxfs pseudo fs
* the netlink event notification
* 网络接口表 the network interface table

当avc报告关于某个共享库的execmod错误时，这意味着该共享库没有文本重定位信息，原因是代码没有以-fPIC或者-fpic选项来编译。

## 进程的域转换规则：

domain_trans + type_transiton: init
domain_trans + setexec	: init/shell
dynamic transition + setcurrent	:zygote


## android中如何隔离不同的账户：
* DAC： uid =  userid * AID_USER + appid
* seapp_contexts:  	isOwner, affect the label of process domain and app_data
* seapp_contexts:	levelFrom, affect the multiple category of a process 

---------------
# 移动操作系统
## IOS 安全白皮书

* 分区隔离
* 沙箱
* 代码签名
* 溢出缓解机制：ASLR DEP
* data encryption
* trustzone

iOS 与 android 在安全方面的区别：

* iOS主要使用硬件加密技术来实现安全性，并保证效率，黑盒。其核心是加密。
* android主要使用DAC、MAC来实现访问控制，使用全盘加密来保护机密性，白盒。其核心是减少攻击面。



## BYOD

BYOD的三个领域：

* MAM: 移动应用管理。

	* 企业应用商店
	* 内部应用
	* 应用策略配置

* MDM: 移动设备管理。扩展主题是EMM

	* 多平台
	* 手机安全
	* 移动应用
	* 成本
	
* MCM：移动内容管理。
	* 企业内容使用的便利性与安全性
	* 个人隐私与企业内容的隔离

---------------
# 附录

1.[信息安全管理介绍](http://leanote.com/blog/post/56f212fde575d50b90000000)
2.[dm-crypt 简介](http://dragon.leanote.com/post/dm-crypt)
3.[BYOD终极指南](http://dragon.leanote.com/post/Guide-to-BYOD)

