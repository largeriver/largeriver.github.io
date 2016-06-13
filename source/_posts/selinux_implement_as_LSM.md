---
title: "SELinux 基于LSM的实现"
date: 20150806 15:22:33
categories:
- security

tags:
- SELinux
- LSM
---
[TOC]
# 介绍
## 注意

原文档为2009年最后修订，基于kernel2.6,这与目前(201508)最新的kernel3.6中的接口差异较多，但是主要的思想应该是一致的。
<!-- more -->
## 名词解释

* LSM(linux security modules): Linux 安全模块
* hook function：钩子函数
* security field：安全字段
* System V IPC: 系统5 进程间通讯机制
* access control：访问控制
* help function：辅助函数

## 主要文件

* **include/linux/security.h**： LSM所有的钩子函数; 通用接口；详细的文档。
* **security/security.c**: 通用接口的实现，主要是将钩子函数的回调封装为函数。
`int security_cred_alloc_blank(struct cred *cred, gfp_t gfp){	return security_ops->cred_alloc_blank(cred, gfp);}`
* **security/selinux/hooks.c** LSM钩子函数的SELinux代码实现
* **security/capability.c** capabilities 模块,也是linux的默认模块


# 3 概览
LSM 框架提供了一种机制，让各种安全检查可以通过钩子函数挂在新的内核扩展上。LSM可以配置所需要的安全模块的名称：

* 内核编译选项 CONFIG_DEFAULT_SECURITY
* 内核命令行参数 "security=..."


LSM框架在内核数据结构中增加了security字段，在内核代码的关键点插入了对hook函数的调用，以便管理安全字段和执行访问控制。它也增加了用于注册和反注册安全模块的函数。用于新安全名字空间的扩展属性句柄（extended attribute handlers）被加入到文件系统中，以支持新的文件安全属性；在用于空间，引入了/proc/pid/attr子目录，以提供对新的进程安全属性的用户空间访问。

LSM security field  仅仅是简单的void*指针。对于进程和程序执行安全信息，security fields 被加入到struct task_struct和struct linux_binprm。对于文件系统安全信息，一个security filed被加入了struct super_block。对于pipe，fifo 和socket 安全信息，security fields被计入了struct sock。为了System V IPC的安全信息，security fields被加入了struct kern_ipc_perm和 struct msg_msg。

每一个LSM 都是 在全局表security_ops中的一个函数指针，该全局表定义在**include/linux/security.h**中（该文件也包含了钩子函数详尽的文档）。钩子函数基于内核目标被逻辑分组为task、file、sock等等，以及一些用于系统操作的琐碎的钩子。

全局表security_ops被初始化为一个dummy 安全模块所提供的钩子函数集，该模块支持传统的superuser 逻辑。register_security 函数(security/security.c)用于容许一个安全模块将security_ops执行自己的钩子函数集，而unregister_security 则相反，用于将security_ops指向dummy安全模块。该机制用于设置主安全模块，它负责为每个钩子做出最终决策。

LSM也提供了一个简单的机制，用于在主安全模块上堆叠额外的安全模块。LSM在security_operations 结构中定义了register_security和unregister_security钩子，并提供了mod_reg_security和mod_unreg_security函数来在执行了某些合理性检查后调用这些钩子。这些辅助安全模块堆叠后如何被调用，由主模块来决定。

尽管LSM 钩子按照内核对象来组织，所有的钩子函数都可以分为两类：

* 用于管理security field的钩子。例如用于含有security field的每个内核数据结构的alloc_security和free_security钩子
* 用于执行访问控制的钩子。例如 inode_permission钩子，在访问inode时执行检查。

# 4 基本概念



# 5 新版SELinux(基于LSM) 内核实现与 旧版SELinux内核实现的对比
# 5.1 通用改变

* 安全中间层的增加
* 动态分配安全字段
* 与capabilities的堆叠
* SELinux API的重新设计
* 利用现存的linux permission函数

## 5.2 程序执行相关的改变
### 5.2.1 Filr execute_no_trans 权限

在初始SELinux内核patch中，file execute权限控制了程序发起执行的能力；process execute权限则控制从一个可执行image执行代码的能力。 这种区分是必要的，因为任何的SID可能在执行时改变。然而，当进程的SID改变时，process execute 权限与process entrypoint 权限是相同的，因此冗余了，因此其仅在SID不改变时起作用。

