######################################
OpenStack Ubuntu Install&Configure Q&A
######################################

本文主要是收集团队在实施OpenStack中所发生的问题及相应的分析和解决方法。

文档以OpenStack官方手册为最终参考依据。

:Version: 1.0
:Authors: Zhang Hui
:License: Apache License Version 2.0
:Keywords: OpenStack, IceHouse, Ubuntu

===========================================


Admin用户登录Horizon后左侧无项目面板，换成Demo用户左侧面板全无
++++++++++++++++++++++++++++++++++++++++++++++++++++++

**分析：** 可能是对应上述用户或租户的endpoint创建有问题

**解决步骤：**

1.从命令行登录controller主机，执行sudo su

2.检查endpoint的创建情况，找出错误的endpoint::
 
 keystone endpoint-list
 
执行后应显示出3个endpoint，即基于9292端口的glance、基于5000端口的keystone、基于8774端口的Nova，但结果显示多出了两个侦听在其他ip地址5000端口的keystone服务，怀疑是创建keystone endpoint时错误所致

3.删除错误的endpoint::
 
 keystone endpoint-delete <endpoint-id>
 
endpoint-id换成keystone endpoint-list执行结果中，显示的错误对应id的字段内容

4.浏览器重新登录Horizon，正常


执行nova-manager service list，显示出错误的hostname且服务状态为XXX
+++++++++++++++++++++++++++++++++++++++++

**解决步骤：**  

由于是nova服务出错，应登录mysql数据库，删除nova数据库中对应的services表的记录::
 
 mysql -u root -p
 	
 use nova;
 select * from services;
 delete from services where host='XXXX';
 		
 exit

XXXX换成nova-manager service list中，对应的错误主机名称主机名称


在ESXi5.5的Ubuntu虚机中执行kvm-ok，显示不支持
++++++++++++++++++++++++++++++++++++++++++

**分析：** 
在 VMware ESXi 虚拟机上运行虚拟机，被称为多层虚拟或者嵌套虚拟机（Nested VMs）。如果机器缺乏但需要测试 OpenStack，使用 VMware ESXi 虚拟几个运行 KVM Hypervisor 的 OpenStack 计算节点是个不错的办法。 但默认情况下不支持嵌套虚拟，执行kvm-ok会报出以下信息::

 kvm-ok 
 
 INFO: Your CPU does not support KVM extensions
 KVM acceleration can NOT be used

**解决步骤：**

1.编辑对应虚拟机的选项（需要先关闭虚拟机），打开编辑设置对话框，在选项页面的常规选项里把客户机操作系统的类型换成其他里面的 VMware ESxi 5.x

2.到ESXi的主机数据存储中，找到对应虚拟机的文件夹，将虚机的vmx文件下载出来，vmx文件名一般对应虚机的名字，编辑此文件，在最后加入一行vhv.enable = "true"，保存后再将其传回ESXi原目录中


如何在Ubuntu中挂载和卸载外部iSCSI存储的VMWARE卷
++++++++++++++++++++++++++++++

**解决步骤：**

安装iSCSI客户端::

 apt-get install open-iscsi 


查找外部存储ip地址的target::

 iscsiadm -m discovery -t sendtargets -p 172.16.10.33:3260
 
 172.16.10.34:3260,1 iqn.2004-04.com.qnap:ts-569l:iscsi.lunll.d78468
 172.16.10.33:3260,1 iqn.2004-04.com.qnap:ts-569l:iscsi.lunll.d78468
 172.16.10.34:3260,1 iqn.2004-04.com.qnap:ts-569l:iscsi.lunsoftware.d78468
 172.16.10.33:3260,1 iqn.2004-04.com.qnap:ts-569l:iscsi.lunsoftware.d78468


得到所需的target名称后建立连接::

 iscsiadm --mode node --targetname iqn.2004-04.com.qnap:ts-569l:iscsi.lunsoftware.d78468 --portal 172.16.10.33:3260 --login


通过fdisk -l列出设备名称(如：/dev/sdb1)，并挂载到目录(如：/nas)::

 mkdir /nas
 apt-get install vmfs-tools
 vmfs-fuse /dev/sdb1 /nas
 
 注：vmfs卷挂载后是只读的（需要什么可以用cp命令拷出来）


