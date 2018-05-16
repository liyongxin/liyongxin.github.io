---
layout: post
title: "GreenPlum脚本化安装"
date: 2018-05-15 20:02:51 +0800
comments: true
categories: GreenPlum
---
由于手动安装`Greenplum`过于繁琐，考虑对安装过程进行自动化，针对手动安装的过程，大致整理了几个脚本，简化安装程序，安装过程中必要的交互操作可以通过`expect`来实现，比如多机互信环节、`master`安装`GreenPlum`过程等。
大致过程如下：
##准备脚本
`root`用户登录`master`节点，创建`/opt/gp`目录
{% highlight bash %}
mkdir /opt/gp
{% endhighlight %}
拷贝`gp.tar`到`/opt/gp`目录，解压
{% highlight bash %}
tar -xvf gp.tar -C /opt/gp
chmod -R 777 /opt/gp
{% endhighlight %}
##master安装GP
{% highlight bash %}
cd /opt/gp
./greenplum-db-4.3.0.0-build-3-RHEL5-x86_64.bin
{% endhighlight %}
##关闭相关服务，执行脚本拷贝
####配置hosts文件,内容参照如下
{% highlight ruby %}
10.68.28.222 mdw
10.68.28.223 smdw
10.68.28.224 sdw1
10.68.28.225 sdw2
{% endhighlight %}
<!--more-->
####执行脚本pre_c.sh
需输入机器密码，进行各节点修改hosts、建目录、关防火墙、selinux等操作
{% highlight bash %}
#!/bin/sh
cat ./all_hosts | while read line
do
   ip=${line% *}
   hostname=${line#* }
   hosts=`cat ./all_hosts`
   sshfile=/etc/ssh/sshd_config
   ssh root@$ip <<-!!!
echo "$hosts"|sed -i '/^M//g' >> /etc/hosts
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0
service iptables save
service iptables stop
chkconfig iptables off
service ip6tables save
service ip6tables stop
chkconfig ip6tables off
#/etc/ssh/sshd_config
sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' $sshfile
sed -i 's/^#RSAAuthentication.*/RSAAuthentication yes/' $sshfile
sed -i 's/^#PubkeyAuthentication.*/PubkeyAuthentication yes/' $sshfile
sed -i 's/^#AuthorizedKeysFile.*/AuthorizedKeysFile .ssh\/authorized_keys/' $sshfile
if [ ! -d /opt/gp ];then
   mkdir -p /opt/gp
   chmod 777 /opt/gp
fi
exit
!!!
done
{% endhighlight %}
##为root用户建立互信并添加必要的用户和组
####配置all_hosts文件，内容参照如下：
{% highlight bash %}
mdw
smdw
sdw1
sdw2
{% endhighlight %}
####执行rtrust_adduser.sh,需输入一次机器密码
{% highlight bash %}
#!/bin/sh
source /usr/local/greenplum-db/greenplum_path.sh
chmod 777 /opt/gp/all_hosts
gpssh-exkeys -f /opt/gp/all_hosts
#./addUser.sh
gpssh -f ./all_hosts 'groupadd -g 3030 gpadmin;groupadd -g 3040 gpmon;useradd -u 3030 -g gpadmin -m -s /bin/bash gpadmin;useradd -u 3040 -g gpmon -m -s /bin/bash gpmon;echo gpadmin | passwd gpadmin --stdin;echo gpmon | passwd gpmon --stdin;chown -R gpadmin:gpadmin /data;chown -R gpadmin:gpadmin /gpdata1;chown -R gpadmin:gpadmin /gpdata2;chown -R gpadmin:gpadmin /gpdata3;chown -R gpadmin:gpadmin /gpdata4;chown -R gpadmin:gpadmin /gpdata5;chown -R gpadmin:gpadmin /gpdata6;chown -R gpadmin:gpadmin /gpdata7;chown -R gpadmin:gpadmin /gpdata8;'
{% endhighlight %}
##各节点安装必要的软件包，拷贝相关的脚本
####配置所有segment节点配置文件all_segs,参照如下
{% highlight bash %}
sdw1
sdw2
{% endhighlight %}
####配置standby节点配置文件standby，若不安装smdw，无需配置，参照如下
{% highlight bash %}
smdw
{% endhighlight %}
####执行cprpm.sh，若环境没法连接外网，需手动搭建本地源服务器，并传递参数本地源服务器ip地址，若可连外网，无需传参
{% highlight bash %}
#!/bin/bash
localyum_ip=$1
echo $localyum_ip
if [[ $localyum_ip = "" ]];then
    echo "Did not set up local yum repository,rpms will be downloaded from Internet"
    yum -y install gcc openssh-clients unzip zip OpenIPMI ipmitool xfsprogs kmod-xfs ntp
    source /usr/local/greenplum-db/greenplum_path.sh
    gpssh -f ./all_hosts yum -y install gcc openssh-clients unzip zip OpenIPMI ipmitool xfsprogs kmod-xfs ntp
    # exit 0
fi

if [ ! -f "./all_hosts" ]; then
    echo "Error:could not find file all_hosts!!!"
    exit 1
fi

if [ ! -f "./gpinstall_inspur.tar" ]; then
    echo "Error:could not find file gpinstall_inspur.tar!!!"
    exit 1
fi
if [[ $localyum_ip != "" ]];then
    #yum -y install gcc openssh-clients unzip zip OpenIPMI ipmitool xfsprogs kmod-xfs ntp
    if [ ! -d "/etc/yum.repos.d_backup" ]; then
        mkdir /etc/yum.repos.d_backup
    fi
    mv /etc/yum.repos.d/*.repo /etc/yum.repos.d_backup
    touch /etc/yum.repos.d/localyum.repo
    cat > /etc/yum.repos.d/localyum.repo <<EOF
        [localyum]
        name=localyum
        baseurl=http://ipforlocalyumrepo/Packages
        enabled=1
        gpgcheck=0
    EOF
    sed -i /baseurl/d /etc/yum.repos.d/localyum.repo
    echo "baseurl=http://${localyum_ip}/Packages" >>/etc/yum.repos.d/localyum.repo
    yum makecache
    yum -y update
fi
source /usr/local/greenplum-db/greenplum_path.sh
gpscp -f ./all_hosts ./gpinstall_inspur.tar =:/opt/gp

gpssh -f ./all_hosts 'tar -xvf /opt/gp/gpinstall_inspur.tar -C /opt/gp;chmod -R 777 /opt/gp;rpm -ivh /opt/gp/ed-1.1-3.3.el6.x86_64.rpm'
#rpm -ivh /opt/gp/ed-1.1-3.3.el6.x86_64.rpm
if [[ $localyum_ip != "" ]];then
    #master node download using local yum repository
    #./install-uselocalyum.sh $localyum_ip
    #source /usr/local/greenplum-db/greenplum_path.sh
    #segment nodes download using local yum repository
    gpssh -f ./all_segs /opt/gp/install-uselocalyum.sh $localyum_ip
    #if having standby node,download rpms for it
    if [ -f ./standby ]; then
       gpssh -f ./standby /opt/gp/install-uselocalyum.sh $localyum_ip
    fi
fi
{% endhighlight %}
##执行osconfig_batch.sh,进行操作系统参数优化
{% highlight bash %}
source /usr/local/greenplum-db/greenplum_path.sh
gpssh -f ./all_hosts /opt/gp/os.sh
{% endhighlight %}
os.sh内容如下：
{% highlight bash %}
#!/bin/bash
gpfiledir=/opt/gp
bakdir=$gpfiledir/backup
hosts=/etc/hosts
sourcedir=/opt/gp
ipaddr=$(echo `hostname -I`)
echo $ipaddr
#exit 0
cat $hosts | while read line
do
   ip=${line% *}
   hostname=${line#* }
   echo $ip$hostname >> ip.out
   if [ "$ipaddr" = "$ip" ]; then
      echo "开始修改$hostname机器的配置"
      hostname $hostname
      sed -i 's/^HOSTNAME=.*/HOSTNAME='"$hostname"'/' /etc/sysconfig/network
      break
   fi
done
#backup configs
if [ ! -d "$bakdir" ]; then
   mkdir -p $bakdir
   chmod 777 $bakdir
fi

cp /etc/security/limits.conf $bakdir
cp /etc/security/limits.d/90-nproc.conf $bakdir
cp /etc/rc.d/rc.local $bakdir
cp /boot/grub/grub.conf $bakdir
cp /etc/inittab $bakdir
cp /etc/sysctl.conf $bakdir

#/etc/limits/limits.conf
cat << EOF >> /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
EOF

#/etc/limits/limits.d/90-nproc.conf
cat << EOF >> /etc/security/limits.d/90-nproc.conf
* soft nproc 131072
* hard nproc 131072
EOF
#/etc/rc.d/rc.local
echo "blockdev --setra 16384 /dev/sd*" >> /etc/rc.d/rc.local
#/4.4.4/boot/grub/grub.conf
echo "elevator=deadline transparent_hugepage=never" >> /boot/grub/grub.conf
#/etc/inittab
sed -i 's/^id:*:initdefault:/id:3:initdefault:/' /etc/inittab
#/etc/sysctl.conf
cat << EOF > /etc/sysctl.conf
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
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
EOF
{% endhighlight %}
##重启各节点
##建立数据目录，进行磁盘挂载，逻辑卷需手动创建
####配置segment数据目录逻辑卷lvm,参照如下
{% highlight bash %}
/dev/sdb1
/dev/sdb2
{% endhighlight %}
####配置master和standby数据目录逻辑卷lvm-master-standby，参照如下
{% highlight bash %}
master :/dev/sdb1
standby :/dev/sdb1
{% endhighlight %}
####执行all_nodes_mount_batch.sh
{% highlight bash %}
#!/bin/sh
#config volum for all segment nodes
#gpssh -f ./all_segs /opt/gp/read.sh
#prepare to config master's volume
dir1=/data
dir2=/data/master
fstab=/etc/fstab
lvmconfig_ms=$1
lvmconfig_segs=$2
dirPath=$(cd "$(dirname "$0")"; pwd)
if [ ! -f "$lvmconfig_ms" ]; then
   lvmconfig_ms=$dirPath/lvm-master-standby
   if [ ! -f "$lvmconfig_ms" ]; then
     echo "Config file does not exist!"
     exit 0
   fi
fi
if [ ! -f "$lvmconfig_seg" ]; then
   lvmconfig_seg=$dirPath/lvm
   if [ ! -f "$lvmconfig_seg" ]; then
     echo "配置文件不存在"
     exit 0
   fi
fi

function isMounted(){
     res=0
     mountstr=$(mount | grep $1)
     #echo $mountstr > isout.log
     constr=$(echo "$mountstr" |grep $1)
     # echo $constr >> isout.log
     if [[ $constr != "" ]]; then
       res=1
     fi
     echo $res
}
#begin to config master's volume
master_str=`cat $lvmconfig_ms | grep master`
echo $master_str
master_device=${master_str#*:}
echo "Device name of master: "$master_device
if [[ $master_device == "" ]]; then
    exit 1
fi
res=$(isMounted $master_device)
echo $res
if [[ $res == 1 ]]; then
    umount $master_device
fi
mkfs.xfs -f $master_device
if [ -d $dir1 ]; then
    rm -rf $dir1
fi
mkdir $dir1
if [ -d $dir2 ]; then
    rm -rf $dir2
fi
#mkdir $dir2
fstabstr=${master_device}" "${dir1}" xfs rw,noatime,inode64,allocsize=16m 1 1"
echo $fstabstr
mount $master_device $dir1 && (grep -q "$fstabstr" $fstab || echo "$fstabstr" >> $fstab)
chmod -R 777 $dir1
mkdir -p $dir2

#segments
source /usr/local/greenplum-db/greenplum_path.sh
gpscp -f ./all_segs $dirPath/lvm =:/opt/gp
gpssh -f ./all_segs $dirPath/read.sh $dirPath/lvm
#if having standby node,config its volume
if [ -f ./standby ]; then
     echo "standby config file exists"
     gpscp -f ./standby $lvmconfig_ms =:/opt/gp
     cat ./standby | while read line
     do
     gpssh -f ./standby $dirPath/standby_mount.sh $lvmconfig_ms
     done
fi
{% endhighlight %}
##执行bashrx.sh，修改bashrc文件
{% highlight bash %}
#!/bin/sh
#edit bashrc on master node
source /usr/local/greenplum-db/greenplum_path.sh
cat << EOF >> /root/.bashrc
source /usr/local/greenplum-db/greenplum_path.sh
EOF
cat << EOF >> /home/gpadmin/.bashrc
    source /usr/local/greenplum-db/greenplum_path.sh
    MASTER_DATA_DIRECTORY=/data/master/gpseg-1
    export MASTER_DATA_DIRECTORY
EOF
#edit bashrc on all segments
gpssh -f ./all_segs /opt/gp/segs_modify_bashrc.sh
#if having standby node,modify its bashrc
if [ -f ./standby ]; then
   gpssh -f ./standby /opt/gp/stb_modify_bashrc.sh
   echo "exist standby node"
fi
{% endhighlight %}
##执行gtrust_seginstall.sh，gpadmin用户互信及其他机器安装GP，需输入密码gpadmin
{% highlight bash %}
source /usr/local/greenplum-db/greenplum_path.sh
chmod 777 /opt/gp/all_hosts
su - gpadmin -c "gpssh-exkeys -f /opt/gp/all_hosts"
chmod 777 /usr/local
gpseginstall -f all_segs -p gpadmin
if [ -f ./standby ];then
    gpseginstall -f ./standby -p gpadmin
fi
cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config /opt/gp
{% endhighlight %}
##配置ntp时钟同步，执行ntpsyc_start.sh
{% highlight bash %}
#!/bin/sh
#this scrip accepts two parameters,the first is ip of external ntp_server,the second one which is optional is the hostname of master node
#external ntp_server ip
ntp_server=$1
#configure master node
source /usr/local/greenplum-db/greenplum_path.sh
echo "server $ntp_server" >> /etc/ntp.conf
echo "server 127.127.1.0" >> /etc/ntp.conf
echo "fudge 127.127.1.0 stratum 8" >> /etc/ntp.conf
service ntpd restart
chkconfig ntpd on
#configure all segments
#hostname of master node
master=$2
if [[ $master = "" ]];then
    master="mdw"
fi
echo "master name is:$master"
#configure segment node
gpssh -f ./all_segs /opt/gp/ntpsyc_segs.sh $master

if [ -f ./standby ]; then
    cat ./standby | while read line
    do
    #gpssh -f ./standby echo "server $master prefer" >> /etc/ntp.conf
    #gpssh -f ./standby echo "server $ntp_server" >> /etc/ntp.conf
    gpssh -f ./standby /opt/gp/ntpsyc_config_standby.sh $master $ntp_server
    gpssh -f ./all_segs /opt/gp/ntpsyc_segs_with_standby.sh $line
    #gpssh -f ./all_segs echo "server $line" >> /etc/ntp.conf
    #gpssh -f ./standby sed -i "a\server $master prefer" /etc/ntp.conf
    #gpssh -f ./standby sed -i "a\server $ntp_server" /etc/ntp.conf
    #gpssh -f ./all_segs sed -i "a\server $line" /etc/ntp.conf
    done
    #gpssh -f ./standby service ntpd restart
    #gpssh -f ./standby chkconfig ntpd on
    #gpssh -f ./all_segs service ntpd restart
    #gpssh -f ./all_segs chkconfig ntpd on
fi
{% endhighlight %}
##修改配置文件进行初始化
{% highlight bash %}
gpinitsystem -c gpinitsystem_config -s smdw #若无standby，无需添加-s smdw
{% endhighlight %}