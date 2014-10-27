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

2.安装好Ubuntu后，修改相应的网卡ip地址

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