因此，process execute权限被file execute_no_trans权限所替代了，在任务的SID不变时检查；process  entrypoint 也被移动到了file class之中，为了保持一致性。

总结：

* SID不变时，检查file:execute, file:excute_no_trans
* SID改变时，检查file:execute, process:transition 和 file:entrypoint

### 5.2.2 状态的继承
当执行execv函数导致上下文切换时，继承的状态将有一些 改变。这些改变包括：文件描述符继承控制的改变，进程跟踪和状态共享的控制改变，新控制的增加。

在程序执行时的文件描述符继承权限检查也基于LSM的SELinux中被修订，我们在5.3.4节中讨论。

在旧版的SELinux内核补丁中，对程序跟踪和共享进程状态的检查 被插入了内核函数 compute_creds。然而该函数不能返回错误，因此当检查失败时，SELinux只能不改变SID，如同Linux在测试失败时不修改uid一样。

新版SELinux中，实现了新的权限检查来控制 信号相关的状态、资源限制的继承性，这些检查描述在12.1.4节。此外，AT_SECURE标识增加到了ELF 辅助表中，这样在执行execv并发生上下文转换时，SELinux就能够通知glibc 容许该进程自己的secure mode，以便清理环境变量以及其他一些继承状态。这个行为也受到权限检查的控制，请参考12.1.6。

## 5.3 文件系统的改变

新版对比旧版的改变：

* 使用扩展属性，而不是通过映射来保存文件的安全上下文
* 重新实现了对伪文件系统的安全标记的支持
* 利用现存permission函数的hook
* 消除了class pipe

## 5.4  socket ipc和网络的改变

新版SELinux和旧版SELinux在socket IPC和网络方面的修改包括：

* 重新设计了网络访问控制机制
* 在socket对应的inode节点的security字段存储安全信息
* 使用最低限度的侵入hook 来重新实现了SELinux 访问控制
* 改变了fd传输控制
* 省略了某些底层ioctl
* 实现了扩展的socket调用。


## 5.5 System V IPC

由于在基于LSM重新实现SELinux之前，System V IPC 安全增强还没来得及从2.2移植到2.4，所以新版的SELinux模块不得不使用SELinux 安全增强的实现来适配到2.4。
此外，新版SELinux与旧版SELinux在System V IPC上的修改还包括：

* 一种更容易的存储IPC安全数据的方式
* 利用现存的ipcperms函数中的hook

## 5.6 零碎改变

* 重新实现了sysctl 的控制方式
* 对在旧版SELinux实现中某些未特别处理的系统操作增加了新的控制，例如syslog，该操作之前仅由粗粒度的capabel来控制
* 在2.6 版本的SELinux中引入了对netlink 合适粒度的控制。请参考19章

# 6 内部架构
本章描述了SELinux安全模型的内部架构，相关代码在内核树的security/selinux 目录中。安全模型包含六个主要组件：

* the Sercurity Server：安全服务
* the access vector cache（AVC）
* the network interface table：网络接口表
* the netlink event notification code（用以通知策略的变更）：netlink事件通知
* the selinuxfs pseudo filesystem：selinux 伪文件系统
* the hook function implementations：钩子函数的实现

SS提供的通用接口包括：获取策略决策，容许安全模型的其他部分独立于所使用的特定安全策略。这些接口定义在SELinux目录中的include/security.h。SS的特定实现可以被修改或者完全替换，而不要求修改模型的其他部分。例如：SELinux 复合实现了RBAC、TE和MLS。RBAC和TE是高度可配置的，能够满足多种不同的安全目标。可以在ss子目录中找到样例SS。

AVC提供了从SS获取到的对访问决策的缓冲，以便最小化SELinux机制的性能开销。AVC给钩子函数提供了接口以便进行权限检查，给SS提供了接口以管理缓冲。

网络接口表 提供网络设备到安全上下文的映射关系。维护一个单独的接口表是必须的，因为LSM网络设备security field 被拒绝了。在第一次通过钩子函数查找网络设备时，他们被加入这个表格；在网络设备被关闭，或者策略重载时，网络设备从该表格中移除。网络接口表提供了一个接口（定义在include/netif.h），用于钩子函数查找并获取某个网络设备相关的SID。回调函数 则被注册，用于在设备配置改变或者策略重载时得到通知。网络接口表的代码在netif.c中。

