---
layout:     post
title:      glusterfs安装配置
subtitle:   glusterfs安装配置
date:       2018-10-30
author:     ishine
header-img: 
catalog: true
tags:
    - glusterfs
---

## 安装 GlusterFS

### 环境说明
使用四台服务器安装 GlusterFS 组成一个集群。

| IP地址 | 主机名 | OS |
| --- | --- | --- |
| 10.100.4.181 | glusterfs-master | CentOS7.4.1708 |
| 10.100.4.182 | glusterfs-node1 | CentOS7.4.1708 |
| 10.100.4.183 | glusterfs-node2 | CentOS7.4.1708 |
| 10.100.4.184 | glusterfs-node3 | CentOS7.4.1708 |

### 配置 hosts

```
10.100.4.181 glusterfs-master
10.100.4.182 glusterfs-node1
10.100.4.183 glusterfs-node2
10.100.4.184 glusterfs-node3
```

### 安装 GlusterFS

CentOS 操作系统安装 GlusterFS 非常简单，官方默认提供了 RPM 包。在4台节点上都安装 glusterfs。

```bash
yum -y install centos-release-gluster
yum -y install glusterfs glusterfs-server glusterfs-fuse glusterfs-rdma
```

**启动 GlusterFS 集群**

在 4台节点上都启动 glusterfs

```bash
systemctl start glusterd.service
systemctl enable glusterd.service
systemctl status glusterd.service
```

**给集群增加节点**

现在4台机器都已经 Yum 安装完毕，并且启动了服务，在任何一台机器上面运行命令都是可以的，因为没有中心节点，每个节点都是平等。

```bash
[root@glusterfs-master ~]# gluster peer probe glusterfs-master
peer probe: success. Probe on localhost not needed
[root@glusterfs-master ~]# gluster peer probe glusterfs-node1
peer probe: success. 
[root@glusterfs-master ~]# gluster peer probe glusterfs-node2
peer probe: success. 
[root@glusterfs-master ~]# gluster peer probe glusterfs-node3
peer probe: success. 
```

**查看集群状态**

```bash
[root@glusterfs-master ~]# gluster peer status
Number of Peers: 3

Hostname: glusterfs-node1
Uuid: 7254e2bf-606d-4027-b17f-a039dfca1168
State: Peer in Cluster (Connected)

Hostname: glusterfs-node2
Uuid: ddbd3ae3-c0f8-47c7-a0a3-9d3f23210bc8
State: Peer in Cluster (Connected)

Hostname: glusterfs-node3
Uuid: aa079f15-af1b-4ab1-86f9-8c98ba9bd463
State: Peer in Cluster (Connected)
```
**在4台服务器上创建数据存储目录**

```bash
mkdir -pv /data/gluster/data
```

生产中 /data 目录应该是独立的磁盘分区，我这里为了演示/data目录是在根分区下。

**查看 Volume 状态**

在 gluster-master上进行查看，因为没有创建任何卷，所以结果是没有卷。

```bash
[root@glusterfs-master ~]# gluster volume info
No volumes present
```

**创建一个 replica 类型卷测试**

replica 副本数为 4，逻辑卷名字叫 myvolume

```bash
[root@glusterfs-master ~]# gluster volume create myvolume replica 4 glusterfs-master:/data/gluster/data glusterfs-node1:/data/gluster/data glusterfs-node2:/data/gluster/data glusterfs-node3:/data/gluster/data
volume create: myvolume: success: please start the volume to access data
```
**注意：如果你跟我一样 /data 目录不是独立的磁盘分区，在执行上面命令时会提示如下信息，解决方法就是在命令最后面加上 force 即可。**

```
volume create: myvolume: failed: The brick glusterfs-master:/data/gluster/data is being created in the root partition. It is recommended that you don't use the system's root partition for storage backend. Or use 'force' at the end of the command if you want to override this behavior.
```

**在查看 Volume 状态**

