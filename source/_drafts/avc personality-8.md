avc personality-8

#问题描述
在Android6中，内核版本3.10.86，下面的avc进程出现：

	avc:  denied  { module_request } for pid=3638 comm="logd" kmod="personality-8" scontext=u:r:logd:s0 tcontext=u:r:kernel:s0 tclass=system

当加载ELF文件时，系统将调用：

	 __libc_init
	-> __libc_init_AT_SECURE
	 -> __initialize_personality


在补丁【https://android-review.googlesource.com/#/c/122131/】中，有这样的说明：解决Bug: 18069809，函数__initialize_personality 将为32bit ELF设置PER_LINUX32 标志，但是在内涵中，elf_set_personality 不支持 PER_LINUX32，因此它将导致 request_module("personality-%d", pers)，该调用将导致请求 personality-8。

#  What is "Bug: 18069809"? why we need this patch?
该问题的上下文是/proc/cpuinfo,很多应用解析该文件来检查CPU的特性，例如NEON。但是在aarch63内核中，该文件看起来差异很大。因此内核中加入了一个**[兼容性补丁](https://lists.linaro.org/pipermail/cross-distro/2014-October/000751.html)**，其中描述了What was done，以及why。

# 该补丁将导致所有的32bit ELF 去request_module personality。如何解决呢？
1. 首先，对于Android， 推荐编译时关闭内核的模块支持功能: CONFIG_MODULES=n。
2. 其次，如果你还是希望可以支持内核模块，你可以后向移植内核补丁，也就是去掉该内核补丁，以消除这种不必要的request_module调用。
3. 最后一招，你可以编写dontaudit策略，关闭其显示。然而，这样也将隐藏其他的module_request违例，甚至包含内核模块，所以你必须非常小心：确保其他合法的module_request调用没有问题。