netlink事件通知代码则用于： 在策略重载或者enforcing状态被改变时，SELinux模块通知各个进程。这些通知被用户空间AVC同内核保持一致。相关代码在netlink.c中。

selinuxfs 伪文件系统 输出安全服务策略到进程中。初期的SELinux 内核API，作为linux2.6重设计的一部分，被分解为三组正交的组件（进程属性，文件属性，策略API）。selinuxfs提供了对策略API调用的底层支持。在新内核API中，这三个组件被封装在更高层的libselinux API中。selinuxfs.c

钩子函数的实现 管理内核对象相关的安全信息，为每个内核操作执行SElinux访问控制。钩子函数调用SS和AVC来获取策略决策，使用那些决策来标记和控制内核对象。 钩子函数也调用文件系统扩展属性代码来获取和设置文件的安全上下文。这些钩子函数的代码实现位于hooks.c ,并且内核对象相关安全信息的数据结构定义在include/object.h之中。

LSM钩子函数的位置，与旧版SELinux patch的插入点位置，并不总是一一对应。LSM利用了现存的NetFilter框架来支持许多网络操作上的钩子。





# 7 初始化
SELinux的初始化由**selinux_init**函数开始，在内核初始化早期被调用；SELinux的某些方面需要延迟到内核初始化后期，由普通的initcall来处理，例如：selinux_nf_ip_init,sel_netif_init和init_sel_fs。SELinux 直到初始安全策略被/sbin/init加载后才基本完成，才调用**selinux_complete_init**函数宣告结束。

## 7.1 selinux_init(hook.c)

selinux_init 用于处理SELinux模块额早期初始化，其主要工作包括：

* 设置initial 任务的安全状态
* 调用avc_init初始化AVC。
* 设置次安全模块到原安全模块，通常是dummy，以支持同dummy或者capabilities模块的堆叠。在第8章详细讨论
* 调用register_security函数，将SELinux注册为LSM的主安全模块。

## 7.2 selinux_nf_ip_init(hook.c)
处理SELinux NetFilter钩子的初始化。NetFilter钩子用于检查发送的报文。该函数调用nf_register_hook函数 来注册 SELinux post-routing 钩子函数，将这些钩子函数注册到NetFilter 框架，用于ipv4和ipv6。详情参见第18章。

## 7.3 selinux_nf_init(netif.h)

处理SELinux网络接口表的初始化：

* 初始化SELinux 网络接口哈希表
* 注册网络设备提醒器，以便在设备关闭时删除对应表现。
* 注册AVC回调，以便在策略重置时清空整张表。

## 7.4 selnl_init(netlink.c)
创建一个SELinux netlink套接字，并容许非root进程接收通知。

## 7.5 init_sel_fs(selinuxfs.c)

初始化selinuxfs伪文件系统。它注册了selinuxfs 文件系统类型，并创建了一个私有的内核mount点，这样创建了公开的selinuxfs 文件系统，并创建了一个特殊的null device 节点。在执行execve并改变进程上下文时，该节点 被SELinux用于关闭未授权的文件。

This function, located in the selinuxfs.c file, handles initialization of the selinuxfs pseudo filesystem.
It registers the selinuxfs filesystem type and creates a private kernel mount of selinuxfs. This results in a populated selinuxfs filesystem and sets up the special null device node used by SELinux when it closes unauthorized files upon a context-changing execve.


## 7.6 selinux_complete_init(hooks.c)

在/sbin/init完成初始策略的加载后调用，标志着SELinux初始化的结束。

* 该函数遍历先于策略加载前的超级块链表，依次调用superblock_init。
* 该函数也调用inode_doinit函数，为任何关联到超级块的现存inode设置安全结构。

# 8 堆叠
本节描述了SELinux与其他安全模块堆叠支持。LSM仅仅提供了最小的堆叠支持，为该目的提供了钩子，但是将如何堆叠的细节留给了主安全模块。当前，SELinux 安全模块作仅能做为主安全模块，对使用dummy或者capabilities作为次安全模块提供了最小支持。dummy安全模块提供了传统的superuser逻辑。

如第七章所提到的，selinux_init函数在注册SELinux安全模块之前， 初始化 次安全模块为dummy。这种方式，容许SELInux钩子函数安全的调用次钩子函数。selinux_register_security钩子函数可以设置次安全模块为另外一个模块，如capabilities。而selinux_unregister_security钩子函数 将次安全模块恢复为dummy。



