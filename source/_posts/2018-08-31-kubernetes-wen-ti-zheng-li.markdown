---
layout: post
title: "kubernetes 问题整理"
date: 2018-08-31 10:36:27 +0800
comments: true
categories: kubernetes,docker
---
本文用以kubernetes 运维过程中遇到问题汇总，方便日后回顾～
## kubernetes多网卡导致的问题
部署机器是阿里云，有两块网卡，`eth0`外网，`eth1 vpc`内网，集群的路由信息如下
{% highlight bash %}
[root@10 src]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gateway         0.0.0.0         UG    0      0        0 eth1
10.0.0.0        10.81.35.247    255.0.0.0       UG    0      0        0 eth0
10.81.32.0      0.0.0.0         255.255.252.0   U     0      0        0 eth0
39.107.40.0     0.0.0.0         255.255.252.0   U     0      0        0 eth1
100.64.0.0      10.81.35.247    255.192.0.0     UG    0      0        0 eth0
link-local      0.0.0.0         255.255.0.0     U     1002   0        0 eth0
link-local      0.0.0.0         255.255.0.0     U     1003   0        0 eth1
172.16.0.0      10.81.35.247    255.240.0.0     UG    0      0        0 eth0
192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 cni0
192.168.0.0     0.0.0.0         255.255.240.0   U     0      0        0 docker0
192.168.1.0     192.168.1.0     255.255.255.0   UG    0      0        0 flannel.1
192.168.2.0     192.168.2.0     255.255.255.0   UG    0      0        0 flannel.1
{% endhighlight %}

- ####docker0网段与cni0网段冲突问题

docker启动时没有指定bip，从上述路由规则发现，docker0使用了192.168的段,刚好给flannel设置的cidr段冲突，
所以需要给docker修改默认的网段，解决方法是给docker配置bip网段,然后重启docker，观察docker0的route规则

{% highlight bash %}
[root@10 ~]# cat /etc/docker/daemon.json
{
    "insecure-registries": [],
    "graph": "/var/lib/docker",
    "bip": "172.17.0.1/16",
    "registry-mirrors": ["https://registry.docker-cn.com"],
    "storage-driver": "devicemapper",
    "storage-opts": ["dm.use_deferred_removal=true", "dm.use_deferred_deletion=true"],
    "storage-opts": [
        "dm.thinpooldev=/dev/mapper/docker-thinpool",
        "dm.min_free_space=0%",
        "dm.use_deferred_deletion=true",
        "dm.use_deferred_removal=true",
        "dm.fs=ext4"
    ],
    "log-driver": "fluentd",
    "log-opts":
    {
        "fluentd-address": "localhost:24224",
        "tag": "docker.{{.ID}}",
        "fluentd-async-connect": "true"
    }
}
{% endhighlight %}

- ####集群初始化问题

使用`kubeadm`搭建，若未指定--advertise-address地址则k8s默认拿default网卡，
而机器的default网卡刚好是外网eth0，所以初始化集群使用的地址是外网地址，导致一堆端口需要开，然后Node加入集群失败，解决办法是kubeadm初始化的
时候指定--advertise-address为内网地址,下面为kubeadm init使用的conf文件
<!--more-->
{% highlight bash %}
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  bindPort: 6443
etcd:
  endpoints:
  #sed -i "/#ETCD_ENDPOINTS/a\  - http://123.456:2379/g" ./abc.yml
  #ETCD ENDPOINTS
  - http://10.81.32.150:2379
apiServerExtraArgs:
  apiserver-count: "1"
  insecure-port: "8080"
  advertise-address=10.81.32.150
  service-node-port-range: "30000-32000"
  admission-control: "Initializers,NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,ResourceQuota"
  feature-gates: "MountPropagation=true"
  endpoint-reconciler-type: "lease"
controllerManagerExtraArgs:
  pod-eviction-timeout: "30s"
  node-monitor-period: "2s"
  node-monitor-grace-period: "16s"
controllerManagerExtraVolumes:
- name: k8s
  hostPath: /etc/kubernetes
  mountPath: /etc/kubernetes
imageRepository: index.docker.cn/claas
networking:
  podSubnet: 192.168.0.0/16
kubernetesVersion: v1.9.6
token: 8d775a.8f70da6999842a27
tokenTTL: "0"
apiServerCertSANs:
- 127.0.0.1
- amazonaws.com.cn
- amazonaws.com
- 10.81.32.150
- 10.81.32.150
{% endhighlight %}
- ####flannel网络问题
多网卡导致flannel网络选择网卡错误,flannel在初始化的时候会默认找defalut网卡，如果需要指定，则在flannel的
初始化yaml文件中通过--iface指定网卡，
{% highlight bash %}
  ... ...
      containers:
      - name: kube-flannel
        image: index.alauda.cn/claas/flannel:v0.9.1
        command: [ "/opt/bin/flanneld", "--ip-masq", "--kube-subnet-mgr", "--iface=eth0"]
        securityContext:
          privileged: true
        env:
  ... ...
 {% endhighlight %}