如需设置开机自动登录到iscsi-target::

 iscsiadm -m node -T iqn.2004-04.com.qnap:ts-569l:iscsi.lunsoftware.d78468 --port 172.16.10.33:3260 --op update -n node.startup -v automatic


如需卸载该卷::

 iscsiadm --mode node --targetname iqn.2004-04.com.qnap:ts-569l:iscsi.lunsoftware.d78468 --portal 172.16.10.33:3260 --logout


如何在VMWare WorkStation中配置OpenStack网络测试环境
++++++++++++++++++++++++++++++++++++++++++

**解决步骤：**

1.在VMWare WorkStation中点击编辑→虚拟网络编辑器，修改VMnet1（仅主机模式）的子网IP为10.0.0.0，掩码为255.255.255.0，修改VMnet8（NAT模式）的子网为192.168.10.0，掩码为255.255.255.0，检查本机的网卡IP地址，会发现VMnet1的地址变为10.0.0.1，VMnet8的地址变为192.168.10.1

2.安装好Ubuntu后，修改相应的网卡ip地址::

 vi /etc/network/interfaces
 
 auto eth0
 iface eth0 inet static
 address 192.168.10.51
 netmask 255.255.255.0
 gateway 192.168.10.2
 dns-nameservers 192.168.10.2
 
 auto eth1
 iface eth0 inet static
 address 10.0.0.51
 netmask 255.255.255.0

3.如果希望从虚机中ping通主机，则需要关闭主机Windows的防火墙或者添加允许入站ICMPv4的规则


如何扩展Ubuntu的LVM分区
+++++++++++++++++++++++++++++

**解决步骤：**

1.查看系统挂载情况::

 df -h
 
 Filesystem                       Size  Used Avail Use% Mounted on
 /dev/mapper/controller--vg-root   19G  2.2G   16G  13% /
 none                             4.0K     0  4.0K   0% /sys/fs/cgroup
 udev                             987M  4.0K  987M   1% /dev
 tmpfs                            200M  616K  199M   1% /run
 none                             5.0M     0  5.0M   0% /run/lock
 none                             998M     0  998M   0% /run/shm
 none                             100M     0  100M   0% /run/user
 /dev/sda1                        236M   66M  158M  30% /boot

系统有一个236M的boot分区和一个19G的vg分区，我们准备扩容vg分区

2.查看系统分区情况::

 fdisk -l
 
 Disk /dev/sda: 85.9 GB, 85899345920 bytes
 255 heads, 63 sectors/track, 10443 cylinders, total 167772160 sectors
 Units = sectors of 1 * 512 = 512 bytes
 Sector size (logical/physical): 512 bytes / 512 bytes
 I/O size (minimum/optimal): 512 bytes / 512 bytes
 Disk identifier: 0x000067e4
    Device Boot      Start         End      Blocks   Id  System
 /dev/sda1   *        2048      499711      248832   83  Linux
 /dev/sda2          501758    41940991    20719617    5  Extended
 /dev/sda5          501760    41940991    20719616   8e  Linux LVM

很明显/dev/sda已经被扩展到了85.9G，原vg逻辑卷创建在/dev/sda2上，只有19G

3.增加分区::

 fdisk /dev/sda
 
 Command (m for help): n
 Partition type:
    p   primary (1 primary, 1 extended, 2 free)
    l   logical (numbered from 5)
 Select (default p): p
 Partition number (1-4, default 3): 
 Using default value 3
 First sector (499712-167772159, default 499712): 
 Using default value 499712
 Last sector, +sectors or +size{K,M,G} (499712-501757, default 501757): 
 Using default value 501757
 Command (m for help): w
 The partition table has been altered!
 Calling ioctl() to re-read partition table.
 Syncing disks.

很奇怪，有些硬盘需要先分一个小的区，不怕，再执行上面的命令继续分区

4.再次增加分区::

 fdisk /dev/sda
 
 Command (m for help): n
 Partition type:
    p   primary (2 primary, 1 extended, 1 free)
    l   logical (numbered from 5)
 Select (default p): p
 Selected partition 4
 First sector (41940992-167772159, default 41940992): 
 Using default value 41940992
 Last sector, +sectors or +size{K,M,G} (41940992-167772159, default 167772159): 
 Using default value 167772159
 Command (m for help): w
 The partition table has been altered!
 Calling ioctl() to re-read partition table.
 Syncing disks.

这次分到了我们需要的大分区

