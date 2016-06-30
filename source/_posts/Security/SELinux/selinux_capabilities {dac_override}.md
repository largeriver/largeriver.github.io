---
title: "capabilities {dac_override}"
date: 2016-05-19 11:21:49
categories:
- security

tags:
- SELinux
- capabilities
---


Linux支持Capability的主要目的是细化root的特权，使一个进程能够以“最小权限原则”去执行任务。比如拿ping程序来说，它需要使用原始套接字(raw_sockets)，如果没有Capability，那么它就需要使用root特权才能运行；如果有了Capability机制，由于该程序只需要一个CAP_NET_RAW的Capability即可运行，那么根据最小权限原则，该程序运行时可以丢弃所有多余的Capability，以防止被误用或被攻击。所以，Capability机制可以将root特权进行很好的细分，当前kernel(2.6.18)已支持30多种不同的Capability。注意在之前的kernel实现中，Capability只能由root进程持有，非root进程是不能保持任何Capability的。但是在2.6.24及以上的kernel版本中一个普通用户进程也将可以持有capability。
capabilities安全模型用来管理 root进程的权限集，防止root用户的任意妄为。而现在的capabilities 实际上是基于selinux LSM。因此，selinux 中class capability对应于Linux中的capabilities。

而dac_override ，则用来容许进程旁路的所有DAC权限：uid，gid，ACL 等等。
<!--more-->

>注意：capabilities 和 ACL （access control lists） 实际上是非常对称的两种权限控制模型：在capabilities，授权信息 绑定到主体，基于行（row）；在ACL中，授权信息绑定到客体，基于列（column）。

# 定义

```
#Used to manage the Linux capabilities granted to root processes. 
#Taken from the header file: /usr/include/linux/capability.h
class capability
{
...
#Overrides all DAC including ACL execute access.
dac_override
...
}
```


在man capabilities中，该权限的对应描述是：

**CAP_DAC_OVERRIDE**：Bypass file read, write, and execute permission checks.  (DAC is an abbreviation of "discretionary access control".)



#HOOK函数


```c++
//kernel/fs/namei
/**
 * generic_permission -  check for access rights on a Posix-like filesystem
 * @inode:      inode to check access rights for
 * @mask:       right to check for (%MAY_READ, %MAY_WRITE, %MAY_EXEC, ...)
 *
 * Used to check for read/write/execute permissions on a file.
 * We use "fsuid" for this, letting us set arbitrary permissions
 * for filesystem access without changing the "normal" uids which
 * are used for other things.
 *
 * generic_permission is rcu-walk aware. It returns -ECHILD in case an rcu-walk
 * generic_permission is rcu-walk aware. It returns -ECHILD in case an rcu-walk
 * request cannot be satisfied (eg. requires blocking or too much complexity).
 * It would then be called again in ref-walk mode.
 */
int generic_permission(struct inode *inode, int mask)
{
        int ret;

        /*
         * Do the basic permission checks.
         */
        ret = acl_permission_check(inode, mask);
        if (ret != -EACCES)
                return ret;

        if (S_ISDIR(inode->i_mode)) {
                /* DACs are overridable for directories */
                if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
                        return 0;
                if (!(mask & MAY_WRITE))
                        if (capable_wrt_inode_uidgid(inode,
                                                     CAP_DAC_READ_SEARCH))
                                return 0;
                return -EACCES;
        }
        /*
         * Read/write DACs are always overridable.
         * Executable DACs are overridable when there is
         * at least one exec bit set.
         */
        if (!(mask & MAY_EXEC) || (inode->i_mode & S_IXUGO))
                if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
                        return 0;

        /*
         * Searching includes executable on directories, else just read.
         */
        mask &= MAY_READ | MAY_WRITE | MAY_EXEC;
        if (mask == MAY_READ)
                if (capable_wrt_inode_uidgid(inode, CAP_DAC_READ_SEARCH))
                        return 0;

        return -EACCES;
}

/**
 * may_follow_link - Check symlink following for unsafe situations
 * @link: The path of the symlink
 * @nd: nameidata pathwalk data
 *
 * In the case of the sysctl_protected_symlinks sysctl being enabled,
 * CAP_DAC_OVERRIDE needs to be specifically ignored if the symlink is
 * in a sticky world-writable directory. This is to protect privileged
 * processes from failing races against path names that may change out
 * from under them by way of other users creating malicious symlinks.
 * It will permit symlinks to be followed only when outside a sticky
 * world-writable directory, or when the uid of the symlink and follower
 * match, or when the directory owner matches the symlink's owner.
 *
 * Returns 0 if following the symlink is allowed, -ve on error.
 */
```