# 9 SELinux API

基于LSM的SELinux API设计策略是减少 侵入性，通过setexecon和setfacreatecon先于普通系统调用设置上下文环境，从而避免引入大量专用的扩展系统调用。

# 10 hook functions：辅助函数

## 10.1 原语分配辅助函数
对于include/objsec.h中定义的安全数据结构，SELinux模块提供了 两个原语:alloc_sercurity和free_security，例如，task_alloc_security 和 task_free_security。这些帮助函数被用于alloc_sercurity和free_security, 后者可能包含更多的额外的处理。

原语alloc_security辅助函数负责分配一个合适类型的安全结构，设置back指针指向对应的内核对象数据结构，初始化安全信息，设置对象的security field以引用新的安全结构。
原语free_security 辅助函数清除security field并释放安全结构。

## 10.2 初始化辅助函数
SELinux为某些安全结构定义了初始化辅助函数，例如 inode_doinit,superblock_doinit。这些初始化辅助函数被特定SELinux hook函数调用，后面再详细讨论。

## 10.3 权限检查辅助函数
提供了一套用于内核对象和权限检查的辅助函数：解引用security field，设置辅助审计数据，调用AVC执行权限检查。辅助函数简化了很多执行权限检查的钩子函数。例如:task_has_perm,inode_has_perm和may_create

尽管这些辅助函数可能非常方便，钩子函数还是被容许自由调用AVC进行权限检查。这种情况是可能发生的，例如，某些权限检查牵扯的SID没有关联内核对象；或者某些操作要求基于相同的某些SID执行多个权限检查。




# 11 hook functions：task
## 11.1 管理task security fields
### 11.1.1 task security structure

```c++
struct task_security_struct {
	struct task_struct * task;
	u32 osid;
	u32 sid;
	u32 exec_sid;
	u32 create_sid;
	u32 ptrace_sid;
};
```

每个字段的定义如下：

|Field |Description|
|--|--|
|task| Back pointer to the associated task_struct structure.|
|osid| SID prior to the last execve.|
|sid| current SID for the task.|
|exec_sid| SID for the task upon the next execve call.|
|create_sid| SID for files created by the task.|

### 11.1.2 task_alloc_security and task_free_security



### 11.1.3 selinux_task_reparent_to_init
内核函数reparent_to_init调用该hook 来设置某个内核task的安全属性。该hook 首先调用次安全模块以支持capabilities，然后设置task SID为初始SID kernel。


### 11.1.4. selinux_task_post_setuid
在setuid操作成功后调用。既然SELinux模块不适用Linux identify 属性（uid、gid），因此该hook函数不执行任何SELInux处理，而是调用次安全函数来支持linux capabilities。

### 11.1.5. selinux_task_to_inode
该hook函数由procfs伪文件系统调用，以设置任务相关的/proc/pid的安全状态。该函数根据task SID来设置inode SID，并将该inode 安全结构标记为已初始化的。

### 11.1.6. selinux_getprocattr
在进程读取/proc/pid/attr下某个节点时，procfs伪文件系统调用该hook 函数从安全模块来获取该进程的安全属性。如果目标任务与当前任务不同，该hook函数将首先检查getattr权限；然后，从task安全结构中提取合适的SID。如果对应SID并没有被设置（例如：没有设置显式的exec SID，任务使用默认的策略行为）,hook将返回一个零长度的结果；否则，hook调用security_sid_to_context ,获取安全上下文，将其拷贝到内核缓冲区并返回其长度。

### 11.1.7. selinux_setprocattr

在某个进程试图写/proc/pid/attr下的某个节点时， procfs调用该hook以在设置该进程的安全属性值。


## 11.2. Controlling Task Operations
### 11.2.1. Helper Functions for Checking Task Permissions
有几个辅助函数，被用以执行任务权限检查。这些函数和他们的所执行的权限检查在表2中描述。

* task_has_perm 函数检查一个task是否持有另外一个task的特定权限；
* task_has_capability 函数检查一个task是否使用特定capability的能力。注意，在sepolicy中我们已经知道，这里的主体和客体的相同的。
* task_has_system 检查task是否具有 class system中的某项权限。其target SID = Kernel
* task_has_security 检查task是否有权限使用某个selinuxfs API。其target SID = Security