```bash
[root@glusterfs-master ~]# gluster volume info
 
Volume Name: myvolume
Type: Replicate
Volume ID: ae8df54e-1dd1-4cf2-abf6-a418bdaa7c7f
Status: Created
Snapshot Count: 0
Number of Bricks: 1 x 4 = 4
Transport-type: tcp
Bricks:
Brick1: glusterfs-master:/data/gluster/data
Brick2: glusterfs-node1:/data/gluster/data
Brick3: glusterfs-node2:/data/gluster/data
Brick4: glusterfs-node3:/data/gluster/data
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

**启动 myvolume 卷**

创建完卷以后是不可以被挂载使用的，要先启动卷。

```bash
[root@glusterfs-master ~]# gluster volume start myvolume
volume start: myvolume: success
```

### 部署 GlusterFS 客户端

部署客户端，并 mount 挂载 GlusterFS 文件系统，客户端必须在本地加入 glusterfs hosts 否则报错。

**安装**

```bash
yum -y install glusterfs glusterfs-fuse
```
**添加 glusterfs hosts**

```bash
10.100.4.181 glusterfs-master
10.100.4.182 glusterfs-node1
10.100.4.183 glusterfs-node2
10.100.4.184 glusterfs-node3
```

**创建一个挂载目录并挂载**

```bash
root@k8s-node1:~ # mkdir /opt/gfsmnt
root@k8s-node1:~ # mount -t glusterfs glusterfs-master:myvolume /opt/gfsmnt/
root@k8s-node1:~ # mount|grep glusterfs
glusterfs-master:myvolume on /opt/gfsmnt type fuse.glusterfs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072)
```

**测试**

在客户端创建一个 1G 的文件

```bash
root@k8s-node1:/opt/gfsmnt # time dd if=/dev/zero of=test_file bs=1000M count=1
记录了1+0 的读入
记录了1+0 的写出
1048576000字节(1.0 GB)已复制，36.4209 秒，28.8 MB/秒

real    0m36.438s
user    0m0.000s
sys     0m1.857s
```

## 创建各种类型的卷

### 创建分布式卷（Distributed GlusterFS Volume）

distribute volume，即 DHT，也叫分布卷。默认模式

```bash
# gluster volume create <NEW-VOLNAME> [transport tcp | rdma | tcp,rdma]
# 使用tcp创建具有四个存储服务器的分布式卷
gluster volume create test-volume glusterfs-master:/data/gluster/data glusterfs-node1:/data/gluster/data glusterfs-node2:/data/gluster/data glusterfs-node3:/data/gluster/data force 
```

![](http://p1ly7m2xj.bkt.clouddn.com/15238740186694.jpg)

### 创建复制卷（Replica GlusterFS Volume）

```bash
# gluster volume create <NEW-VOLNAME> [replica] [transport tcp | rdma | tcp,rdma]
# 创建具有2个存储服务器的复制卷
gluster volume create test-volume replica 2 glusterfs-master:/data/gluster/data glusterfs-node1:/data/gluster/data force
```

![](http://p1ly7m2xj.bkt.clouddn.com/15238739479862.jpg)

### 创建条带卷（Striped GlusterFS Volume）

```bash
# gluster volume create <NEW-VOLNAME> [stripe] transport tcp | rdma | tcp,rdma]
gluster volume create test-volume stripe 2 transport tcp glusterfs-node2:/data/gluster/data glusterfs-node3:/data/gluster/data force
```

![](http://p1ly7m2xj.bkt.clouddn.com/15238744131831.jpg)

### 创建分布式条带卷（Distributed Striped GlusterFS Volume）

```bash
# gluster volume create  [stripe ] [transport tcp | rdma | tcp,rdma]
# 要跨8个存储服务器创建分布式条带卷
gluster volume create test-volume stripe 4 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4 server5:/exp5 server6:/exp6 server7:/exp7 server8:/exp8
```

### 创建分布式复制卷（Distributed Replica GlusterFS Volume）

```bash
# gluster volume create  [replica ] [transport tcp | rdma | tcp,rdma]
# 具有双向镜像的四个节点分布（复制）卷
gluster volume create test-volume replica 2 transport tcp glusterfs-master:/data/gluster/data glusterfs-node1:/data/gluster/data glusterfs-node2:/data/gluster/data glusterfs-node3:/data/gluster/data force
```

![](http://p1ly7m2xj.bkt.clouddn.com/15238751024944.jpg)

### 创建分布式条带复制卷 （Distributed Striped Replicated Volumes）

```bash
# gluster volume create  [stripe ] [replica ] [transport tcp | rdma | tcp,rdma]

