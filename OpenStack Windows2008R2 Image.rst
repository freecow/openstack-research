==============================
OpenStack安装Windows2008R2镜像
==============================

上传windows2008的安装iso文件至compute节点
----------------------------------------

方法一：通过FTP方式
~~~~~~~~~~~~~~~~~~~

安装vsftpd服务器::

 apt-get install -y vsftpd
 vi /etc/vsftpd.conf
 
 #允许ftp目录可写
 write_enable = YES
 service vsftpd restart

用ftp客户端上传iso文件至compute节点ftp登录用户的home目录


方法二：通过iSCSI连接VMFS存储卷
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

适用于iso文件存于已格式化为VMFS卷的存储上

连接iSCSI存储，挂载至/nas目录（参见Ubuntu iSCSI Config）

拷贝iso文件至home目录::

 cp /nas/cn_windows...iso win2008r2.iso



创建镜像文件
------------

上传virtio驱动iso文件至compute节点::

 rz
 选择virtio-win-drivers-20120712-1.iso文件


创建空白镜像文件::

 qemu-img create -f qcow2 win2008r2.img 20G


启动镜像安装::

 qemu-system-x86_64 --enable-kvm -m 2048 -boot d -drive file=win2008r2.img, if=virtio -cdrom win2008r2.iso -drive file=virtio-win-drivers-20120712-1.iso, media=cdrom -net nic, model=virtio -net user -nographic -vnc :1


vnc客户端后续安装
-----------------

1. 通过vnc客户端连接172.16.10.52:5901
2. 安装选择自定义
3. 选择加载启动程序→浏览→选择virtio光盘→/STORAGE/SERVER_2008_R2/AMD64，找到驱动后点击下一步，扫描出未分配的硬盘空间
4. 安装完毕设置Administrator口令
5. 进入Windows后，初始配置任务窗口中，选择第三项的启用远程桌面，选择第二项登录方式（可选择更安全的方式）
6. 选择配置Windows防火墙，关闭防火墙（不推荐）
7. 在初始配置任务窗口，勾选登录时不显示此窗口
8. 弹出服务器管理窗口，勾选下次不显示此窗口，左侧选择诊断→设备管理器，选择未驱动的网卡，右键选择更新驱动程序
9. 选择virtio光盘中的/NETWORK/SERVER_2008_R2/AMD64目录
10. 以管理员身份进入命令提示符窗口，进入c:\windows\system32\sysprep目录，运行sysprep程序，勾选通用，选择关机
11. 至此，Windows 2008的镜像制作完毕

**注：** 如vnc客户端连接闪退，则需修改vnc客户端options→expert→colorlevel，改为rgb222


上传Glance
-----------

将镜像文件回传至controller::

 scp win2008r2.img loginname@controller:~
 #loginname采用你登陆的用户名，~ 表示回传到/home/zhanghui中


glance上传镜像::

 source creds
 glance image-create --name="Windows 2008 R2" --is-public=true --disk-format=qcow2 --container-format=ovf --file win2008r2.img


实例创建测试
------------

1. 登录dashboard，创建相应镜像的实例
2. 到实例管理界面，分配浮动地址给实例
3. 切换到实例的novnc画面，由于系统进行过sysprep的封装，需要进行几个初始化配置
4. 修改default访问规则，添加TCP 3389
5. 配置完毕后，可通过远程桌面可以访问该浮动地址
