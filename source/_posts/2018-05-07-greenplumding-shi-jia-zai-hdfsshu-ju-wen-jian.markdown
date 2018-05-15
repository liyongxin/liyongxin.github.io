---
layout: post
title: "CentOS使用GreenPlum定时加载HDFS数据文件"
date: 2018-05-03 14:08:00 +0800
comments: true
categories: GreenPlum
---

最近有个场景，数据会定时写入hdfs，需要GP从hdfs中将数据载入。网上关于gp连接hdfs的介绍不是很多，实现的过程中走了很多弯路。
整个过程大概分为4步：

- **安装JDK** ：java必备；
- **安装必要的PHD软件包** ：安装phd中必要的组件，让GPDB主机作为Hadoop的client；
- **设置GPDB** ：配置gpconfig；
- **定时任务** ：利用cron任务实现定时加载数据文件；

### 安装JDK
推荐版本是1.7.x，注意需要在所有节点进行安装，安装完成后添加以下内容到`gpadmin`用户对应的`.bashrc`文件中
{% highlight ruby %}
export JAVA_HOME=/opt/jdk1.7.0_45
{% endhighlight %}

如果还会提示找不到`Error: JAVA_HOME is not set and could not be found.`尝试将上述命令添加到`/etc/environment中`.
编辑`/etc/profile`文件，添加如下内容：
{% highlight bash %}
export JAVA_HOME=/opt/jdk1.7.0_45
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
{% endhighlight %}
最后执行如下命令验证安装是否成功：
{% highlight bash %}
source /etc/profile
java -version
{% endhighlight %}

### 安装必要的PHD软件包
默认安装完GP后，无法直接ipc连接远程hdfs，官方推荐安装phd的一些软件，当然拷贝一份hadoop的包到本地也可以，目的都是让GPDB中的主机作为hadoop的客户端，能进行hdfs的访问。
所有的软件均可在phd的安装包中获得，我下载的是PHD-2.0.1.0-148版本，整个安装包大概805MB，以下rpm是需要顺序安装的：
``` go
rpm -ivh bigtop-jsvc-1.0.15_gphd_3_0_1_0-148.x86_64.rpm
rpm -ivh bigtop-utils-0.4.0_gphd_3_0_1_0-148.noarch.rpm
rpm -ivh zookeeper-3.4.5_gphd_3_0_1_0-148.noarch.rpm
yum -y install nc
rpm -ivh hadoop-2.2.0_gphd_3_0_1_0-148.x86_64.rpm
rpm -ivh hadoop-yarn-2.2.0_gphd_3_0_1_0-148.x86_64.rpm
rpm -ivh hadoop-mapreduce-2.2.0_gphd_3_0_1_0-148.x86_64.rpm
rpm -ivh hadoop-hdfs-2.2.0_gphd_3_0_1_0-148.x86_64.rpm
```
安装gphd的时候需要nc，所以先yum安装一下。
> **注意：**官方说所有的segment节点需要安装这些软件，实际过程中整个GPDB中的机器都需要安装才可以运行，否则会提示无法加载class的错误。

###设置GPDB
在gp中使用gpadmin用户进行设置以下两个配置项，具体的值需要根据官方的资料进行匹配
{% highlight bash %}
gpconfig -c gp_hadoop_home -v "'/usr/lib/gphd'"
gpconfig -c gp_hadoop_target_version -v "'gphd-2.0'"
{% endhighlight %}
<!--more-->
其他hadoop版本以及target对应的值可参考下图：
<table>
   <tr>
      <td>Hadoop Distribution</td>
      <td>Version</td>
      <td>gp_hadoop_target_version</td>
   </tr>
   <tr>
      <td>Pivotal HD</td>
      <td>Pivotal HD 2.0Pivotal HD 1.01</td>
      <td>gphd-2.0</td>
   </tr>
   <tr>
      <td>Greenplum HD</td>
      <td>Greenplum HD 1.2</td>
      <td>gphd-1.2</td>
   </tr>
   <tr>
      <td></td>
      <td>Greenplum HD 1.1</td>
      <td>gphd-1.1 (default)</td>
   </tr>
   <tr>
      <td>Cloudera</td>
      <td>CDH 5.2, 5.3</td>
      <td>cdh5</td>
   </tr>
   <tr>
      <td></td>
      <td>CDH 5.0, 5.1</td>
      <td>cdh4.1</td>
   </tr>
   <tr>
      <td></td>
      <td>CDH 4.12 - CDH 4.7</td>
      <td>cdh4.1</td>
   </tr>
   <tr>
      <td>Hortonworks Data Platform</td>
      <td>HDP 2.1, 2.2</td>
      <td>hdp2</td>
   </tr>
   <tr>
      <td>MapR3</td>
      <td>MapR 4.x</td>
      <td>gpmr-1.2</td>
   </tr>
   <tr>
      <td></td>
      <td>MapR 1.x, 2.x, 3.x</td>
      <td>gpmr-1.0</td>
   </tr>
   <tr>
      <td></td>
   </tr>
