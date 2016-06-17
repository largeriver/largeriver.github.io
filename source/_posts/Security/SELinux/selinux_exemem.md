---
title: "SELinux: 什么是process::execmem权限？"
date: 2015-06-19 11:21:49
categories:
- security

tags:
- SELinux
---


# 定义

	class process
	{
	...
		execmem
		execstack
		execheap
	...
	}

- **execmem**
Make executable an anonymous mapping or private file mapping that is
writable
- **execheap**: Make the heap executable.
- **execstack**: Make the main process stack executable.
<!-- more -->
# 相关allow规则

	tios_zygote.te(69):allow tios_zygote self:process execmem;
	tios_radio.te(58):allow tios_radio self:process execmem;
	tiosappdomain.te(3):allow tiosappdomain self:process execmem;
	tios_unconfined.te(11):allow tios_unconfined self:process execmem;

	recovery.te(63):  allow recovery self:process execmem;
	system_server.te(10):allow system_server self:process execmem;
	healthd.te(36):allow healthd self:process execmem;
	dumpstate.te(84):allow dumpstate self:process execmem;
	mediaserver.te(17):allow mediaserver self:process execmem;
	app.te(10):allow appdomain self:process execmem;

在android中,execmem 主要与dalvik有关：

	system_server.te
	...
	# Dalvik Compiler JIT Mapping.
	allow system_server self:process execmem;
	allow system_server ashmem_device:chr_file execute;
	allow system_server system_server_tmpfs:file execute;
	....
# 解释
## English
* **execmem** is purely a task-self check, i.e. a process can either make an anonymous mapping executable (and thus execute arbitrary code) or not.
* **execmod** is a task-file check to allow finer granularity for the case of text relocations; it is applied when a process attempts to make a modified private file mapping executable, which normally only occurs for text relocations.  Thus, under strict policy, execmod is normally restricted to a particular file type (texrel_shlib_t) and all files requiring text relocation must be explicitly labeled with that type in order to allow the relocation.  allow_execmod just controls whether or not execmod is _ever_ allowed, but even when it is enabled, you are still limited to texrel_shlib_t.
## 中文版解释
* execmem纯粹是任务自身的一种检查，例如，进程可以设置或者取消匿名映射内存 的可执行能力。如果设置该能力，也就可以执行任意代码。
* execmode则是对文本重定位（text relocation）的更好粒度的任务文件检查。当进程试图将修改过的私有文件映射变为可执行时（通常仅发生于文本重定位），执行该检查。因此，在策略限制下，execmod用于限制特定的文件类型（texrel_shlib_t）,所有要求文本重定位的文件必须显示标记为该类型。allow_execmod 仅控制execmod是否 ”曾经“被容许，但即使被allow，其文本类型仍限制为texrel_shlib_t.

# 参考文献：

* [allow execmod and execmem for self debugging process](https://www.redhat.com/archives/fedora-selinux-list/2005-June/msg00152.html)
