---
layout: post
title: "GreenPlum安装"
date: 2018-05-14 20:24:55 +0800
comments: true
categories: GreenPlum
---
GP的部署相对来讲是比较麻烦的，主要是为了最大化的利益系统资源，需要对数据盘、系统参数、机器互信等方面进行调整，大致过程如下：
###安装前准备
####网络规划
Greenplum数据库系统常见的拓扑图如上图所示，由Master主机和Segment主机组成。Master主机和Segment主机之间会组成一个内部网络（LAN）。
为了充分发挥Greenplum数据库并行处理的性能，对网络带宽要求较高。服务器会配置多个网卡，内部网需要配置多个网段的IP。需要对外连接的服务器需配置外部IP。
建议在Greenplum数据库系统安装之前，把网络配置规划好。
![](/images/network.png)
####存储空间规划
首先，需要评估目标数据库数据所需要的空间容量。建议了解客户搭建Greenplum数据库的具体应用。举例：估计数据库所需空间为U，数据库需要启用Mirror，磁盘阵列总可用空间为D（Raid之后）。空间规划服务和如下公式：
`2 * U + U / 3 = D * 70%`
磁盘空间D平均分配到各个Segment服务器上。
Master需要相应的空间。使用服务器内置硬盘的计算方式类似。
####数据库实例规划
规划每个Segment服务器上建立的数据库实例的数量（instance数量），通常建议每2个CPU内核（core）对应一个数据库实例。
如 ：`2*4`核CPU的服务器，可配置4个实例。
<!--more-->
###操作系统规划
####修改主机名
修改各台主机的主机名称。一般建议的命名规则如下：
{% highlight ruby %}
Master: mdw
Standby Master: smdw
Segment Host: sdw1 sdw2 ...sdwn
{% endhighlight %}
修改hostname:
{% highlight bash %}
hostnamectl set-hostname mdw
{% endhighlight %}
2、修改`/etc/hosts`文件
{% highlight ruby %}
21.104.138.21   mdw-ext1
192.168.1.254   mdw-1 mdw
192.168.2.254   mdw-2
192.168.3.254   mdw-3
192.168.4.254   mdw-4
192.168.5.254   mdw-5
192.168.6.254   mdw-6
{% endhighlight %}
####关闭相关服务
{% highlight sql %}
service iptables save
service iptables stop
chkconfig iptables off
service ip6tables save
service ip6tables stop
chkconfig ip6tables off
{% endhighlight %}
####修改系统参数
编辑系统文件`/etc/sysctl.conf`,Sysctl是一个允许您改变正在运行中的Linux系统的接口。它包含一些 TCP/IP 堆栈和虚拟内存系统的高级选项
{% highlight python %}
kernel.shmmax = 500000000
kernel.shmmni = 4096
kernel.shmall = 4000000000
kernel.sem = 250 512000 100 2048
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.ipv4.conf.default.arp_filter = 1
net.core.netdev_max_backlog = 10000
vm.overcommit_memory = 2
kernel.msgmni = 2048
net.ipv4.ip_local_port_range = 1025 65535
{% endhighlight %}
编辑系统文件`/etc/security/limits.conf`
{% highlight python %}
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
{% endhighlight %}
####修改磁盘预读配置
在参数文件`/etc/rc.d/rc.local`中增加下列内容:
{% highlight ruby %}
DELL/IBM: blockdev --setra 16384 /dev/sd*
HP: blockdev --setra 16384 /dev/cciss/c?d?*
{% endhighlight %}
####修改系统引导文件
编辑`/boot/grub/menu.lst`,Deadline scheduler 用 deadline 算法保证对于既定的 IO 请求以最小的延迟时间，从这一点理解，对于 DSS 应用应该会是很适合的
{% highlight ruby %}
elevator=deadline
{% endhighlight %}
####启动IPMI服务
IPMI（Intelligent Platform Management Interface）即智能平台管理接口是使硬件管理具备“智能化”的新一代通用接口标准。如果没有安装相关服务，建议安装
{% highlight ruby %}
service ipmi start
chkconfig ipmi on
{% endhighlight %}
####修改启动配置
编辑`/etc/inittab`，修改运行级别为3，多用户命令行模式
{% highlight ruby %}
id:3:initdefault:
{% endhighlight %}
####关闭selinux并重启机器
{% highlight bash %}
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
{% endhighlight %}
如果临时关闭，使用
{% highlight bash %}
setenforce 0
{% endhighlight %}
##建立数据目录
####Master以及Standby Master主机
分区及格式化：
{% highlight go %}
mkfs.xfs /dev/sda3  # mkfs -t xfs /dev/sda3
mkdir -p /data/master # Master数据目录
{% endhighlight %}
在/etc/fstab文件中增加
{% highlight go %}
/dev/sda3 /data xfs rw,noatime,inode64,allocsize=16m 1 1
{% endhighlight %}
把/data/master 赋予777权限
{% highlight go %}
chmod -R 777 /data/master
{% endhighlight %}
####Segment主机
分区及格式化：
{% highlight go %}
mkfs.xfs  /dev/sda2  # mkfs -t xfs /dev/sda2
mkfs.xfs  /dev/sdb2
mkdir /data1  # Segment数据目录，可根据实例和分配空间不同规划不同的目录
mkdir /data2
{% endhighlight %}
在/etc/fstab文件中增加
{% highlight go %}
/dev/sda2 /data1 xfs rw,noatime,inode64,allocsize=16m 1 1
/dev/sdb2 /data2 xfs rw,noatime,inode64,allocsize=16m 1 1
{% endhighlight %}
把/data/master 赋予777权限
{% highlight go %}
chmod -R 777 /data1
chmod -R 777 /data2
{% endhighlight %}
##Master节点安装GreenPlum
####Master机器运行安装文件
{% highlight bash %}
unzip greenplum-db-4.1.1.3-build-4-RHEL5-x86_64.zip
/bin/bash greenplum-db-4.1.1.3-build-4-RHEL5-x86_64.bin
{% endhighlight %}
安装完成后修改root用户home的~/.bashrc配置文件，增加
{% highlight bash %}
source /usr/local/greenplum-db/greenplum_path.sh
{% endhighlight %}
####配置hostname文件（用于建立多机信任）
建议在安装目录下新建gpconfigs目录,包含所有master和segment主机名和别名的文件
多网卡可能如下,`hostfile_exkeys`:
{% highlight ruby %}
mdw
mdw-1
smdw
smdw-1
sdw1-1
sdw1-2
sdw2-1
sdw2-2
{% endhighlight %}
单网卡可能如下,`hostfile_exkeys`:
{% highlight ruby %}
mdw
smdw
sdw1
sdw2
{% endhighlight %}
建立`all_hosts_only`,只包含主机名，不包含各个网段对应的`hostname`，用于`gpssh`命令,all_hosts_only:
{% highlight ruby %}
mdw
smdw
sdw1
sdw2
{% endhighlight %}
##建立多机互信
root用户建立多机信任
{% highlight bash %}
source /usr/local/greenplum-db/greenplum_path.sh
gpssh-exkeys -f ./hostfile_exkeys
{% endhighlight %}
> **注意：**对于`RHEL6.x`版本，建议先关闭`OPENSSL_CONF`环境变量并设置`selinux`为`disabled`再做多机互信
如建立多机信任时出现`permission denied(publickey.gssapi-with-mic)`或者类似的错误，需要修改每台机器的`/etc/ssh/sshd_config`文件
{% highlight bash %}
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile.ssh/authorized_keys
{% endhighlight %}
默认使用的22端口，如果22端口没开建议先打开22端口
####建立用户以及用户组
{% highlight bash %}
gpssh -f ./all_hosts_only
=>groupadd -g 3030 gpadmin
=>groupadd -g 3040 gpmon
=>useradd -u 3030 -g gpadmin -m -s /bin/bash gpadmin
=>useradd -u 3040 -g gpmon -m -s /bin/bash gpmon
=>echo gpadmin | passwd  gpadmin --stdin
=>echo gpmon | passwd  gpmon --stdin
=>chown -R gpadmin:gpadmin /data
{% endhighlight %}
####修改gpadmin用户配置
使用`gpadmin`用户操作，对于Master和Standby Master主机，修改 `~/.bashrc`文件，添加如下内容：
{% highlight bash %}
source /usr/local/greenplum-db/greenplum_path.sh
MASTER_DATA_DIRECTORY=/data/master/gpseg-1
export MASTER_DATA_DIRECTORY  # gpstart默认启动的目录
{% endhighlight %}
对于Segment主机，修改 `~/.bashrc`文件，添加如下内容：
{% highlight bash %}
source /usr/local/greenplum-db/greenplum_path.sh
{% endhighlight %}
####gpadmin用户建立多机信任
使用gpadmin用户在Master主机上操作,提示密码，输入`gpadmin`
{% highlight bash %}
gpssh-exkeys -f ./hostfile_exkeys
{% endhighlight %}
##时钟同步
所有涉及到的机器之间使用NTP做时钟同步
##其他机器安装GP
####配置hostname文件
其他机器的安装主要操作时把安装在Master主机上的GP安装文件打包传到其他各台机器中。因此，需要配置一个hostname文件包含Standbymaster和各台Segment主机，配置文件stby_all_segs内容参考如下：
{% highlight ruby %}
smdw
smdw-1
sdw1-1
sdw1-2
sdw2-1
sdw2-2
{% endhighlight %}
####并行安装
安装gzip，在Master主机上，使用root用户操作：
{% highlight ruby %}
chmod 777 /usr/local
gpseginstall -f ./stby_all_segs -p gpadmin
{% endhighlight %}
##系统检查
####配置检测
{% highlight bash %}
gpcheck -f /usr/local/greenplum-db/gpconfigs/all_hosts_single -m mdw -s smdw
{% endhighlight %}
####网络性能检测
{% highlight bash %}
gpcheckperf -f /usr/local/greenplum-db/gpconfigs/all_net_1 -r N -d /tmp > checknetwork.out
gpcheckperf -f /usr/local/greenplum-db/gpconfigs/all_net_2 -r N -d /tmp > checknetwork.out
{% endhighlight %}
####磁盘性能检测
{% highlight bash %}
gpcheckperf -f /usr/local/greenplum-db/gpconfigs/all_hosts_single -r ds -D -d /data1/primary -d /data2/primary -d /data1/mirror -d /data2/mirror > checkio.out
{% endhighlight %}
##初始化数据库
####获取初始化配置
请注意，Greenplum3.x版本和4.x版本的初始化配置文件格式存在差异，配置时建议从 $GPHOME/docs/cli_help/gpconfigs/ 目录中获取样例文件，然后进行修改。
####配置样例
获取配置文件样例：
{% highlight bash %}
cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config  $GPHOME/gpconfigs/
{% endhighlight %}
修改配置文件：
{% highlight python %}
ARRAY_NAME="EMC Greenplum DW"
SEG_PREFIX=gpseg
PORT_BASE=40000
declare -a DATA_DIRECTORY=(/data1 /data1 /data1 /data1)   # 主实例
MASTER_HOSTNAME=mdw    # 主机名
MASTER_DIRECTORY=/data/master
MASTER_PORT=5432
TRUSTED SHELL=ssh
CHECK_POINT_SEGMENT=8
ENCODING=UNICODE
MIRROR_PORT_BASE=50000
REPLICATION_PORT_BASE=41000
MIRROR_REPLICATION_PORT_BASE=51000
declare -a MIRROR_DATA_DIRECTORY=(/data2 /data2 /data2 /data2)    # 备实例
MACHINE_LIST_FILE=/usr/local/greenplum-db/gpconfigs/all_segs  # segment主机列表文件
{% endhighlight %}
####整理实例列表
只列出各个网段IP的主机名称，不能添加sdw1、sdw2等
{% highlight ruby %}
sdw1-1
sdw1-2
sdw2-1
sdw2-2
sdw3-1
sdw3-2
sdw4-1
sdw4-2
{% endhighlight %}
####初始化
{% highlight ruby %}
gpinitsystem -c /usr/local/greenplum-db/gpconfigs/gpinitsystem_config -s smdw
{% endhighlight %}
####修改访问权限
修改Master数据目录（MASTER_DATA_DIRECTORY）下`pg_hba.conf`文件。需要了解客户实际情况，有多少客户端的IP地址以及角色需要访问数据库。举例如下：
{% highlight ruby %}
host     all         gpadmin         10.32.38.0/16          trust
{% endhighlight %}
IP范围格式：IP 地址/CIDR，如：`10.32.38.0/16`；`255.0.0.0`表示 IPv4 CIDR 掩码长度 8，`255.255.255.0`表示 IPv4 CIDR 掩码长度 24，而 `255.255.255.255` 表示 CIDR 掩码长度 32；32就表示指定IP，24就表示小子网。
修改完后数据库重载参数文件：
{% highlight ruby %}
gpstop -u
{% endhighlight %}