5.查看分区情况::

 fdisk -l
 
 Disk /dev/sda: 85.9 GB, 85899345920 bytes
 255 heads, 63 sectors/track, 10443 cylinders, total 167772160 sectors
 Units = sectors of 1 * 512 = 512 bytes
 Sector size (logical/physical): 512 bytes / 512 bytes
 I/O size (minimum/optimal): 512 bytes / 512 bytes
 Disk identifier: 0x000067e4
    Device Boot      Start         End      Blocks   Id  System
 /dev/sda1   *        2048      499711      248832   83  Linux
 /dev/sda2          501758    41940991    20719617    5  Extended
 /dev/sda3          499712      501757        1023   83  Linux
 /dev/sda4        41940992   167772159    62915584   83  Linux
 /dev/sda5          501760    41940991    20719616   8e  Linux LVM

很明显，我们需要把vg扩展到/dev/sda4

6.使分区生效::

 partprobe


7.找到需要扩展的vg分区::

 vgdisplay
 
 --- Volume group ---
   VG Name               controller-vg
   System ID             
   Format                lvm2
   Metadata Areas        1
   Metadata Sequence No  3
   VG Access             read/write
   VG Status             resizable
   MAX LV                0
   Cur LV                2
   Open LV               2
   Max PV                0
   Cur PV                1
   Act PV                1
   VG Size               19.76 GiB
   PE Size               4.00 MiB
   Total PE              5058
   Alloc PE / Size       5053 / 19.74 GiB
   Free  PE / Size       5 / 20.00 MiB
   VG UUID               jyqjWB-aW6I-PE9b-HsyQ-iB4n-whxK-nepqOw

我们需要扩展的vg为controller-vg

8.扩展vg分区::

 vgextend controller-vg /dev/sda4
 
   No physical volume label read from /dev/sda4
   Physical volume "/dev/sda4" successfully created
   Volume group "controller-vg" successfully extended


9.检查扩展情况::

 vgdisplay
 
 --- Volume group ---
   VG Name               controller-vg
   System ID             
   Format                lvm2
   Metadata Areas        2
   Metadata Sequence No  4
   VG Access             read/write
   VG Status             resizable
   MAX LV                0
   Cur LV                2
   Open LV               2
   Max PV                0
   Cur PV                2
   Act PV                2
   VG Size               79.76 GiB
   PE Size               4.00 MiB
   Total PE              20418
   Alloc PE / Size       5053 / 19.74 GiB
   Free  PE / Size       15365 / 60.02 GiB
   VG UUID               jyqjWB-aW6I-PE9b-HsyQ-iB4n-whxK-nepqOw

显示Free PE/Size还可扩展60.02G

10.找到LV的设备名::

 lvscan
 
 ACTIVE            '/dev/controller-vg/root' [18.74 GiB] inherit
 ACTIVE            '/dev/controller-vg/swap_1' [1020.00 MiB] inherit

显示设备名为/dev/controller-vg/root

11.扩展LV卷::

 lvextend -L +40G /dev/controller-vg/root
 
 Extending logical volume root to 58.74 GiB
 Logical volume root successfully resized

先扩40G，不够未来再扩

12.检查LV扩展情况::

 lvscan
 
 ACTIVE            '/dev/controller-vg/root' [58.74 GiB] inherit
 ACTIVE            '/dev/controller-vg/swap_1' [1020.00 MiB] inherit


13.扩大文件系统分区::

 resize2fs /dev/controller-vg/root
 
 resize2fs 1.42.9 (4-Feb-2014)
 Filesystem at /dev/controller-vg/root is mounted on /; on-line resizing required
 old_desc_blocks = 2, new_desc_blocks = 4
 The filesystem on /dev/controller-vg/root is now 15398912 blocks long.


14.检查文件系统分区::

 df -h
 
 Filesystem                       Size  Used Avail Use% Mounted on
 /dev/mapper/controller--vg-root   58G  2.2G   54G   4% /
 none                             4.0K     0  4.0K   0% /sys/fs/cgroup
 udev                             987M  4.0K  987M   1% /dev
 tmpfs                            200M  624K  199M   1% /run
 none                             5.0M     0  5.0M   0% /run/lock
 none                             998M     0  998M   0% /run/shm
 none                             100M     0  100M   0% /run/user
 /dev/sda1                        236M   66M  158M  30% /boot
