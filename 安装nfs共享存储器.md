### 安装nfs共享存储器

NFS服务器IP: 192.168.11.31

客户端IP: 192.168.11.34

目标： 在nfs服务器上共享一个目录，在客户端上可以直接操作NFS服务器上的这个共享目录下的文件

**NFS服务器配置** 

**1.安装NFS服务** 

首先使用yum安装nfs服务：

```javascript
yum -y install rpcbind nfs-utils
```

复制

**2.创建共享目录** 

在服务器上创建共享目录，并设置权限。

```javascript
mkdir /data/share/
chmod 755 -R /data/share/
```

复制

**3.配置NFS** 

nfs的配置文件是 /etc/exports ，在配置文件中加入一行：

```javascript
/data/share/ 192.168.11.34(rw,no_root_squash,no_all_squash,sync)
```

这行代码的意思是把共享目录/data/share/共享给192.168.11.34这个客户端ip，后面括号里的内容是权限参数，其中：

rw 表示设置目录可读写。

sync 表示数据会同步写入到内存和硬盘中，相反 rsync 表示数据会先暂存于内存中，而非直接写入到硬盘中。

no_root_squash NFS客户端连接服务端时如果使用的是root的话，那么对服务端分享的目录来说，也拥有root权限。

no_all_squash 不论NFS客户端连接服务端时使用什么用户，对服务端分享的目录来说都不会拥有匿名用户权限。

如果有多个共享目录配置，则使用多行，一行一个配置。保存好配置文件后，需要执行以下命令使配置立即生效：

```javascript
exportfs -r
```

**4.设置防火墙** 

如果你的系统没有开启防火墙，那么该步骤可以省略。

NFS的防火墙特别难搞，因为除了固定的port111、2049外，还有其他服务如rpc.mounted等开启的不固定的端口，这样对防火墙来说就比较麻烦了。为了解决这个问题，我们可以设置NFS服务的端口配置文件。

修改/etc/sysconfig/nfs文件，将下列内容的注释去掉，如果没有则添加：

```javascript
RQUOTAD_PORT=1001
LOCKD_TCPPORT=30001
LOCKD_UDPPORT=30002
MOUNTD_PORT=1002
```

保存好后，将端口加入到防火墙允许策略中。执行：

```javascript
firewall-cmd --zone=public --add-port=111/tcp --add-port=111/udp --add-port=2049/tcp --add-port=2049/udp --add-port=1001/tcp --add-port=1001/udp --add-port=1002/tcp --add-port=1002/udp --add-port=30001/tcp --add-port=30002/udp --permanent
firewall-cmd --reload
```

**5.启动服务** 

按顺序启动rpcbind和nfs服务：

```javascript
systemctl start rpcbind
systemctl start nfs
```

加入开机启动：

```javascript
systemctl enable rpcbind 
systemctl enable nfs
```

nfs服务启动后，可以使用命令 rpcinfo -p 查看端口是否生效。

服务器的后，我们可以使用 showmount 命令来查看服务端(本机)是否可连接：

```javascript
[root@localhost ~]# showmount -e localhost
Export list for localhost:
/data/share 192.168.11.34
```

出现上面结果表明NFS服务端配置正常。



**客户端配置** 

**1.安装rpcbind服务** 

客户端只需要安装rpcbind服务即可，无需安装nfs或开启nfs服务。

```javascript
yum -y install rpcbind
```

**2.挂载远程nfs文件系统** 

查看服务端已共享的目录:

```javascript
[root@localhost ~]# showmount -e 192.168.11.31
Export list for 192.168.11.31:
/data/share 192.168.11.34
```

建立挂载目录，执行挂载命令：

```javascript
mkdir -p /mnt/share
mount -t nfs 192.168.11.31:/data/share /mnt/share/ -o nolock,nfsvers=3,vers=3
```

如果不加 -onolock,nfsvers=3 则在挂载目录下的文件属主和组都是nobody，如果指定nfsvers=3则显示root。

如果要解除挂载，可执行命令：

```javascript
umount /mnt/share
```

**3.开机自动挂载** 

如果按本文上面的部分配置好，NFS即部署好了，但是如果你重启客户端系统，发现不能随机器一起挂载，需要再次手动操作挂载，这样操作比较麻烦，因此我们需要设置开机自动挂载。我们不要把挂载项写到/etc/fstab文件中，因为开机时先挂载本机磁盘再启动网络，而NFS是需要网络启动后才能挂载的，所以我们把挂载命令写入到/etc/rc.d/rc.local文件中即可。

```javascript
[root@localhost ~]# vim /etc/rc.d/rc.local
#在文件最后添加一行：
mount -t nfs 192.168.11.31:/data/share /mnt/share/ -o nolock,nfsvers=3,vers=3
```

保存并重启机器看看。

**测试验证** 

查看挂载结果，在客户端输入 df -h

```javascript
文件系统    容量 已用 可用 已用% 挂载点
/dev/mapper/centos-root   18G 5.0G 13G 29% /
devtmpfs      904M  0 904M 0% /dev
tmpfs       916M  0 916M 0% /dev/shm
tmpfs       916M 9.3M 906M 2% /run
tmpfs       916M  0 916M 0% /sys/fs/cgroup
/dev/sda1      497M 164M 334M 33% /boot
tmpfs       184M  0 184M 0% /run/user/0
192.168.11.31:/data/share  18G 1.7G 16G 10% /mnt/share
```

看到最后一行了没，说明已经挂载成功了。接下来就可以在客户端上进入目录/mnt/share下，新建/删除文件，然后在服务端的目录/data/share查看是不是有效果了，同样反过来在服务端操作在客户端对应的目录下看效果。