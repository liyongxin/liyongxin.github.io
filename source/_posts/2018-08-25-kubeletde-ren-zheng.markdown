---
layout: post
title: "kubelet的认证"
date: 2018-08-25 18:17:17 +0800
comments: true
categories: kubernetes, k8s
---
研究完[kubectl的认证与授权](../kuberneteszhong-de-ren-zheng-xiang-guan)，使用相同的方式去找kubelet的访问，
首先定位配置文件`/etc/kubernetes/kubelet.conf`，然后用相同的方式对`client-key-data`做base64解码，保存为kubelet.crt文件。

openssl查看crt证书，
{% highlight bash %}
[root@k8s-master kubernetes]# openssl x509 -text -in kubelet.crt -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 8126553944389053218 (0x70c751c18f5beb22)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Aug 20 05:50:39 2018 GMT
            Not After : Aug 20 05:50:42 2019 GMT
        Subject: O=system:nodes, CN=system:node:k8s-master
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:9f:92:83:49:aa:cc:52:0e:de:bd:af:a6:fd:ef:
    ... ...
{% endhighlight %}

得到我们期望的内容：
{% highlight bash %}
Subject: O=system:nodes, CN=system:node:k8s-master
{% endhighlight %}

> 关于Subject，在k8s中可以理解为角色绑定主体，RoleBinding或者ClusterRoleBinding可以将角色绑定到角色绑定主体（Subject）。
角色绑定主体可以是用户组（Group）、用户（User）或者服务账户（Service Accounts）


然后我尝试去k8s中找到一些关于`system:nodes`的RoleBindings或者ClusterRoleBindings,
{% highlight bash %}
[root@k8s-master kubernetes]# for bd in `kubectl get clusterrolebindings |awk '{print $1}'`; do echo $bd;kubectl get clusterrolebindings $bd -o yaml|grep 'system:nodes';done
NAME
Error from server (NotFound): clusterrolebindings.rbac.authorization.k8s.io "NAME" not found
cluster-admin
flannel
kubeadm:kubelet-bootstrap
kubeadm:node-autoapprove-bootstrap
kubeadm:node-autoapprove-certificate-rotation
  name: system:nodes
kubeadm:node-proxier
system:aws-cloud-provider
system:basic-user
... ...
{% endhighlight %}

结局有点意外，除了`kubeadm:node-autoapprove-certificate-rotation`外，没有找到system相关的rolebindings，显然和我们的理解不一样。
尝试去找[资料](https://github.com/rootsongjc/kubernetes-handbook/blob/master/guide/rbac.md)，发现了这么一段

![](/images/rbac.png)

Kubernetes 1.7开始, apiserver的启动中默认增加了`--authorization-mode=Node,RBAC`,也就是说，除了RBAC外，还有Node这种特殊的授权方式，

继续查找[kubernetes官方的信息](https://kubernetes.io/docs/reference/access-authn-authz/node/),得知
{% highlight bash %}
Node authorization is a special-purpose authorization mode that specifically authorizes API requests made by kubelets.
{% endhighlight %}
