=======================
OpenStack Icehouse 两节点内外网设置
=======================

**OpenStack** 是一款开源的云计算平台，包括独立的计算、网络、存储管理、门户、远程桌面等模块，目前较新且稳定的版本为I版（即IceHouse版），可以说，OpenStack为IT界带来了又一次的重大挑战和机遇。

本文简单介绍了在Ubuntu 14.04基础上搭建控制和计算两节点的IceHouse。

文档以OpenStack官方手册为最终参考依据。

:Version: 1.0
:Authors: Zhang Hui
:License: Apache License Version 2.0
:Keywords: OpenStack, IceHouse, Ubuntu

===========================================

注意事项
=============
- 本文采用扁平网络（FlatDHCP）模式，未采用Neutron模式
- 网段分两部分，采用172.16.10.0/24用于各虚机之间的通讯和访问internet，采用10.0.0.0/24用于管理信息的传递


环境准备
============

配置控制节点
------------

切换为超级用户模式::

 sudo su

设置主机名::

 vi /etc/hostname
 controller

设置网卡::

 vi /etc/network/interfaces
 
 auto eth0
 iface eth0 inet static
 address 172.16.10.51
 netmask 255.255.255.0
 gateway 172.16.10.1
 dns-nameservers 172.16.10.1
 
 auto eth1
 iface eth0 inet static
 address 10.0.0.51
 netmask 255.255.255.0

添加主机::

 vi /etc/hosts

 10.0.0.51       controller
 10.0.0.52       compute1

网卡生效::
 
 ifdown eth0 && ifup eth0
 ifdown eth1 && ifup eth1
 
配置计算节点
------------

切换为超级用户模式::

 sudo su

设置主机名::

 vi /etc/hostname
 controller

设置网卡::

 vi /etc/network/interfaces
 
 auto eth0
 iface eth0 inet static
 address 172.16.10.52
 netmask 255.255.255.0
 gateway 172.16.10.1
 dns-nameservers 172.16.10.1
 
 auto eth1
 iface eth0 inet static
 address 10.0.0.52
 netmask 255.255.255.0

添加主机::

 vi /etc/hosts
 
 10.0.0.51       controller
 10.0.0.52       compute1

网卡生效::

 ifdown eth0 && ifup eth0
 ifdown eth1 && ifup eth1

验证网络连接
------------

在controller节点::

 ping compute1

在compute节点::

 ping controller


安装控制节点
============

安装基础支撑服务
---------------

更新和升级操作系统::

 apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y


安装NTP服务::

 apt-get install -y ntp


安装MySQL::

 apt-get install -y mysql-server python-mysqldb


设置MySQL的绑定地址::

 vi /etc/mysql/my.cnf
 
 bind-address = 10.0.0.51


设置MySQL激活InnoDB, 设置UTF-8字符集及UTF-8 collation选项::

 vi /etc/mysql/my.cnf
 
 [mysqld]
 default-storage-engine = innodb
 innodb_file_per_table
 collation-server = utf8_general_ci
 init-connect = 'SET NAMES utf8'
 character-set-server = utf8

重启MySQL服务::

 service mysql restart


删除MySQL首次安装完毕后建立的anonymous账户::

 mysql_install_db
 mysql_secure_installation


安装RabbitMQ服务::

 apt-get install -y rabbitmq-server


安装Keystone认证服务
--------------------

安装Keystone软件包::

 apt-get install -y keystone


创建与Keystone相关的数据库::

 mysql -u root -p
 
 CREATE DATABASE keystone;
 GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
 GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
 exit;


删除Keystone SQLite数据库::

 rm /var/lib/keystone/keystone.db


修改/etc/keystone/keystone.conf中的参数::

 vi /etc/keystone/keystone.conf
 
 [database]
 #connection = sqlite:////var/lib/keystone/keystone.db
 connection = mysql://keystone:KEYSTONE_DBPASS@controller/keystone
 
 [DEFAULT]
 admin_token=ADMIN
 log_dir=/var/log/keystone


重启Keystone服务并同步数据库::

 service keystone restart
 keystone-manage db_sync


检查数据库同步情况::

 mysql -u root -p keystone
 
 show TABLES;


创建用户、租户及角色::

 export OS_SERVICE_TOKEN=ADMIN
 export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
 
 keystone user-create --name=admin --pass=admin_pass --email=admin@geniuslab.local
 keystone role-create --name=admin
 keystone tenant-create --name=admin --description="Admin Tenant"
 keystone user-role-add --user=admin --tenant=admin --role=admin
 keystone user-role-add --user=admin --role=_member_ --tenant=admin
 
 keystone user-create --name=demo --pass=demo_pass --email=demo@geniuslab.local
 keystone tenant-create --name=demo --description="Demo Tenant"
 keystone user-role-add --user=demo --role=_member_ --tenant=demo
 
 keystone tenant-create --name=service --description="Service Tenant"