# 应用
通常，非root程序的capabilities的有效集为空。如果程序没有setuid属性，但是需要某些root特权才能工作，这就是capabilities大显身手的地方了。

下面使用cap_dac_override对于普通程序的例子，测试环境为ubuntu：
1）首先，将系统setuid-root程序beep做一个拷贝，然后去除特权。现在，beep2就是一个非root进程，这样执行时就会因为权限不足而失败。

```
$ ls -l /usr/bin/beep
-rwsr-xr-x 1 root audio 10392 Jun 11  2012 /usr/bin/beep
$ cp /usr/bin/beep beep2
$ chmod a-s ./beep2
$ ./beep2
Could not open /dev/tty0 or /dev/vc/0 for writing
open: No such file or directory
```

2） 然后调用setcap给beep授予赋予cap_dac_override和cap_sys_tty_config：
```
$ sudo setcap cap_dac_override,cap_sys_tty_config+ep ./beep2
$ ./beep2  
```
3）我们可以看看beep和beep2的capability有什么不同。
```
$getcap ./beep2
./beep = cap_dac_override,cap_sys_tty_config+ep
$ getcap /usr/bin/beep
--nothing--
```
我们可以发现beep的有效集和许可集都为空。


>关于setcap cap_dac_override,cap_sys_tty_config+ep的解释：
>
-   \+ 增加
+  e 代表有效集
+  p 代表许可集
+  cap_dac_override 对应于 CAP_DAC_OVERRIDE
+  cap_sys_tty_config 对应于 CAP_SYS_TTY_CONFIG 在虚拟终端上，执行各种特权ioctl操作。
     

#Android 


在AndroidL中，分析sepolicy，具有dac_override权限的系统服务主要包括：

* ueventd.te(7):allow ueventd self:capability { chown mknod net_admin setgid fsetid sys_rawio dac_override fowner };
* zygote.te(7):allow zygote self:capability { dac_override setgid setuid fowner chown };
* netd.te(44):allow netd self:capability { dac_override chown fowner };
* runas.te(14):dontaudit runas self:capability dac_override;
* vold.te(20):allow vold self:capability { net_admin dac_override mknod sys_admin chown fowner fsetid };
* installd.te(6):allow installd self:capability { chown dac_override fowner fsetid setgid setuid };
* tee.te(9):allow tee self:capability { dac_override };

> tip： class capability 仅作用于进程自身，这就是上述规则中的客体都是self的原因。

**需要注意的是，在AndroidL中，即使是root进程，在使用到capability时，也需要显式申请。** 

>我会在另外一篇文章中详细探讨。


----------------------
#参考文献

1. [Comparing ACLs and Capabilities](http://www.eros-os.org/essays/ACLSvCaps.html)
2. [Passing capabilities through exec](http://unix.stackexchange.com/questions/128394/passing-capabilities-through-exec)
3. [CAP_DAC_OVERRIDE - ArchWiki](https://wiki.archlinux.org/index.php/Capabilities)
4. [Exploiting capabilities](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=4&cad=rja&uact=8&ved=0CDMQFjAD&url=http://packetstorm.foofus.com/papers/attack/exploiting_capabilities_the_dark_side.pdf&ei=lOWQVYbTN4jR-QGfhILoDw&usg=AFQjCNHqdejGbVmIrxh1DxAOXrqJvKzhYQ&sig2=Vh2pu-zeZZ45fFGKydPPXg&bvm=bv.96783405,d.cWw)