除了task_has_perm，这些检查函数都基于一个task，所有target SID没有必要。

![](/img/selinux_implement_as_LSM_01.png)


从后面的源码可以看出，这些帮助函数最终调用了**avc_has_perm**系列的函数。

#### task_has_perm
```c
security/selinux/hooks.c
/*
 * Check permission between a pair of tasks, e.g. signal checks,
 * fork check, ptrace check, etc.
 * tsk1 is the actor and tsk2 is the target
 * - this uses the default subjective creds of tsk1
 */
static int task_has_perm(const struct task_struct * tsk1,
			 const struct task_struct * tsk2,
			 u32 perms)
{
	const struct task_security_struct *__tsec1, *__tsec2;
	u32 sid1, sid2;

	rcu_read_lock();
	__tsec1 = __task_cred(tsk1)->security;	sid1 = __tsec1->sid;
	__tsec2 = __task_cred(tsk2)->security;	sid2 = __tsec2->sid;
	rcu_read_unlock();
	return avc_has_perm(sid1, sid2, SECCLASS_PROCESS, perms, NULL);
}

```

#### task_has_capability(cred_has_capability)
在kernel 3.4中，task_has_capability被cred_has_capability 取代。

security/selinux/hooks.c

```c
/* Check whether a task is allowed to use a capability. */
static int cred_has_capability(const struct cred * cred,
			       int cap, int audit)
{
	struct common_audit_data ad;
	struct av_decision avd;
	u16 sclass;
	u32 sid = cred_sid(cred);
	u32 av = CAP_TO_MASK(cap);
	int rc;

	ad.type = LSM_AUDIT_DATA_CAP;
	ad.u.cap = cap;

	switch (CAP_TO_INDEX(cap)) {
	case 0:
		sclass = SECCLASS_CAPABILITY;
		break;
	case 1:
		sclass = SECCLASS_CAPABILITY2;
		break;
	default:
		printk(KERN_ERR
		       "SELinux:  out of range capability %d\n", cap);
		BUG();
		return -EINVAL;
	}

	rc = avc_has_perm_noaudit(sid, sid, sclass, av, 0, &avd);
	if (audit == SECURITY_CAP_AUDIT) {
		int rc2 = avc_audit(sid, sid, sclass, av, &avd, rc, &ad, 0);
		if (rc2)
			return rc2;
	}
	return rc;
}
```
#### task_has_system
security/selinux/hooks.c

```c
/* Check whether a task is allowed to use a system operation. */
static int task_has_system(struct task_struct *tsk,
			   u32 perms)
{
	u32 sid = task_sid(tsk);

	return avc_has_perm(sid, SECINITSID_KERNEL,
			    SECCLASS_SYSTEM, perms, NULL);
}
```

#### task_has_security
security/selinux/selinuxfs.c
```c
/* Check whether a task is allowed to use a security operation. */
static int task_has_security(struct task_struct *tsk,
			     u32 perms)
{
	const struct task_security_struct *tsec;
	u32 sid = 0;

	rcu_read_lock();
	tsec = __task_cred(tsk)->security;
	if (tsec)
		sid = tsec->sid;
	rcu_read_unlock();
	if (!tsec)
		return -EACCES;

	return avc_has_perm(sid, SECINITSID_SECURITY,
			    SECCLASS_SECURITY, perms, NULL);
}
```
### 11.2.2. Hook Functions for Controlling Task Operations

![](/img/selinux_implement_as_LSM_02.png)
```c
static int selinux_task_setpgid(struct task_struct *p, pid_t pgid)
{
	return current_has_perm(p, PROCESS__SETPGID);
}

static int selinux_task_getpgid(struct task_struct *p)
{
	return current_has_perm(p, PROCESS__GETPGID);
}
```


这些钩子函数中，只有3个需要进一步解释:

* selinux_task_kill 钩子函数检查 当前task是否有权限向目标task发送指定的信号。
* selinux_task_wait 检查当前task是否有权限等待子task的exit信号，子task可能与当前task有着不同的SID。
	* 就这两个钩子函数而言，SIGKILL和SIGSTOP有着他们自己独立的权限，因为他们都不能被阻塞。
	* SIGCHLD 有着单独的权限，因为其通常由子进程发送给父进程。
	* signull权限用于检查是否信号0 被传递给kill，因为它纯粹代表了一种存在测试，而不是分发实际信号。
	* 对于所有的其他信号，则使用通用的signal权限。

