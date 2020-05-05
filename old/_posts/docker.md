---
title: docker
date: 2019-09-17 14:08:49
tags:
---
1 docker原理篇
---
>- Linux容器（LXC）基础上进行封装
[linux的LXC技术](https://www.jianshu.com/p/63fea471bc43)
>- Cgroups&namespace来提供容器隔离
>
>>- namespace提供独立的进程命名空间、IPC、新的网络命名空间(一个命名空间对应一个网络设备，通过实现虚拟网络设备与实际网络设备通讯)、新的mount挂载点命名空间、UTS/UNIX Timesharing System（setting hostname, domainname will not affect rest of the system (CLONE_NEWUTS flag)）
>>- Cgroups(control group)用于控制物理资源，不同组占比不同的cpu、mem; 按比例，树形结构，一个进程只能属于一个group
>>- 资源管理通过Cgroup控制
>>- 隔离通过namespace控制
[linux系统实现: clone函数](https://blog.51cto.com/tasnrh/1760061)
[Cgroup解释](https://www.cnblogs.com/lisperl/archive/2012/04/17/2453838.html)

>- union文件系统
>
>>- 层级结构：[union文件系统](https://www.jianshu.com/p/5ec3d4dbf580)

[docker原理简书](https://www.jianshu.com/p/7a58ad7fade4)
[docker命令图解](http://dockone.io/article/783)
[docker](https://mp.weixin.qq.com/s/zyDGaT6SGFUVU60r9L7S3Q)