</table>
> **注意：**设置完成之后需要完全重启GPDB，官方提示只需gpstop -u重新load即可，实测中会报错找不到java command。
测试hdfs命令
{% highlight ruby %}
[gpadmin@mdw opt]$ hdfs dfs -ls hdfs://10.110.17.181:8020/
Found 13 items
drwxrwxrwx - root supergroup 0 2016-01-19 14:53 hdfs://10.110.17.181:8020/hbase
drwxrwxrwx - root supergroup 0 2016-01-21 08:34 hdfs://10.110.17.181:8020/liyongxin
-rwxrwxrwx 2 root supergroup 10 2016-01-19 14:49 hdfs://10.110.17.181:8020/lyx.dat
-rwxrwxrwx 3 gpadmin supergroup 16 2016-01-19 19:54 hdfs://10.110.17.181:8020/lyx.txt
drwxrwxrwx - root supergroup 0 2016-01-21 16:16 hdfs://10.110.17.181:8020/mpp_data
drwxrwxrwx - root supergroup 0 2016-01-21 16:16 hdfs://10.110.17.181:8020/mpp_tmp
drwxrwxrwx - root supergroup 0 2016-01-20 14:20 hdfs://10.110.17.181:8020/tmp
drwxrwxrwx - root supergroup 0 2016-01-21 09:45 hdfs://10.110.17.181:8020/user
{% endhighlight %}
将数据文件put到hdfs
{% highlight ruby %}
[gpadmin@mdw opt]$ cat lyx.txt
east,15
west,25
[gpadmin@mdw opt]$ hdfs dfs -put lyx.txt hdfs://10.110.17.181:8020/mpp_data/1.txt
[gpadmin@mdw opt]$ hdfs dfs -ls hdfs://10.110.17.181:8020/mpp_data/
Found 1 items
-rw-r--r-- 3 gpadmin supergroup 16 2016-01-21 17:00 hdfs://10.110.17.181:8020/mpp_data/lyx.txt
{% endhighlight %}
建立外部表并测试,查询数据
{% highlight ruby %}
lyx=# select * from test_hdfs_ext1;
 age | name
------+------
 east | 15
 west | 25
(2 rows)
{% endhighlight %}

###脚本定时加载数据文件
由于文件会不定时的写入到hdfs目录中，建立外部表采用通配符匹配的方式，但如果在执行加载的过程中有文件写入到目录，会将该文件一并加载到gp中，这样就无法实现对已加载的文件的记录，所以脚本中先将当前已有文件剪切到一个临时目录，加载完成后全部删除临时目录中文件，大致内容如下：
{% highlight bash %}
database=postgres
tablename=hdfs
columns='age text, name text'
#echo $hdfsurl$filepath
hdfs_nns=('10.110.17.181:8020' '10.110.17.182:8020')
tmp_path=mpp_tmp
data_path=mpp_data
function get_hdfs_url(){
 standby="ls: Operation category READ is not supported in state standby";
 res=hdfs_nns[0]
 for var in ${hdfs_nns[@]};
     do
         res=`hdfs dfs -ls hdfs://$var/ 2>&1`;
         if [ "$res" = "$standby" ];then
            res=''
         else
            res=$var
            break
         fi
     done
 echo $res;
}
hdfs_url=$(get_hdfs_url)

function is_file_exist(){
 hdfs dfs -ls hdfs://$hdfs_url/$data_path/ | awk '$1 ~ /Found/ {print 1}'
}

fss=$(is_file_exist)
if [ "$fss" != "1" ];
 then
 echo 'no data file'
 exit 1
fi
hdfs dfs -mv hdfs://$hdfs_url/$data_path/*.txt hdfs://$hdfs_url/$tmp_path/
psql -U gpadmin -d $database << EOF
create external table hdfs_ext ($columns) location('gphdfs://$hdfs_url/$tmp_path/*.txt') format 'text' (delimiter ',');
insert into $tablename select * from hdfs_ext;
drop external table hdfs_ext;
EOF
#drop external table hdfs_ext;
hdfs dfs -rm -r hdfs://$hdfs_url/$tmp_path/*.txt

#hdfs dfs -ls $hdfsurl$filepath | awk 'BEGIN{cur=0;filenames[cur]=0;all=0;}
# $1 ~ /Found/ {all=$2;print all;}
# $8 ~ /hdfs*/ {print $8;
# filenames[cur]=$8;cur++;print "cur="+'cur';
# if(all==cur){
 #external tables
 # psql -U gpadmin -d lyx << EOF
 # CREATE SCHEMA ;
 #EOF
# print "over,all="'all'}
#}'
{% endhighlight %}
最后放入crontab中即可实现定时增量的处理