* selinux_task_rlimit 钩子函数在试图修改硬限制（hardlimit）时检查setrlimit权限，以便硬限制稍后可以被用于软限制（softlimit）上下文转换时的一个安全的reset点。请查看selinux_bprm_creds 关于资源限制继承控制的进一步讨论。

此外，检查ptrace权限时，selinux_ptrace钩子函数也在子任务的security structure中设置了tracer的SID，以便稍后由selinux_bprm_apply_creds和selinux_setprocattr使用。请移步12.1.4和11.1.7查看更多信息。

有几个task钩子函数并不由SELinux安全模块使用，

• selinux_task_setuid
• selinux_task_setgid
• selinux_task_setgroups
• selinux_task_prctl

由于SELinux不依赖于Linux 身份属性，并且既然这些操作仅影响当前进程，所以目前SELinux没有控制这些操作。这些操作的特权方面也已经通过了selinux_capable钩子函数进行了控制。**然而，SELinux也可能在将来控制这些钩子，以在更好的粒度上约束Linux身份更改动作。**

# 12 hook functions：程序加载
S ELinux binprm 钩子函数 实现了对structure linux_binprm的安全字段的管理，并执行程序加载操作的访问控制检查。

## 12.1 管理binprm安全字段
### 12.1.1 binprm安全结构
```c++
struct bprm_security_struct {
	struct linux_binprm *	bprm;//Back pointer to the associated linux_binprm structure.
	u32 sid;                   //SID for the transformed process.
	unsigned char set;         //Flag indicating whether sid has been set.
	char unsafe;               //Flag indicating whether an unsafe transition was attempted.
};
```

### 12.1.2. selinux_bprm_alloc_security and selinux_bprm_free_security
### 12.1.3. selinux_bprm_set_security

在加载新程序时，调用selinux_bprm_set_security钩子函数用以填充linux_binprm安全字段， 并条件执行权限检查。


在execve脚本时，该钩子函数可能会标调用多次。该函数首先调用辅助安全模块以支持capabilities。如果bprm安全结构中的set 标志已经在先前的对本钩子函数的调用中被设置，这个钩子函数将直接返回。如果策略批准，这将容许在执行脚本时安全转换。自然，这样的转换将仅仅发上在 调用者比新域更可信的情况下，。。。因为脚本是极其可疑的。然而，SELinux容许符合策略的脚本的暗转转换。

	This hook function may be called multiple times during a single execve, e.g. for interpreted scripts. This hook function first calls the secondary security module to support Linux capabilities. If the set flag in the bprm security structure has already been set by a prior call to this hook, this hook merely returns without further processing. This allows security transitions to occur on scripts if permitted by the policy. Naturally, such transitions should only occur when the caller is more trusted than the new domain, as script invocation is subject to an inherent race and scripts are highly susceptible to influence by their caller. However, SELinux does allow transitions on scripts subject to policy, e.g. to support shedding of permissions upon script invocation where the caller is trusted.

该钩子默认设置bprm结构中的SID字段为当前任务的SID。它也将清除任务先前设置的任何文件创建SID，以保证新程序以干净的初始状态启动。该钩子检查当前任务的安全结构，查看是否任务已经指定了下次execve调用的exec SID。如果是，就将使用这个exec SID，执行完毕后再清除；否则，该钩子函数咨询使用security_transition_sid来咨询SS，检查安全策略是否容许转换SID。**如果文件系统使用nosuid选项来加载，那么之前设置的exec SID 和 策略转换SID都将被忽略，新任务的SID 保持不变**。

该钩子函数然后根据SID是否改变来执行不同的权限检查。在open_exec过程中，selinux_inode_permission钩子已经检查了file execute权限，因此这里并不列出。

执行程序时，如果新任务的SID不变，将检查file execute_no_trans权限。该权限保证 一个任务被容许执行并不改变其安全属性。例如，尽管登录进程可以执行一个用户shell，它总是同时修改器SID，那么这种情形就无需execute_no_trans权限。

在任务的SID改变时，将检查process transition权限和file entrypoint 权限。前者检查旧的SID是否被容许转换到新SID；后者保证只能在执行指定程序时才能进入新的SID。

### 12.1.4. selinux_bprm_apply_creds

