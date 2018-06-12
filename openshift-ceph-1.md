## openshift实践-持久化存储

官网文档：https://docs.openshift.com/enterprise/3.1/install_config/persistent_storage/persistent_storage_ceph_rbd.html

### ceph 集群部署

ceph上当前非常流行的开源分布式存储解决方案. Ceph是一个可靠地、自动重均衡、自动恢复的分布式存储系统，根据场景划分可以将Ceph分为三大块，分别是对象存储、块设备存储和文件系统服务。
Ceph 独一无二地用统一的系统提供了对象、块、和文件存储功能，它可靠性高、管理简便、并且是自由软件。 Ceph 的强大足以改变贵公司的 IT 基础架构、和管理海量数据。 Ceph 可提供极大的伸缩性——供成千用户访问 PB 乃至 EB 级的数据。 Ceph 节点以普通硬件和智能守护进程作为支撑点， Ceph 存储集群组织起了大量节点，它们之间靠相互通讯来复制数据、并动态地重分布数据.

#### 环境准备
系统：ubuntu 16.04

172.16.130.11 node1

172.16.130.12 node2

172.16.130.13 node3

节点角色：

```
admin-node, deploy-node(ceph-deploy)：172.16.130.11 node1
mon.node1，(mds.node1): 172.16.130.11  node1
osd.0: 172.16.130.12 node2
osd.1: 172.16.130.13 node3
```

Ceph分布式存储集群由若干组件组成，包括：Ceph Monitor、Ceph OSD和Ceph MDS，其中如果你仅使用对象存储和块存储时，MDS不是必须的（本次我们也不需要安装MDS），仅当你要用到Cephfs时，MDS才是需要安装的。

Ceph的安装模型是通过一个deploy node远程操作其他Node以create、prepare和activate各个Node上的Ceph组件，官方手册中给出的示意图如下

![image](openshift-imaeges/ceph-1.png)


部署之前需要解决几个问题

1. ntp 部署

2. hostname设置

3. ssh 免密登录

Ceph提供了一键式安装工具ceph-deploy来协助Ceph集群的安装，在deploy node上，我们首先要来安装的就是ceph-deploy
默认的源比较老, 我们需要添加Ceph源，安装最新的ceph-deploy：

```
# wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
OK

echo deb https://download.ceph.com/debian-jewel/ $(lsb_release -sc) main | sudo tee 

apt-get update && apt-get install ceph-deploy


```

注意： 如果我们使用普通用户，在ceph-deploy真正执行安装之前，需要确保所有Ceph node都要开启NTP，同时建议在每个node节点上为安装过程创建一个安装账号，即ceph-deploy在ssh登录到每个Node时所用的账号。这个账号有两个约束
	
	具有sudo权限；
	执行sudo命令时，无需输入密码
```
以下命令在每个Node上都要执行：

useradd -d /home/cephd -m cephd
passwd cephd

添加sudo权限：
echo "cephd ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephd
sudo chmod 0440 /etc/sudoers.d/cephd
```

#### 部署ceph


##### ceph install node

如果之前安装过ceph，可以先执行如下命令以获得一个干净的环境：

```
ceph-deploy purgedata node1 node2 node3
ceph-deploy forgetkeys
ceph-deploy purge node1 node2 node3
```

接下来我们就可以来全新安装Ceph了。在deploy node上，建立ceph install目录，然后进入ceph install目录执行相关步骤。

我们首先来创建一个ceph cluster，这个环节需要通过执行ceph-deploy new {initial-monitor-node(s)}命令。按照上面的安装设计，我们的ceph monitor node就是node1，因此我们执行下面命令来创建一个名为ceph的ceph cluster：

```
$ mkdir my-cluster && cd my-cluster
$ ceph-deploy new node1
```

new命令执行完后，ceph-deploy会在当前目录下创建一些辅助文件:

```
root@node1:~/my-cluster# ls -alh
total 112K
drwxr-xr-x  3 root root 4.0K Jun 12 16:10 .
drwx------ 30 root root 4.0K Jun 12 15:18 ..
-rw-------  1 root root   71 Jun 12 15:19 ceph.bootstrap-mds.keyring
-rw-------  1 root root   71 Jun 12 15:19 ceph.bootstrap-osd.keyring
-rw-------  1 root root   71 Jun 12 15:19 ceph.bootstrap-rgw.keyring
-rw-------  1 root root   63 Jun 12 15:19 ceph.client.admin.keyring
-rw-r--r--  1 root root  253 Jun 12 15:18 ceph.conf
-rw-r--r--  1 root root  72K Jun 12 15:30 ceph-deploy-ceph.log
-rw-------  1 root root   73 Jun 12 15:15 ceph.mon.keyring


### 这是我修改之后的ceph.conf

root@node1:~/my-cluster# cat ceph.conf
[global]
fsid = b20b5e7b-9518-4500-acc9-de2ee642f47a
mon_initial_members = node1
mon_host = 172.16.130.11
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx


## 这些是我们手动加上去，因为ceph推荐使用xfs文件系统，但是我们使用ext4文件系统
osd max object name len = 256 
osd max object namespace len = 64

## 默认pool 是 3
osd pool default size = 2

```

由于我们仅有两个OSD节点，因此我们在进一步安装之前，需要先对ceph.conf文件做一些配置调整：
修改配置以进行后续安装：