# 例如，要跨八个存储服务器创建分布式复制条带卷
gluster volume create test-volume stripe 2 replica 2 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4 server5:/exp5 server6:/exp6 server7:/exp7 server8:/exp8
```

### 创建条带复制卷（Striped Replicated Volumes）

```bash
# gluster volume create  [stripe ] [replica ] [transport tcp | rdma | tcp,rdma]
# 例如，要在四个存储服务器上创建分条复制卷，请执行以下操作
gluster volume create test-volume stripe 2 replica 2 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4
```

![](http://p1ly7m2xj.bkt.clouddn.com/15238762899870.jpg)

## 其它维护类命令

### 1. 查看 GlusterFS 中所有的 Volume

```bash
gluster volume list
```

### 2. 删除 GlusterFS 磁盘

```bash
# 停止名为 test-volume 的磁盘
[root@glusterfs-master ~]# gluster volume stop test-volume
Stopping volume will make its data inaccessible. Do you want to continue? (y/n) y
volume stop: test-volume: success
# 删除名字为 test-volume 的磁盘
[root@glusterfs-master ~]# gluster volume delete test-volume
Deleting volume will erase all information about the volume. Do you want to continue? (y/n) y
volume delete: test-volume: success
```

### 3. 卸载某个节点的 GlusterFS 磁盘

```bash
# 从存储池中删除 glusterfs-node3 节点
gluster peer detach glusterfs-node3
```

![](http://p1ly7m2xj.bkt.clouddn.com/15238873389758.jpg)

### 4. 设置访问限制,按照每个 volume 来限制

```bash
# 限制只允许10.100.4.198 主机访问访问 test-volume 卷
[root@glusterfs-master ~]# gluster volume set test-volume auth.allow 10.100.4.198
volume set: success

# 限制只允许 10.100.4.*,10.101.2.* 访问 test-volume 卷
gluster volume set test-volume auth.allow 10.100.4.*,10.101.2.*
```

![](http://p1ly7m2xj.bkt.clouddn.com/15238879214079.jpg)

### 5. 添加 GlusterFS 节点

```bash
# 
[root@glusterfs-master ~]# gluster peer probe glusterfs-node2
peer probe: success. 
[root@glusterfs-master ~]# gluster peer probe glusterfs-node3
peer probe: success. 
[root@glusterfs-master ~]# gluster volume add-brick test-volume glusterfs-node2:/data/gluster/data glusterfs-node3:/data/gluster/data force
volume add-brick: success
```
注意：如果是复制卷或者条带卷，则每次添加的Brick数必须是replica或者stripe的整数倍

### 6. 配置卷

```bash
gluster volume set mamm-volume key value 
```

### 7. 缩容 Volume

```
[root@glusterfs-master ~]# gluster volume info
 
Volume Name: myvolume
Type: Distribute
Volume ID: 8023edea-ceeb-4f58-a59f-99ca603d0810
Status: Started
Snapshot Count: 0
Number of Bricks: 3
Transport-type: tcp
Bricks:
Brick1: glusterfs-master:/data/gluster/data
Brick2: glusterfs-node1:/data/gluster/data
Brick3: glusterfs-node2:/data/gluster/data
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
```
先将数据迁移到其它可用的Brick，迁移结束后才将该Brick移除

```bash
[root@glusterfs-master ~]# gluster volume remove-brick myvolume glusterfs-node2:/data/gluster/data glusterfs-node1:/data/gluster/data/ start
volume remove-brick start: success
ID: 7eb1cf7b-489a-42b3-9c86-a8721aa4fd45
```

在执行了start之后，可以使用status命令查看移除进度

```bash
[root@glusterfs-master ~]# gluster volume remove-brick myvolume glusterfs-node2:/data/gluster/data glusterfs-node1:/data/gluster/data/ status
```
不进行数据迁移，直接删除该Brick

```bash
[root@glusterfs-master ~]# gluster volume remove-brick myvolume glusterfs-node2:/data/gluster/data glusterfs-node1:/data/gluster/data/ commit
```
注意：如果是复制卷或者条带卷，则每次移除的Brick数必须是replica或者stripe的整数倍。

### 8. 扩容 volume

```bash
gluster volume add-brick test-volume glusterfs-node3:/data/gluster/data
```
### 9. 迁移 volume

```bash
使用start命令开始进行迁移：

# gluster volume replace-brick <VOLNAME> <BRICK> <NEW-BRICK> start

在数据迁移过程中，可以使用pause命令暂停迁移：

# gluster volume replace-brick <VOLNAME> <BRICK> <NEW-BRICK> pause

在数据迁移过程中，可以使用abort命令终止迁移：

# gluster volume replace-brick <VOLNAME> <BRICK> <NEW-BRICK> abort

在数据迁移过程中，可以使用status命令查看迁移进度：

# gluster volume replace-brick <VOLNAME> <BRICK> <NEW-BRICK> status

在数据迁移结束后，执行commit命令来进行Brick替换：

# gluster volume replace-brick <VOLNAME> <BRICK> <NEW-BRICK> commit
```
## 参考资料

https://www.cnblogs.com/jicki/p/5801712.html

https://www.ibm.com/developerworks/cn/opensource/os-cn-glusterfs-docker-volume/index.html

http://blog.51niux.com/?id=154

https://blog.csdn.net/liuaigui/article/details/17331557