当执行execve时发生SID转换时，内核调用selinux_bprm_appluy_creds来设置新进程的新安全属性。该钩子函数首先调用capabilities，然后从bprm安全结构中提取新task的SID，并将当前SID拷贝到 task security structure的old SID字段，然后清除bprm security structure中的unsafe标志。如果新SID与旧SID相同，那么该钩子的工作就结束了。

当任务的SID改变时，可能发生两个额外的权限检查。如果task由clone创建，并且共享了状态，那么需要检查新旧SID间是否有share 权限。如果task正在被跟踪，那么需要检查跟踪者的task SID（保存当前任务的security structure中的 ptrace_sid字段）和新的SID之间的ptrace权限。

* 任何一个权限检查的失败，都会导致SID不会改变，并且unsafe标识被设置为真，供随后的selinux_bprm_post_apply_creds使用，然后钩子函数立即返回。
* 如果所有的权限都被授予了，钩子函数将当前SID改为新的SID，然后返回。
![](/img/selinux_implement_as_LSM_03.png)

### 12.1.5. selinux_bprm_post_apply_creds

该钩子函数在selinux_bprm_apply_creds之后被调用。它首先检查bprm安全结构中的unsafe标识，如果为真，强制向任务发送SIFKILL并立即退出。然后，它将检查task SID是否改变，如果没有，立即返回。

否则，如果任务不再被容许访问关联的tty，它将继续调用副主函数flush_unauthorized_files  来撤销对控制tty的访问，并关闭任何不再访问的fd。
** 然后，它将在每个打开的文件上调用file_has_perm，检查是否任务在新SID中是否还具有相对应的访问权限（即 fd use permission），如果没有，关闭fd。** file_has_perm描述在15.2.1节。为了避免那些期望某些描述符被打开的应用中引入错误，该辅助函数将重打开那些引用null device 节点的描述符，null device节点 在初始化时在selinuxfs中设置。

。。。


### 12.1.6. selinux_bprm_secureexec

在selinux_bprm_post_apply_creds 之后调用本钩子函数，检查ELF辅助表中AT_SECURE标识是否应该被设置，如果被设置，将使能其安全模式，这将导致libc清理执行环境和其他状态，以防止新程序被调用者以某种形式被影响。

如果任务 SID已经改变，那么该钩子函数检查新旧SID间的noatsecure权限。如果noatsecure权限被拒绝，该钩子函数将设置 AT_SECURE标识；如果权限检查通过，且UID/GID/capabilities发生改变，该钩子函数继续调用辅助安全模块看是否需要设置AT_SECURE标志；否则，该标志将不设置，libc的secure模式将不被使能。


# 13 hook functions：superblock

管理super_block structure的安全字段，执行文件系统操作的安全检查

## 13.1. Managing Superblock Security Fields

### 13.1.1. Superblock Security Structure
```c
struct superblock_security_struct {
	struct super_block* sb;//Back pointer to the associated superblock.
	struct list_head list;//Link into list of superblock security structures setup prior to initial policy load.
	u32 sid;//SID for the file system.
	u32 def_sid;//default SID for labeling
	unsigned int behavior;//Labeling behavior to apply to inodes.
	unsigned char initialized;//Flag indicating whether the security structure has been initialized.
	unsigned char proc;
	struct semaphore sem;//Semaphore used to synchronize initialization
	struct list_head isec_head;//List of inode security structures setup prior to superblock security initialization.
	spinlock_t isec_lock;//Lock for list of inode security structures.
};
```
注意， superblock的安全结构是一个链表结构，why？一个文件系统的superblock将存在多个sercurity structure?


# 14 hook functions：inode
# 15 hook functions： File
file 钩子函数管理struct file的安全字段，并为文件操作执行访问控制。每个struct file包含状态信息，例如 便宜、打开标识。既然**fd可以经由execve继承，并且也可能通过IPC传递**，因此它们可能被不同安全属性的进程跨进程共享，因此 这值得分别标记这些结构并控制其使用。此外，也有必要为SIGIO信号在这些结构中保存任务安全信息。

## 15.1 Manager File Security Fields

### 15.1.1 file_security_struct

```c
struct file_security_struct {
struct file * file;//Back pointer to the associated file.
u32 sid;//SID of the open file descriptor
u32 fown_sid;//SID of the file owner; used for SIGIO events
};
```