```
在[global]标签下，添加下面一行：
osd pool default size = 2
```
接下来，我们执行下面命令在node1和node2上安装ceph运行所需的各个binary包：

```
ceph-deploy install nod1 node2 node3
```

这一过程ceph-deploy会SSH登录到各个node上去，执行apt-get update, 并install ceph的各种组件包，这个环节耗时可能会长一些（依网络情况不同而不同），请耐心等待。


##### 初始化ceph monitor node

有了ceph启动的各个程序后，我们首先来初始化ceph cluster的monitor node。在deploy node的工作目录cephinstall下，执行：

```
# ceph-deploy mon create-initial

[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.35): /usr/bin/ceph-deploy mon create-initial
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create-initial
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f0f7ea2fe60>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mon at 0x7f0f7ee93de8>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  keyrings                      : None
[ceph_deploy.mon][DEBUG ] Deploying mon, cluster ceph hosts node1
[ceph_deploy.mon][DEBUG ] detecting platform for host node1...
[node1][DEBUG ] connected to host: node1
[node1][DEBUG ] detect platform information from remote host
[node1][DEBUG ] detect machine type
```

这一过程很顺利。命令执行完成后我们能看到一些变化：

在当前目录下，出现了若干*.keyring，这是Ceph组件间进行安全访问时所需要的,在node1(monitor node)上，我们看到ceph-mon已经运行起来了：

```
root@node1:~/my-cluster$ ps -ef|grep ceph
ceph     32326     1  0 14:19 ?        00:00:00 /usr/bin/ceph-mon --cluster=ceph -i node1 -f --setuser ceph --setgroup ceph
```

##### prepare ceph OSD node

至此，ceph-mon组件程序已经成功启动了，剩下的只有OSD这一关了。启动OSD node分为两步：prepare 和 activate。OSD node是真正存储数据的节点，我们需要为ceph-osd提供独立存储空间，一般是一个独立的disk。但我们环境不具备这个条件，于是在本地盘上创建了个目录，提供给OSD。

```
在deploy node上执行：

ssh node2
sudo mkdir /var/local/osd0
exit

ssh node3
sudo mkdir /var/local/osd1
exit
```

接下来，我们就可以执行prepare操作了，prepare操作会在上述的两个osd0和osd1目录下创建一些后续activate激活以及osd运行时所需要的文件：

```
ceph-deploy osd prepare node2:/var/local/osd0 node3:/var/local/osd1
```
这时候osd 进程不会去启动

##### 激活ceph OSD node

接下来，来激活各个OSD node

```
ceph-deploy osd activate node2:/var/local/osd0 node3:/var/local/osd1
```


执行ceph admin:


```
ceph-deploy admin node1 node2 node3
```

接下来，查看一下ceph集群中的OSD节点状态：

```
root@node1:~/my-cluster# ceph osd tree
ID WEIGHT  TYPE NAME      UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1 0.36499 root default
-2 0.18250     host node2
 0 0.18250         osd.0       up  1.00000          1.00000
-3 0.18250     host node3
 1 0.18250         osd.1       up  1.00000          1.00000


root@node1:~/my-cluster# ceph -s
    cluster b20b5e7b-9518-4500-acc9-de2ee642f47a
     health HEALTH_OK
     monmap e1: 1 mons at {node1=172.16.130.11:6789/0}
            election epoch 3, quorum 0 node1
     osdmap e11: 2 osds: 2 up, 2 in
            flags sortbitwise,require_jewel_osds
      pgmap v2213: 64 pgs, 1 pools, 37572 kB data, 18 objects
            37414 MB used, 318 GB / 373 GB avail
                  64 active+clean
root@node1:~/my-cluster# ceph health
HEALTH_OK
root@node1:~/my-cluster#
```

##### 遇到的问题


1. 权限问题

```
node1][WARNIN] 2018-6-12 14:25:40.325075 7fd1aa73f800 -1  ** ERROR: error creating empty object store in /var/local/osd0: (13) Permission denied
[node1][WARNIN]
[node1][ERROR ] RuntimeError: command returned non-zero exit status: 1
[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: /usr/sbin/ceph-disk -v activate --mark-init upstart --mount /var/local/osd0
```
osd0被root拥有，ceph-deploy是以ceph用户启动的ceph-osd程序自然没有权限在/var/local/osd0目录下创建文件并写入数据了。这个问题在ceph官方issue中有很多人提出来，也给出了临时修正方法：

```
将osd0和osd1的权限赋予ceph:ceph：

node2：
sudo chown -R ceph:ceph /var/local/osd0

node3：
sudo chown -R ceph:ceph /var/local/osd1

```

2. ceph使用ext4文件系统的问题

错误日志

```
2018-6-12 15:13:47.138895 7f568d6db800  1 journal close /var/lib/ceph/osd/ceph-0/journal
2018-6-12 15:13:47.140041 7f568d6db800 -1  ** ERROR: osd init failed: (36) File name too long
```

官方不建议采用ext4文件系统作为ceph的后端文件系统，如果采用，那么对于ext4的filesystem，应该在ceph.conf中添加如下配置：

```
osd max object name len = 256
osd max object namespace len = 64
```

####  k8s（openshift）使用rbd

##### rdb 用作 volume 挂载

##### rbd 用作 PV/PVC

##### rdb 用作 storage class

##### 使用StatefulSet解决多副本的问题

