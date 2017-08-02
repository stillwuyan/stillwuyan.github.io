---
title: UEFI安装CentOS 7
date: 2017-02-04 16:35:22
tags: [UEFI, CentOS]
categories: Technology
---
目前，大部分主板已经使用UEFI替代了BIOS，由于第一次在UEFI环境下安装CentOS 7，因此遇到很多坑，在网上查找答案的过程中发现很少有人将完整的安装步骤分享出来，因此利用本文记录一下。

## 1. 使用U盘制作安装盘
在[CentOS官网](https://www.centos.org/download/)下载到相应版本的ISO文件后，使用[Rufus](https://rufus.akeo.ie)可以很容易做成USB安装盘，Rufus的使用方法参考下图即可，注意图中使用的是Ubuntu镜像，请替换成CentOS镜像。
![Rufus使用说明](rufus_en_2x.png)

## 2. 引导U盘显示安装菜单
+ 在机器上插入USB安装盘，然后设置BIOS为USB优先启动，保存重启后等待进入USB引导界面。
+ CentOS 7已经支持UEFI的[Secure Boot][1]模式，如果是7以前的版本需要在BIOS中关闭Secure Boot。
+ 如果开机过程中没有进入USB引导界面，可以尝试在BIOS中关闭[Fast Boot][2]。
+ USB引导成功后，选择“Install CentOS 7”即进入系统安装界面。

## 3. 安装CentOS 7及相关设置
+ 根据CentOS 7安装界面的提示信息基本可以完成全部相关设置，这里仅介绍一下分区设置。
+ 必须创建独立的`/boot/efi`分区，UEFI就是使用该分区保存的信息引导系统的。分区大小为300M~500M之间即可。
+ 其他分区设置信息：
+ `/`分区，100G。
+ `/var`分区，200G。由于docker等使用本路径存储镜像和容器，因此独立作为一个分区。另外，`/tmp`和`/opt`分别用于存放临时文件和第三方软件，可以根据使用情况考虑是否独立成区。
+ `swap`分区，具体size根据内存大小而定。
+ `/home`分区，使用所有剩余空间（不考虑多系统情况）。

## 4. 设置UEFI引导信息
正常情况下，CentOS安装成功并重启机器后，就可以进入系统登陆界面。但是由于机器预装Windows系统，因此该机器的UEFI不支持其他操作系统启动，因此需要变更UEFI引导信息，使用Windows引导项引导CentOS系统。
+ 使用USB安装盘进入修复模式。
+ 使用`chroot`设置`/`为安装系统的`/`路径: `chroot /mnt/sysimage`
+ 可以使用`efibootmgr -v`命令查看UEFI引导信息。
+ [变更Windows引导项内容][3]： `efibootmgr -c -d /dev/sda -p 1 -L "Windows Boot Manager" -l '\EFI\centos\grubx64.efi'`

[1]: https://wiki.centos.org/HowTos/UEFI "UEFI support"
[2]: http://www.rodsbooks.com/linux-uefi "Linux on UEFI"
[3]: https://forums.lenovo.com/t5/ThinkStation-Workstations/UEFI-Mode-installation-of-Linux-distributions-on-Thinkstation/td-p/1018555 "UEFI Mode installation of Linux distributions"