---
title: Linux硬盘分区删除、创建、格式化、磁盘挂载
date: 2019-11-10 15:30:58
categories: Linux
tags:
- Linux
- 分区
- 磁盘挂载
---
Linux作为日常应用服务器，它的稳定性直接决定着业务的可靠性。今天探讨的是Linux磁盘的扩容。磁盘扩容大致有以下几个步骤：插入磁盘，建立分区，格式化分区，磁盘挂载

## 建立分区
输入`fdisk -l`，该命令可以看到当前系统有哪些磁盘，这些磁盘的容量，分区，磁盘的逻辑名称。
```shell
[root@MiWiFi-R3L-srv env]# fdisk -l

磁盘 /dev/sdb：160.0 GB, 160041885696 字节，312581808 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x8e502bee

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   312581807   156289880   83  Linux
WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.

磁盘 /dev/sda：16.0 GB, 16013942784 字节，31277232 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：gpt


#         Start          End    Size  Type            Name
 1         2048       411647    200M  EFI System      EFI System Partition
 2       411648      2508799      1G  Microsoft basic 
 3      2508800     31277055   13.7G  Linux LVM       
```
得到需要分区的磁盘名称过后，输入`fdisk /dev/sdb`，该命令可以查询有创建分区等等
```shell
[root@MiWiFi-R3L-srv env]# fdisk /dev/sdb
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。

命令(输入 m 获取帮助)：m
命令操作
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   g   create a new empty GPT partition table
   G   create an IRIX (SGI) partition table
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

命令(输入 m 获取帮助)：p

磁盘 /dev/sdb：160.0 GB, 160041885696 字节，312581808 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x8e502bee

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   312581807   156289880   83  Linux

命令(输入 m 获取帮助)：n
Partition type:
   p   primary (1 primary, 0 extended, 3 free)
   e   extended
Select (default p): p
分区号 (2-4，默认 2)：
```
在该命令中，输入m查看帮助，n：创建，在创建模式下输入p，表示创建新分区，然后输入分区编号和起始大小，w保存当前分区。
## 格式化分区
输入`mkfs.ext4 /dev/sdb1`，这里的ext4表示文件系统格式。注意：只有格式化了的分区，才会生成磁盘UUID
## 挂载磁盘分区
有的同学误认为（我之前是这样认为的）将磁盘插入服务器过后，分好区并格式化过后就可以了，其实这个时候并没有真正被使用。我们还需要将某个磁盘分区挂载到某个文件夹
输入命令`mount /dev/sdb1 /env`，表示将/dev/sdb1挂载到/env目录，这个时候我们使用`df`就可以看到/dev/sdb1被挂载到/env目录了
```shell
[root@MiWiFi-R3L-srv env]# df
文件系统                1K-块   已用      可用 已用% 挂载点
/dev/mapper/cl-root  12806144 969624  11836520    8% /
devtmpfs              1874080      0   1874080    0% /dev
tmpfs                 1885416      0   1885416    0% /dev/shm
tmpfs                 1885416   8716   1876700    1% /run
tmpfs                 1885416      0   1885416    0% /sys/fs/cgroup
/dev/sda2             1038336 134652    903684   13% /boot
/dev/sda1              204580   9672    194908    5% /boot/efi
/dev/sdb1           153704800  61464 145812460    1% /env
tmpfs                  377084      0    377084    0% /run/user/0
```
OK，大功告成，但是当服务器重启过后，挂载点将会消失。怎么办呢，我们可以将其设置为开机自动挂载
首先查看磁盘分区的UUID`blkid`，将看到如下信息
```shell
[root@MiWiFi-R3L-srv env]# blkid
/dev/mapper/cl-root: UUID="0ebb80a1-9ec8-41c4-86dc-45ae67c55de9" TYPE="xfs" 
/dev/sda3: UUID="d7HW9s-WSz0-Bjuz-CLIo-3l6A-HZvX-aeLQ59" TYPE="LVM2_member" PARTUUID="bfa0e4d2-8885-47e9-bf84-30349637ecc3" 
/dev/sda2: UUID="4cd8b6a8-8e73-4ec4-9b92-ff481c2e86d2" TYPE="xfs" PARTUUID="f3757ea1-b200-4b58-b090-b80b3fd0a64b" 
/dev/sda1: SEC_TYPE="msdos" UUID="BA05-713D" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="54e1380d-0f9a-4951-a82a-03bdfb51baea" 
/dev/mapper/cl-swap: UUID="4140b465-c638-4008-83c2-80c58e4340e3" TYPE="swap" 
/dev/sdb1: UUID="a1d5ba39-8bb7-4e26-9946-22dc5368ac85" TYPE="ext4" 
```
/dev/sdb1: UUID="a1d5ba39-8bb7-4e26-9946-22dc5368ac85" TYPE="ext4"，其中`a1d5ba39-8bb7-4e26-9946-22dc5368ac85`就是磁盘的UUID，复制一下，在`/etc/fstab`文件中添加记录
```shell
UUID=a1d5ba39-8bb7-4e26-9946-22dc5368ac85 /env  ext4    defaults        1 1
```
reboot重启，完成。