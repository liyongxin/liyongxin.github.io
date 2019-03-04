---
layout: post
title: "somethings about kubernetes"
date: 2019-02-28 11:12:56 +0800
comments: true
categories: kubernetes k8s
---

- k8s默认驱逐设置
{% highlight bash %}
// DefaultEvictionHard includes default options for hard eviction.
var DefaultEvictionHard = map[string]string{
        "memory.available":  "100Mi",
        "nodefs.available":  "10%",
        "nodefs.inodesFree": "5%",
        "imagefs.available": "15%",
}
{% endhighlight %}

- kubernetes 滚动升级
{% highlight bash %}
# run test deploy
[root@k8s-master ~]# kubectl run --image=nginx --port=80 --replicas=2 yxli-nginx
# scale replica
[root@k8s-master ~]# kubectl scale --replicas=1 deploy/yxli-nginx
[root@k8s-master ~]# kubectl get deploy
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
busybox      2         2         2            2           39d
busybox1     1         1         1            1           39d
yxli-nginx   1         1         1            1           2h
# update image
[root@k8s-master ~]# kubectl set image  deploy/yxli-nginx yxli-nginx=nginx:alpine
# 查看升级历史
[root@k8s-master ~]# kubectl rollout history deploy/yxli-nginx
deployments "yxli-nginx"
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
# 回顾至上次版本
[root@k8s-master ~]# kubectl rollout undo deploy/yxli-nginx
[root@k8s-master ~]# kubectl rollout history deploy/yxli-nginx
deployments "yxli-nginx"
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
# 回滚至指定版本
[root@k8s-master ~]# kubectl rolloutundo deployment/lykops-dpm --to-revision=2
{% endhighlight %}
<!--more-->
- 查看docker使用的cpu核心数
{% highlight bash %}
[root@yxli-onebox docker]# pwd
/sys/fs/cgroup/cpuset/docker
[root@yxli-onebox docker]# cat cpuset.cpus
0-7
{% endhighlight %}
