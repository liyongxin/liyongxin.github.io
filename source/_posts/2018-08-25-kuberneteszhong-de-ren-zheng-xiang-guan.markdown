---
layout: post
title: "kubectl的认证与授权"
date: 2018-08-25 14:41:00 +0800
comments: true
categories: kubernetes, k8s
---

关于k8s认证、授权相关的基础，只简单回顾，具体内容先参考如下文章：

- [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/)
- [Kubernetes 中的用户与身份认证授权](https://jimmysong.io/posts/user-authentication-in-kubernetes/)
- [Kubernetes RBAC - 基于角色的访问控制](https://github.com/rootsongjc/kubernetes-handbook/blob/master/guide/rbac.md)

##Kubernetes API的访问控制原理回顾
Kubernetes集群的访问权限控制由`kube-apiserver`负责，`kube-apiserver`的访问权限控制由身份验证(authentication)、授权(authorization)
和准入控制（admission control）三步骤组成，这三步骤是按序进行的：
![](/images/k8s-apiserver-access-control-overview.svg)

#### 身份验证（Authentication）
这个环节它面对的输入是整个`http request`，它负责对来自client的请求进行身份校验，支持的方法包括：client证书验证（https双向验证）、
`basic auth`、普通token以及`jwt token`(用于serviceaccount)。

APIServer启动时，可以指定一种Authentication方法，也可以指定多种方法。如果指定了多种方法，那么APIServer将会逐个使用这些方法对客户端请求进行验证，
只要请求数据通过其中一种方法的验证，APIServer就会认为Authentication成功；

在较新版本kubeadm引导启动的k8s集群的apiserver初始配置中，默认支持`client证书`验证和`serviceaccount`两种身份验证方式。
证书认证通过设置`--client-ca-file`根证书以及`--tls-cert-file`和`--tls-private-key-file`来开启。

在这个环节，apiserver会通过client证书或
`http header`中的字段(比如serviceaccount的`jwt token`)来识别出请求的`用户身份`，包括”user”、”group”等，这些信息将在后面的`authorization`环节用到。

#### 授权（Authorization）
这个环节面对的输入是`http request context`中的各种属性，包括：`user`、`group`、`request path`（比如：`/api/v1`、`/healthz`、`/version`等）、
`request verb`(比如：`get`、`list`、`create`等)。

APIServer会将这些属性值与事先配置好的访问策略(`access policy`）相比较。APIServer支持多种`authorization mode`，包括`Node、RBAC、Webhook`等。

APIServer启动时，可以指定一种`authorization mode`，也可以指定多种`authorization mode`，如果是后者，只要Request通过了其中一种mode的授权，
那么该环节的最终结果就是授权成功。在较新版本kubeadm引导启动的k8s集群的apiserver初始配置中，`authorization-mode`的默认配置是`”Node,RBAC”`。

Node授权器主要用于各个node上的kubelet访问apiserver时使用的，其他一般均由RBAC授权器来授权。
<!--more-->
##kubectl的授权认证
`kubeadm init`启动完master节点后，会默认输出类似下面的提示内容：
{% highlight bash %}
... ...
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
... ...
{% endhighlight %}
这些信息是在告知我们如何配置`kubeconfig`文件。按照上述命令配置后，master节点上的`kubectl`就可以直接使用`$HOME/.kube/config`的信息访问`k8s cluster`了。
并且，通过这种配置方式，`kubectl`也拥有了整个集群的管理员(root)权限。

很多K8s初学者在这里都会有疑问：

- 当`kubectl`使用这种`kubeconfig`方式访问集群时，`Kubernetes`的`kube-apiserver`是如何对来自`kubectl`的访问进行身份验证(`authentication`)和授权(`authorization`)的呢？
- 为什么来自`kubectl`的请求拥有最高的管理员权限呢？

####kubectl的身份认证（authentication）
我们先从kubectl使用的`kubeconfig`入手。kubectl使用的kubeconfig文件实质上就是`kubeadm init`过程中生成的`/etc/kubernetes/admin.conf`，
我们查看一下该kubeconfig文件的内容：
{% highlight bash %}
[root@k8s-master kubernetes]# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://192.168.8.33:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
{% endhighlight %}
关于kubeconfig文件的解释，可以在[这里](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)自行脑补。在这些输出信息中，我们着重提取到两个信息：
{% highlight bash %}
user name: kubernetes-admin
client-certificate-data: XXXX
{% endhighlight %}
前面提到过apiserver的authentication支持通过`tls client certificate、basic auth、token`等方式对客户端发起的请求进行身份校验，
从kubeconfig信息来看，kubectl显然在请求中使用了`tls client certificate`的方式，即客户端的证书。另外我们知道Kubernetes是没有user这种资源的，
通过k8s API也无法创建user。

那么kubectl的身份信息就应该“隐藏”在`client-certificate`的数据中，我们来查看一下。

首先把`/etc/kubernetes/kubelet.conf`中的`client-key-data`内容保存在`admin-client-certificate.txt`中
{% highlight bash %}
[root@k8s-master kubernetes]# cat admin-client-certificate.txt
LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBbjVLRFNhck1VZzdldmErbS9lOCswQ0tSaUl4ajY4a1lwazBYbFBEcEN4WmtLVWhpCkt2K0lqZG80RUFvWG5SWThHRUEvaUpVbmUyNzJBU3ZIeXB0OGFwOWtha3l6ZEZ1bXlKbmV2TjBuWldYeEFkQXMKUTlmcUZyVGcyTTFpTjJPYjFPb3VnR2pHenJ6bEw3Wm5xc3hQcHppOEtCeEVsM0dUVGFLVG5hSzFRWDUwNkt2aApIVkdXMTdkazZLWEltZDZhYWo0VmNNOXJaeEZ5bCs3SzFyQ2hPWXBhNzJUYWpqdEtuZm9FdzNxV0tmb0JXclBkCjJCRDhSaW4vcmlEbGNKTy9GRUt3azJQb1ZrcWx4bzdObWtld0Zma2txTVJrOHY3WTZVUTNvT1lhVDdsMFV0bEUKVzdCaEw1U0lkcTZIbGVzVU1CQkk4NDZEMnFZNk4zdGlndTFES3dJREFRQUJBb0lCQUFjMlJmekVYV3V3QkYwcQpYUy9JNmx2WjFCNEp5bEpUeW10cHZKRWN1a3VuL1dyb1BKZVk2UUVRUmN4anlHRnZLZFFtd3poWEZXdTh2aDJiCmJ2STNTTTVBMmZiNzlIaGoxQXZvK0dvc3pLVUdrSGYyZ3FtbVRvd3NMS1ZmMHZxUjQrOGhqbXg3VDlEME5KK04KYk80SlFlaGE1aFlpQU8rZlVIc0h5QWd0M0dkVFQ0eEh6elBzYjd0dXAxd1hJUVp5SnRkSTR1MGdaMmJFcUw2dQpGdm12RlF2RzY0S1dtQ0Eydk9zclNpb1QxTGpFMDJuZk11dWhvbEhxRVoxNEx4Lzk0elI4MXlYMjV2MnBCVE9yCitvOFBYNGhtUEprckJCVUNvQVZrSmlCYmVqMUh4TW1iQXlhM2ZlaXZBR01LcXh6b2wrVkltUmUwVE8xNk1WbWoKT1dzRGJYRUNnWUVBeW84aUNRR05iQ0p3QzIvOEkzaXU2TktGc2ZYZ08waC9RT0xhbGlReWlzbEx3anBadXNrVQpsaE1zR0xkQUFoZDVrZmVpTUF1UHRPbHNxb1NjeE80eDh1RUFqcUtndklVMHR1NjFjYWFPV29BTnczMkt2bDRnCk05UmxRbFVDMXRRT0htdFRsdURrSmRUQWFGQSswYjFMeVBnZFNlamV1eFl3dlR1MHV1R1BXNlVDZ1lFQXlhd0wKNEcyN05VT0JpSmpGalNPdGEvcjJtVHpEbFNPVUl6NlpyandVd1ZHbTQvRXdPZmx6czROYWd3R0RHK0tvM2lDagpySVJnUkhaMktCWXNjQUlKQk4xWVNpVjZxVzgvaEUzT3k0Y0EySFVZd1NmTktXNXZQSUNMcnVaMGxZWTJlcHFlClZLZXZUNUJWMlJvK3gxZFo2TE4vcXk0c3RHTzhTYWp0b1FxUUtvOENnWUFiQmNOTm5rWm1vYVYrOFI2YkFOT2MKdmRFV0w2NE5XcHVYWld3eDBYeG9wWGdVM2tId09Ea2wyRUx1dlN1dDI4SGRKa01kMDcwRkxvclBxTWRkUWtXcAptRGpCenBKUTlCaFhPenM3Z1RQR2dRVFZDcCtDeS8zUnpFa0I4Mk5nazRPYXJVakdmUlFTcy9KRE9FbFpJNzdECmZjNHllUDJWeWQwUXNiRm5xUVc5L1FLQmdRQzhZblU1c09jV2F6ZStCSTlOTjAyUk4zNVJTRnllblB5Tks3WGMKOXd5Z1JRaXpscUpwRldjS0FpSnppOThRRmx1T0cwa3BKd0xTRVNKd2NiNFM1eVBMb29RTnh4TGM0U21oQ2htcApMelFQL3RvZjNIRWVTYVdwQzU3dnd5Q1dhQ2ZOd1U4elh1dzVVMmVPQktFdURwL1M2cEhRc3JKWjAyeVlGaS9iCnBnVmphd0tCZ1FDbS9sejBRWXFUUFlQcG93RHpSZE84K1VaUmNVRHFKcFZ4TmdpRVdWN0pUN09qaENxVDdMUFAKQkZMYzdCY2hYTElMaElxOGhHNmJrL2dvNWt1TzhuMWpOTjFPTlNMaHh6RVp2VVgva000RE5FYjRaLzVON1JxWAp0Q0hMeFNDZXFUZVJ6K1FXRGRFL2pXaklEcU9McEdUVjdVbXJKN2kvU04rcE02dHZTWVFSUlE9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
{% endhighlight %}
然后对上述内容base64解密，然后存储在`admin.crt`文件中
{% highlight bash %}
[root@k8s-master kubernetes]# cat admin-client-certificate.txt |base64 -d > admin.crt
[root@k8s-master kubernetes]# cat admin.crt
-----BEGIN CERTIFICATE-----
MIIC8jCCAdqgAwIBAgIIQD98S9X+SVUwDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0xODA4MjAwNTUwMzlaFw0xOTA4MjAwNTUwNDFaMDQx
FzAVBgNVBAoTDnN5c3RlbTptYXN0ZXJzMRkwFwYDVQQDExBrdWJlcm5ldGVzLWFk
bWluMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAs5Rpnr5a68Cp4/EC
1IeebFl1z3ixDVT3pwR5YZgfy5E98IYB42c3fFrgV6fOuewOaW3bwrS9SZ4NMqB9
6TYXlZYkwxeQIp1HE+VSdbAHww7XE8zizv9/BSEynSqDglodmmDres0Cs9/3PG2F
B9OcAVycRzvxx87iecSzRwVF5DoopFbYkPJY/OjTQMeO8LX8YvoBYCD0ZpHxu7NE
b9qdUhPKH7ExbgSsSVZ75npjNtdDzlFzD4+tyYkvpBIYttOWHMFMQhipOyG+t1Af
ydVHzzsiAxgA/00ulxNCvhx1WXN+mL3PeDeBYMSFWo6cg60ih0nnNPk3rezrHoAg
294QxwIDAQABoycwJTAOBgNVHQ8BAf8EBAMCBaAwEwYDVR0lBAwwCgYIKwYBBQUH
AwIwDQYJKoZIhvcNAQELBQADggEBAFQ9ZvPCQsQQVR1kszsHip3qqcmIwUlkJiF6
YUVRzeG/QG15dIid5i87q5ZyK+NZhsuBrROnNUDSlg77jD61iHw+jREWd1pYAoK3
OyLcFd5q73xp+0aP1yEsRDnTmb7gzvKAYnFwKue7OZOVfpzWk0qakWkaPrzx6Bzp
G62X6p6701sL+9Gru56M8+tp+3/z635Z+56VjAFErzs5Sv5Pw5eAYxA12ebigNeh
0fIpVyPSZtA1MYgkbtqvjR6qxpgQUBvTCL7unNOGmdrvZI73fDLl+tTvlcgFDWcm
jlt8d2/5x/55BAfH/6LfqzfbDfOqlicKYvogLa7QE/A0uquVjLg=
-----END CERTIFICATE-----
{% endhighlight %}
然后用openssl工具查看crt证书内容
{% highlight bash %}
[root@k8s-master kubernetes]# openssl x509 -text -in admin.crt -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4629555607114762581 (0x403f7c4bd5fe4955)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Aug 20 05:50:39 2018 GMT
            Not After : Aug 20 05:50:41 2019 GMT
        Subject: O=system:masters, CN=kubernetes-admin
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
            ... ...

{% endhighlight %}
从证书输出的信息中，我们看到了下面这行：
{% highlight python %}
Subject: O=system:masters, CN=kubernetes-admin
{% endhighlight %}

说明在认证阶段，`apiserver`会首先使用`--client-ca-file`配置的CA证书去验证kubectl提供的证书的有效性,基本的方式
{% highlight bash %}
[root@k8s-master kubernetes]# openssl verify -CAfile /etc/kubernetes/pki/ca.crt /etc/kubernetes/admin.crt
/etc/kubernetes/admin.crt: OK
{% endhighlight %}
然后认证通过后，提取出签发证书时指定的CN(Common Name),`kubernetes-admin`，作为请求的用户名 (User Name),
O(Organization)， 从证书中提取该字段作为请求用户所属的组 (Group)，`group = system:masters`，然后传递给后面的授权模块
> X509 客户端证书
通过将 --client-ca-file=SOMEFILE 选项传递给 API server 来启用客户端证书认证。引用的文件必须包含一个或多个证书颁发机构，用于验证提交给 API server 的客户端证书。如果客户端证书已提交并验证，则使用 subject 的 Common Name（CN）作为请求的用户名。从 Kubernetes 1.4开始，客户端证书还可以使用证书的 organization 字段来指示用户的组成员身份。

####kubectl的授权
kubeadm在init初始引导集群启动过程中，创建了许多`default`的`role、clusterrole、rolebinding`和`clusterrolebinding`，
在k8s有关RBAC的官方文档中，我们看到下面一些`default clusterrole`列表:

![](/images/kubeadm-default-clusterrole-list.png)
其中第一个cluster-admin这个cluster role binding绑定了system:masters group，这和authentication环节传递过来的身份信息不谋而合。
沿着system:masters group对应的cluster-admin clusterrolebinding“追查”下去，真相就会浮出水面。

我们查看一下这一binding：
{% highlight bash %}
[root@k8s-master kubernetes]# kubectl get clusterrolebinding/cluster-admin -n kube-system -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2018-08-20T05:51:30Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "93"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/cluster-admin
  uid: 163adc34-a43d-11e8-89a4-000c2948e532
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters
 {% endhighlight %}
 我们看到在kube-system名字空间中，一个名为cluster-admin的clusterrolebinding将cluster-admin cluster role与system:masters Group绑定到了一起，
 赋予了所有归属于system:masters Group中用户cluster-admin角色所拥有的权限。

 我们再来查看一下cluster-admin这个role的具体权限信息：
 {% highlight bash %}
 [root@k8s-master kubernetes]# kubectl get clusterrole/cluster-admin -n kube-system -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2018-08-20T05:51:30Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "40"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/cluster-admin
  uid: 15f71927-a43d-11e8-89a4-000c2948e532
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
 {% endhighlight %}
从rules列表中来看，cluster-admin这个角色对所有resources、verbs、apiGroups均有无限制的操作权限，
即整个集群的root权限。于是kubectl的请求就可以操控和管理整个集群了。

###疑问
使用kubeadm-1.11.2版本初始化集群，即使不配置.kube/config文件，也可以直接访问到kubernetes cluster,
[官方文档](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)中对这块的记录如下：

To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:
 {% highlight bash %}
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
 {% endhighlight %}

Alternatively, if you are the root user, you can run:
 {% highlight bash %}
export KUBECONFIG=/etc/kubernetes/admin.conf
 {% endhighlight %}

- 关于非root用户，尝试新建了用户，且没有配置kubeconfig文件的情况下，依然可以通过kubectl直接访问集群。
- 关于`KUBECONFIG`环境变量，发现未设置该env，而且尝试把`/etc/kubernetes/admin.conf` 文件删除掉，重启apiserver的情况，依然可以访问

关于以上疑问已经弄明白，当我们在master节点中使用kubectl请求时，如果没有设置$HOME/.kube/config文件，则默认是通过本地的非安全端口来访问
apiserver，
{% highlight bash %}
[root@ip-172-31-10-236 centos]# kubectl get no -v 7
I0930 06:55:41.036661   13865 cached_discovery.go:72] returning cached discovery info from /root/.kube/cache/discovery/localhost_8080/alauda.io/v3/serverresources.json
I0930 06:55:41.037250   13865 cached_discovery.go:72] returning cached discovery info from /root/.kube/cache/discovery/localhost_8080/v1/serverresources.json
I0930 06:55:41.037484   13865 round_trippers.go:383] GET http://localhost:8080/api/v1/nodes
I0930 06:55:41.037502   13865 round_trippers.go:390] Request Headers:
I0930 06:55:41.037515   13865 round_trippers.go:393]     Accept: application/json
I0930 06:55:41.037528   13865 round_trippers.go:393]     User-Agent: kubectl/v1.7.3 (linux/amd64) kubernetes/2c2fe6e
I0930 06:55:41.046449   13865 round_trippers.go:408] Response Status: 200 OK in 8 milliseconds
I0930 06:55:41.067926   13865 cached_discovery.go:119] returning cached discovery info from /root/.kube/cache/discovery/localhost_8080/servergroups.json
I0930 06:55:41.068015   13865 cached_discovery.go:72] returning cached discovery info from /root/.kube/cache/discovery/localhost_8080/apiregistration.k8s.io/v1beta1/serverresources.json
 {% endhighlight %}
 如果
###小结
总结一下kubectl的认证过程：

![](/images/how-kubectl-be-authorized.png)
