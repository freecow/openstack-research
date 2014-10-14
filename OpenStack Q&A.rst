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
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

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
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**解决步骤：**  

由于是nova服务出错，应登录mysql数据库，删除nova数据库中对应的services表的记录::
 
 mysql -u root -p
 	
 use nova;
 select * from services;
 delete from services where host='XXXX';
 		
 exit

XXXX换成nova-manager service list中，对应的错误主机名称主机名称