创建Keystone服务及API的入口::

 keystone service-create --name=keystone --type=identity --description="OpenStack Identity"
 
 keystone endpoint-create \
 --service-id=$(keystone service-list | awk '/ identity / {print $2}') \
 --publicurl=http://172.16.10.51:5000/v2.0 \
 --internalurl=http://controller:5000/v2.0 \
 --adminurl=http://controller:35357/v2.0


创建环境变量运行文件::

 vi creds

 export OS_TENANT_NAME=admin
 export OS_USERNAME=admin
 export OS_PASSWORD=admin_pass
 export OS_AUTH_URL=http://172.16.10.51:5000/v2.0

 vi admin_creds

 export OS_USERNAME=admin
 export OS_PASSWORD=admin_pass
 export OS_TENANT_NAME=admin
 export OS_AUTH_URL=http://controller:35357/v2.0


测试Keystone服务::

 unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
 
 keystone --os-username=admin --os-password=admin_pass --os-auth-url=http://controller:35357/v2.0 token-get

 source admin_creds
 keystone token-get
 
 source creds
 keystone user-list
 keystone user-role-list --user admin --tenant admin


安装Glance镜像服务
------------------

安装Glance软件包::

 apt-get install -y glance python-glanceclient


创建与Glance相关的数据库::

 mysql -u root -p
 
 CREATE DATABASE glance;
 GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
 GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
 exit;


配置Service用户及角色::

 keystone user-create --name=glance --pass=service_pass --email=glance@domain.com
 keystone user-role-add --user=glance --tenant=service --role=admin


创建Glance服务及API的入口::

 keystone service-create --name=glance --type=image --description="OpenStack Image Service"
 
 keystone endpoint-create \
 --service-id=$(keystone service-list | awk '/ image / {print $2}') \
 --publicurl=http://172.16.10.51:9292 \
 --internalurl=http://controller:9292 \
 --adminurl=http://controller:9292


修改/etc/glance/glance-api.conf::

 vi /etc/glance/glance-api.conf
 
 [database]
 #sqlite_db = /var/lib/glance/glance.sqlite
 connection = mysql://glance:GLANCE_DBPASS@controller/glance
 
 [DEFAULT]
 rpc_backend = rabbit
 rabbit_host = controller
 
 [keystone_authtoken]
 auth_uri = http://controller:5000
 auth_host = controller
 auth_port = 35357
 auth_protocol = http
 admin_tenant_name = service
 admin_user = glance
 admin_password = service_pass
 
 [paste_deploy]
 flavor = keystone


修改/etc/glance/glance-registry.conf::

 vi /etc/glance/glance-registry.conf
 
 [database]
 #sqlite_db = /var/lib/glance/glance.sqlite with:
 connection = mysql://glance:GLANCE_DBPASS@controller/glance
 
 [keystone_authtoken]
 auth_uri = http://controller:5000
 auth_host = controller
 auth_port = 35357
 auth_protocol = http
 admin_tenant_name = service
 admin_user = glance
 admin_password = service_pass
 
 [paste_deploy]
 flavor = keystone


重启glance-api和glance-registry服务::

 service glance-api restart; service glance-registry restart


同步glance数据库::

 glance-manage db_sync


上传测试镜像文件::

 apt-get install -y lrzsz
 
 rz
 选择下载好的cirros-0.3.2-x86_64-disk.img上传

创建测试镜像::

 source creds
 glance image-create --name "cirros-0.3.2-x86_64" --is-public true \
 --container-format bare --disk-format qcow2 \
 --file cirros-0.3.2-x86_64-disk.img

列出镜像::

 glance image-list


安装Nova计算服务
---------------

安装Nova软件包::

 apt-get install -y nova-api nova-cert nova-conductor nova-consoleauth \
 nova-novncproxy nova-scheduler python-novaclient


创建与Nova相关的数据库::

 mysql -u root -p
 
 CREATE DATABASE nova;
 GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
 GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
 exit;


配置Service用户及角色::

 keystone user-create --name=nova --pass=service_pass --email=nova@geniuslab.local
 keystone user-role-add --user=nova --tenant=service --role=admin


创建Glance服务及API的入口::

 keystone service-create --name=nova --type=compute --description="OpenStack Compute"
 
 keystone endpoint-create \
 --service-id=$(keystone service-list | awk '/ compute / {print $2}') \
 --publicurl=http://172.16.10.51:8774/v2/%\(tenant_id\)s \
 --internalurl=http://controller:8774/v2/%\(tenant_id\)s \
 --adminurl=http://controller:8774/v2/%\(tenant_id\)s


修改/etc/nova/nova.conf::

 vi /etc/nova/nova.conf
 
 [database]
 connection = mysql://nova:NOVA_DBPASS@controller/nova
 
 [DEFAULT]
 rpc_backend = rabbit
 rabbit_host = controller
 my_ip = 10.0.0.51
 vncserver_listen = 10.0.0.51
 vncserver_proxyclient_address = 10.0.0.51
 auth_strategy = keystone
 
 [keystone_authtoken]
 auth_uri = http://controller:5000
 auth_host = controller
 auth_port = 35357
 auth_protocol = http
 admin_tenant_name = service
 admin_user = nova
 admin_password = service_pass


