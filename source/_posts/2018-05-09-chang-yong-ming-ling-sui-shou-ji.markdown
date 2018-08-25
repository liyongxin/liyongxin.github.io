---
layout: post
title: "常用命令随手记"
date: 2018-05-09 16:33:42 +0800
comments: true
categories: 
---
- 统计系统各类连接数
{% highlight python %}
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
{% endhighlight %}
结果类似如下格式：
{% highlight bash %}
ESTABLISHED 23
TIME_WAIT 707
{% endhighlight %}
- ab命令简单压力测试
{% highlight python %}
ab -n 50000000 -c 10 http://127.0.0.1:8080/
{% endhighlight %}

- grep 正则或的写法
{% highlight python %}
grep -E "\<(root|gao|uer1)\>" /etc/passwd
{% endhighlight %}

- uwsgi reload
{% highlight python %}
ps afx;
kill -HUP $pid_of_uwsgi
{% endhighlight %}

- xargs相关
{% highlight bash %}
find . -type f -name "*.jpg" -print | xargs tar -czvf images.tar.gz

echo "nameXnameXnameXname" | xargs -dX
name name name name

echo "nameXnameXnameXname" | xargs -dX -n2
name name
name name
{% endhighlight %}

- 多行输入单行输出：
{% highlight bash %}
cat test.txt | xargs

a b c d e f g h i j k l m n o p q r s t u v w x y z
{% endhighlight %}

- 压缩加速
{% highlight bash %}
tar --use-compress-program=pigz -xvpf PKG-20180627.tar.gz
{% endhighlight %}

- curl发送get或者post请求
{% highlight bash %}
curl -H "Content-Type:application/json" -X POST -d '{"user": "admin", "passwd":"12345678"}' http://127.0.0.1:8000/login
curl -d "user=admin&passwd=12345678" http://127.0.0.1:8080/login
{% endhighlight %}