### 15.1.2. file_alloc_security and file_free_security
file_alloc_security 将 sid 字段设置为分配file_securityy_struct实例的任务的SID。

### 15.1.3. selinux_file_set_fowner
例如，当执行fcntl（F_SETOWN）命令时，内核调用该钩子函数，将fown_sid字段设置为当前任务的SID。

## 15.2. Controlling File Operations
### 15.2.1. file_has_perm
该辅助函数检查一个任务是否能通过fd以指定的方式访问一个文件。该函数首先设置辅助审计数据，然后调用avc检查 任务的fd use权限。如果该权限被授予，那么该函数将继续调用inode_has_perm在目标文件上检查 所请求的权限。在某些情形下，权限参数可能为空，这样就仅仅检查任务是否能够使用该fd。

```c
/* Check whether a task can use an open file descriptor to
   access an inode in a given way.  Check access to the
   descriptor itself, and then use dentry_has_perm to
   check a particular permission to the file.
   Access to the descriptor is implicitly granted if it
   has the same SID as the process.  If av is zero, then
   access to the file is not checked, e.g. for cases
   where only the descriptor is affected like seek. */
static int file_has_perm(const struct cred * cred,
			 struct file * file,
			 u32 av)
{
	struct file_security_struct * fsec = file->f_security;
	struct inode * inode = file_inode(file);
	struct common_audit_data ad;
	u32 sid = cred_sid(cred);
	int rc;

	ad.type = LSM_AUDIT_DATA_PATH;
	ad.u.path = file->f_path;

	if (sid != fsec->sid) {
		rc = avc_has_perm(sid, fsec->sid,
				  SECCLASS_FD,
				  FD__USE,
				  &ad);
		if (rc)
			goto out;
	}

	/* av is zero if only checking access to the descriptor. * /
	rc = 0;
	if (av)
		rc = inode_has_perm(cred, inode, av, &ad, 0);

out:
	return rc;
}

```


# 16 hook functions：SystemV IPC
# 17 hook functions：socket
## 17.1. Managing Socket Security Fields
## 17.1.1. Socket Security Structure

每个用户空间socket都有一个关联的inode，因此inode 安全结构 也被扩展用于socket对象。请参考14.1节查看inode 安全结构和相关函数的讨论。在网络层socket structure（struct socket）中也存在一个安全字段，但是 **这个字段只能被安全的用于local/unix 域套接字。** TCP 代码的一个修改将要求确保在新创建的server socket时对此字段的正确处理，相应的修改也包含在LSM 内核patch中，但是没能进入内核主线。

对于unix/local域套接字，该**sk_security_struct**用于存储连接建立阶段时对端的安全信息，此时连接中的用户socket还没有分配好。

![](/img/selinux_implement_as_LSM_04.png)

### 17.1.2. sk_alloc_security and sk_free_security

### 17.1.3. selinux_socket_getpeersec
This hook function is called to handle the SO_PEERSEC getsockopt option.


# 18 hook functions：IP 网络
# 19 hook functions： 其他


# 20 kernel 3.4

## task_security_struct

```c++
//old
struct task_security_struct {
	struct task_struct * task;
	u32 osid;
	u32 sid;
	u32 exec_sid;
	u32 create_sid;
	u32 ptrace_sid;
};
//new kernel3.4
struct task_security_struct {
	u32 osid;		/* SID prior to last execve * /
	u32 sid;		/* current SID * /
	u32 exec_sid;		/* exec SID *  /
	u32 create_sid;		/* fscreate SID * /
 	u32 keycreate_sid;	/* keycreate SID * /
	u32 sockcreate_sid;	/* fscreate SID * /
};
```

## selinux_capable

kernel3.4  security/selinux/hooks.c

```c++
/*
 * (This comment used to live with the selinux_task_setuid hook,
 * which was removed).
 *
 * Since setuid only affects the current process, and since the SELinux
 * controls are not based on the Linux identity attributes, SELinux does not
 * need to control this operation.  However, SELinux does control the use of
 * the CAP_SETUID and CAP_SETGID capabilities using the capable hook.
 */

static int selinux_capable(const struct cred * cred, struct user_namespace * ns,
			   int cap, int audit)
{
	int rc;

	rc = cap_capable(cred, ns, cap, audit);
	if (rc)
		return rc;

	return cred_has_capability(cred, cap, audit);
}

```
