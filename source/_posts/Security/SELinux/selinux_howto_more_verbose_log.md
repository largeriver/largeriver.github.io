---
title: "SELinux: 如何输出更详尽的日志？"
date: 2016-05-28 11:22:33
categories:
- security
tags: [SELinux, Log, howto]
---

avc内核日志是编写SELinux 安全策略的重要参考，其主要信息由进程号、source contexts、target context、class、permission等等，参见下文。不过，有时候我们还希望有着更详尽的信息。
 ><38>[   12.496008] type=1400 audit(1422757038.179:12): avc: denied { open } for pid=628 comm="mtrild_dual" path="/newplus_temp/SoftSupport/TG3SMEM_X_TG3DLL_DataArea" dev="tmpfs" ino=366 scontext=u:r:tios_radio:s0 tcontext=u:object_r:tios_unconfined_tmpfs:s0 tclass=file permissive=1

# 如何输出更详尽的avc日志呢？
我们可以启用syscall auditing 机制，
## CONFIG_AUDITSYSCALL
在内核配置文件中， **CONFIG_AUDITSYSCALL** 选项用来打开selinux audit日志系统中的 SYSCALL记录。
 默认的selinux错误日志类型为AVC记录，配置该选项后，其日志更为详细：
>node=holycross.devel.redhat.com type=AVC msg=audit(12/13/2006 11:28:14.395:952) : avc: denied { getattr } for pid=7236 comm=vsftpd name=public_html dev=dm-0 ino=9601649 scontext=system_u:system_r:ftpd_t:s0 tcontext=system_u:object_r:httpd_sys_content_t:s0 tclass=dir

>node=holycross.devel.redhat.com type=SYSCALL msg=audit(12/13/2006 11:28:14.395:952) : arch=i386 syscall=lstat64 success=no exit=0 a0=8495230 a1=849c830 a2=874ff4 a3=328d28 items=0 ppid=7234 pid=7236 auid=dwalsh uid=dwalsh gid=dwalsh euid=dwalsh suid=dwalsh fsuid=dwalsh egid=dwalsh sgid=dwalsh fsgid=dwalsh tty=(none) comm=vsftpd exe=/usr/sbin/vsftpd subj=system_u:system_r:ftpd_t:s0 key=(null)

在使能CONFIG_AUDITSYSCALL后，如果发生访问为例，内核将同时输出**AVC记录和SYSCALL记录**。 在SYSCALL日志记录中，我们可以发现主体的DAC详细信息。
## 内核命令行参数audit=1
在内核命令行参数中，增加audit=1，否则，audit默认为0，将不启用syscall auditing机制。
当然，你也可以直接修改 kernel/audit/audit.c，将audit_default 设置为1。

参考资料：
* [Why doen't SELinux give me the full path in an error message?](https://danwalsh.livejournal.com/34903.html)
* https://www.mail-archive.com/seandroid-list%40tycho.nsa.gov/msg02286.html
* [the Linux Audit Documentation Project](https://github.com/linux-audit/audit-documentation/wiki)