删除Nova SQLite数据库::

 rm /var/lib/nova/nova.sqlite


同步Nova数据库::

 nova-manage db sync


重启Nova服务::

 service nova-api restart
 service nova-cert restart
 service nova-conductor restart
 service nova-consoleauth restart
 service nova-novncproxy restart
 service nova-scheduler restart


检查Nova服务是否已正常运行::

 nova-manage service list
 
 图案:-)表示已正常

检查可用镜像::

 source creds
 nova image-list



配置网络
--------

修改/etc/nova/nova.conf文件::

 vi /etc/nova/nova.conf
 
 [DEFAULT]
 network_api_class = nova.network.api.API
 security_group_api = nova


重新启动Nova服务::

 service nova-api restart
 service nova-scheduler restart
 service nova-conductor restart


安装Horizon（Dashboard）服务
----------------------------

安装所需软件包::

 apt-get install -y apache2 memcached libapache2-mod-wsgi openstack-dashboard

如果apache2启动报错，可修改/etc/apache2/apache2.conf，在最后一行添加ServerName localhost


删除openstack-dashboard-ubuntu-theme::

 apt-get remove -y --purge openstack-dashboard-ubuntu-theme


修改/etc/openstack-dashboard/local_settings.py文件::

 vi /etc/openstack-dashboard/local_settings.py
 
 ALLOWED_HOSTS = '*'
 OPENSTACK_HOST = "controller"


重新启动服务::

 service apache2 restart; service memcached restart


测试：打开http://172.16.10.51/horizon，登录admin/admin_pass


安装计算节点
============

更新和升级操作系统::

 apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y


安装NTP服务::

 apt-get install -y ntp
 sed -i 's/server ntp.ubuntu.com/server controller/g' /etc/ntp.conf
 service ntp restart


检查kvm是否支持::

 apt-get install -y cpu-checker
 kvm-ok


安装kvm软件包::

 apt-get install -y kvm libvirt-bin pm-utils
 
 如果不支持kvm，需再安装如下：
 apt-get install -y qemu-kvm 


安装Compute软件包::

 apt-get install -y nova-compute-kvm python-guestfs


激活当前Kernel可读::

 dpkg-statoverride  --update --add root root 0644 /boot/vmlinuz-$(uname -r)


激活未来Kernel升级后仍可读::

 vi /etc/kernel/postinst.d/statoverride
 
 #!/bin/sh
 version="$1"
 # passing the kernel version is required
 [ -z "${version}" ] && exit 0
 dpkg-statoverride --update --add root root 0644 /boot/vmlinuz-${version}


修改文件权限::

 chmod +x /etc/kernel/postinst.d/statoverride


修改/etc/nova/nova.conf文件::

 vi /etc/nova/nova.conf
 
 [DEFAULT]
 auth_strategy = keystone
 rpc_backend = rabbit
 rabbit_host = controller
 my_ip = 10.0.0.52
 vnc_enabled = True
 vncserver_listen = 0.0.0.0
 vncserver_proxyclient_address = 10.0.0.52
 novncproxy_base_url = http://172.16.10.51:6080/vnc_auto.html
 glance_host = controller
 
 [database]
 connection = mysql://nova:NOVA_DBPASS@controller/nova
 
 [keystone_authtoken]
 auth_uri = http://controller:5000
 auth_host = controller
 auth_port = 35357
 auth_protocol = http
 admin_tenant_name = service
 admin_user = nova
 admin_password = service_pass


删除 /var/lib/nova/nova.sqlite文件::

 rm /var/lib/nova/nova.sqlite


安装基本网络组件::

 apt-get install -y nova-network nova-api-metadata


修改/etc/nova/nova.conf文件::

 vi /etc/nova/nova.conf
 
 network_api_class = nova.network.api.API
 security_group_api = nova
 network_size = 254
 allow_same_net_traffic = False
 multi_host = True
 send_arp_for_ha = True
 share_dhcp_address = True
 force_dhcp_release = True
 firewall_driver = nova.virt.libvirt.firewall.IptablesFirewallDriver
 network_manager = nova.network.manager.FlatDHCPManager
 flat_network_bridge = br100
 flat_interface = eth0
 public_interface = br100


修改/etc/sysctl.conf文件::

 vi  /etc/sysctl.conf
 
 net.ipv4.ip_forward=1


生效::

 sysctl -p


重启服务::

 service nova-compute restart
 service nova-network restart
 service nova-api-metadata restart


检查Nova服务是否已正常运行::

 nova-manage service list
 
 图案:-)表示已正常



配置网络
========

定义私有网络::

 nova network-create private --bridge br100 --multi-host T --dns1 8.8.8.8 --fixed-range-v4 10.0.0.24/29


定义浮动地址（从外部可ping通）::

 nova-manage floating create --pool=nova --ip_range=172.16.10.24/29
 nova-manage floating list


修改default访问控制策略（也可通过管理页面）::

 #允许ICMP
 nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
 
 #允许SSH访问
 nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
