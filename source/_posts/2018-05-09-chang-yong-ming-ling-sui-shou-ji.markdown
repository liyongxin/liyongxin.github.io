---
layout: post
title: "常用命令随手记"
date: 2018-05-09 16:33:42 +0800
comments: true
categories: shell
---
- ####统计系统各类连接数
{% highlight python %}
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
{% endhighlight %}
结果类似如下格式：
{% highlight bash %}
ESTABLISHED 23
TIME_WAIT 707
{% endhighlight %}

- ####ab命令简单压力测试
{% highlight python %}
ab -n 50000000 -c 10 http://127.0.0.1:8080/
{% endhighlight %}

- ####uwsgi reload
{% highlight python %}
ps afx;
kill -HUP $pid_of_uwsgi
{% endhighlight %}

- ####xargs相关
{% highlight bash %}
find . -type f -name "*.jpg" -print | xargs tar -czvf images.tar.gz

echo "nameXnameXnameXname" | xargs -dX
name name name name

echo "nameXnameXnameXname" | xargs -dX -n2
name name
name name
{% endhighlight %}

- ####多行输入单行输出：
{% highlight bash %}
cat test.txt | xargs

a b c d e f g h i j k l m n o p q r s t u v w x y z
{% endhighlight %}

- ####压缩加速
{% highlight bash %}
tar --use-compress-program=pigz -xvpf PKG-20180627.tar.gz
{% endhighlight %}

- ####curl发送get或者post请求
{% highlight bash %}
curl -H "Content-Type:application/json" -X POST -d '{"user": "admin", "passwd":"12345678"}' http://127.0.0.1:8000/login
curl -d "user=admin&passwd=12345678" http://127.0.0.1:8080/login
{% endhighlight %}
<!--more-->

- ####sed 匹配替换
{% highlight bash %}
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
{% endhighlight %}

- ####dd命令
{% highlight python %}
#创建一个大小为256M的文件
dd if=/dev/zero of=/swapfile bs=1024 count=262144
#测试硬盘的读写速度
dd if=/dev/zero bs=1024 count=1000000 of=/root/1Gb.file
dd if=/root/1Gb.file bs=64k | dd of=/dev/null
{% endhighlight %}

- ####iostat命令
{% highlight python %}
#安装
yum install sysstat
#测试机器设备读写情况，所有设备，间隔1s，执行10次
iostat -d -x 1 10
{% endhighlight %}

- ####route表操作
{% highlight python %}
#添加路由
route add default gw 192.168.1.1
route add -net 10.20.30.40 netmask 255.255.255.248 eth0
#删除路由
route del -net 10.20.30.48 netmask 255.255.255.248 gw 10.20.30.41
{% endhighlight %}

- ####tail 取输出结果的最后几行
{% highlight python %}
[root@cloud-cn-master-1 ~]# `which kubectl` get po -n alauda-system |grep alauda-redis
alauda-redis-67d89d58db-2gqdd                 1/1       Running   0          1d
alauda-redis-cluster-1-56489c6c87-4nwdc       1/1       Running   0          14d
alauda-redis-cluster-2-75fb5cb78b-h5l82       1/1       Running   0          14d
alauda-redis-cluster-3-8697f4746f-645sj       1/1       Running   1          14d
alauda-redis-cluster-4-7677cbcd8d-z8pvb       1/1       Running   0          14d
alauda-redis-cluster-5-564b7444d7-8z5dh       1/1       Running   1          63d
alauda-redis-cluster-6-7955ffc758-cp59h       1/1       Running   1          63d
[root@cloud-cn-master-1 ~]# `which kubectl` get po -n alauda-system |grep alauda-redis |tail -n1
alauda-redis-cluster-6-7955ffc758-cp59h       1/1       Running   1          63d
[root@cloud-cn-master-1 ~]# `which kubectl` get po -n alauda-system |grep alauda-redis |tail -n2
alauda-redis-cluster-5-564b7444d7-8z5dh       1/1       Running   1          63d
alauda-redis-cluster-6-7955ffc758-cp59h       1/1       Running   1          63d
{% endhighlight %}

- ####awk 指定行号输出
{% highlight python %}
#输出最后一行
[root@cloud-cn-master-1 ~]# kubectl get po -n alauda-system |grep alauda-redis |awk 'END{print $1}'
#输出指定行
[root@cloud-cn-master-1 ~]# kubectl get po -n alauda-system |grep alauda-redis |awk 'NR==1{print $1}'
{% endhighlight %}

- ####sed 删除指定行
{% highlight python %}
#删除前两行，输出剩下的行
[root@cloud-cn-master-1 ~]# `which kubectl` get po -n alauda-system |grep alauda-redis |sed 1,2d
#删除最后一行
[root@cloud-cn-master-1 ~]# `which kubectl` get po -n alauda-system |grep alauda-redis |sed '$'d
{% endhighlight %}

- ####find
find命令是根据文件的属性进行查找，如文件名，文件大小，所有者，所属组，是否为空，访问时间，修改时间等
{% highlight python %}
#在/tmp目录下查找大于10000字节并在最后2分钟内修改的文件
[root@cloud-cn-ake ~]# find /tmp -size +10000c -and -mtime +2

#查找当前文件系统中的所有目录并排序
[root@cloud-cn-ake ~]# find . -type d | sort
#查找`/var/log`目录下大于200M的以`.log`结尾的文件并删除掉
[root@yxli-onebox docker]# find  /var/log/  -name  "*.log" -size +200M -type f -exec rm {} \;
{% endhighlight %}

- ####grep
grep是根据文件的内容进行查找，会对文件的每一行按照给定的模式(patter)进行匹配查找
{% highlight python %}
#显示/usr/src目录下的文件(包含子目录)包含magic的行
[root@cloud-cn-ake ~]# grep -r magic /usr/src
#正则表达式
grep -E "\<(root|gao|uer1)\>" /etc/passwd
{% endhighlight %}