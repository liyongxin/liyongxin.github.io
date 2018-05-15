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

ab -n 50000000 -c 10 http://127.0.0.1:8080/
- grep 正则或的写法
grep -E "\<(root|gao|uer1)\>" /etc/passwd
