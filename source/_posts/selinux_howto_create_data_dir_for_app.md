---
title: "SELinux: 如何为进程创建专有的数据目录？"
date: 20160528 15:22:33
categories:
- security

tags:
- SELinux
- howto
---
# 需求

在data分区中为某个原生进程（如系统服务）创建一个专有目录，用来存储该服务的数据。应该如何设定安全策略了？

------------------
# 策略1（FAILED）

先定义一个策略，该策略的意图是：cpnoui 进程在创建专有目录/data/eeprom_data_from_sysprops.tmp时,内核会根据file_contexts中的设定，直接将新目录的安全标记设定为eeprom_data_file

    file_contexts
    /data/eeprom_data_from_sysprops\.tmp  u:object_r:eeprom_data_file:s0
    ------------------------------
    file.te:
    type eeprom_data_file, file_type, data_file_type;
    ------------------------------
    cpnoui.te
    allow cpnoui eeprom_data_file:file create;


不过，该策略并不能工作，其denied日志如下：

    type=1400 audit(1463139871.920:5): avc: denied { create } for pid=167 comm="sh" name="eeprom_data_from_sysprops.tmp" scontext=u:r:cpnoui:s0 tcontext=u:object_r:system_data_file:s0 tclass=file permissive=0

WHY？

------------------
# 策略2 (just works)

**file_contexts是用户空间配置，其作用是：1）编译时，编译系统使用该文件标记初始文件系统镜像； 2）运行时，init/ueventd等程序使用该文件来决定其创建的文件系统、目录的安全标签。**

内核实际上用不到file_contexts，因此无法影响运行时的文件安全标记。要做到这一点，你可以使用type_transition规则，例如：

    type_transition cpnoui system_data_file:file eeprom_data_file;

因此，完整的策略定义是：


    file.te:
    type eeprom_data_file, file_type, data_file_type;
    ------------------------------
    cpnoui.te
    type_transition cpnoui system_data_file:dir eeprom_data_file;
    type_transition cpnoui system_data_file:file eeprom_data_file;
    allow cpnoui system_data_file:dir {  remove_name add_name };
    allow cpnoui eeprom_data_file:dir { create_dir_perms remove_name add_name };
    allow cpnoui eeprom_data_file:file { create_file_perms };



------------------        
# 策略3 （推荐）

较好的方法应该在init.<board>.rc中创建该专有目录，这种方法的好处是：

2. init 进程将会根据file_contexts来决定该专有目录和目录下的文件的安全标记。
3. cpnoui进程无需system_data_file:dir { write add_name } 权限
3. 当cpnoui进程在该专有目录中继续创建文件或者文件夹时，将自动继承父目录的安全标记。

所以，最终的策略是怎样的呢？

    file_contexts
    /data/eeprom_data_from_sysprops\.tmp  u:object_r:eeprom_data_file:s0
    -----------------------------
    file.te:
    type eeprom_data_file, file_type, data_file_type;
    ------------------------------
    cpnoui.te
    allpw cpnoui eeprom_data_file:dir rw_dir_perms;
    allow cpnoui eeprom_data_file:file create_file_perms;
    ------------------------------
    init.<board>.rc
    mkdir /data/eeprom_data_from_sysprops.tmp



------------------
# 参考链接

http://seandroid-list.tycho.nsa.narkive.com/mLU2RVyz/regarding-issue-in-defining-file-in-file-contexts
