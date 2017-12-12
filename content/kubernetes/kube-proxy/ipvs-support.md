---
categories : [kubernetes"]
tags : ["ipvs","kubernetes","lvs"]
type : "post"
title :  "ipvs 在 k8s 1.8 中的应用"
subtitle : "ipvs 在 k8s 1.8.3 中的应用"
date : "2017-09-24"
draft : false
description: "lvs 在 k8s 1.8 中的应用-ipvs"
---

- [在 kubernetes 1.8.3 上设置 kube-proxy 的 ipvs 模式](#org6e9048b)
  - [ipvs 简单介绍](#org0ba5832)
  - [kube-proxy ipvs 模式设置](#org5c503c4)
    - [centos 关闭 SELinux](#org486cd30)
    - [firewall 关闭](#orgb7f74da)
    - [内核参数调整](#org061f978)
    - [load kernel ko](#org405e0d6)
    - [修改 kube-proxy 配置](#orgff2cbf8)
    - [错误以及解决方案](#org936dedf)
  - [测试](#org3b43ce4)
  - [调试编译](#org7dd1fe7)
    - [编译](#org4d6d39e)
    - [debug 命令](#orga7c46a3)
  - [扩展阅读](#orge8e3134)
  - [Test validation](#orgd8c41f5)
    - [Functionality tests, all below traffic should be reachable](#org4f4d143)
    - [Test service with ServiceAffinity=ClientIP. Validate IPVS has persistence for the service.](#org62c56bc)
  - [IPCS vs IPTables 对比](#org64125a5)
    - [IPTables:](#org7bd76ba)
    - [Why use IPVS?](#orgfbe4d57)
  - [注意](#orgb020cb9)
    - [ipvs 与 kube-dns](#org97382f9)


<a id="org6e9048b"></a>

# 在 kubernetes 1.8.3 上设置 kube-proxy 的 ipvs 模式

在 k8s 1.8 中，kube-proxy 对 ipvs 的支持进入到 alpha 版本,目前 ipvs 只支持 nat 模式，本 片文章主要从讲解 kube-proxy ipvs 模式的配置，以及验证方式，其他相关知识会会有响 应的章节介绍，但不是主要内容


<a id="org0ba5832"></a>

## ipvs 简单介绍

ipvs 是于 netfilt 结合在一起使用的，ipvs 作为 netfilter 的模块存在, ipvs 附属在 INPUT 链上工作的。ipvs 支持三种工作模式：Nat，ip tunel,DirectMode,针对这几种默认 以及 ipvs 的简单介绍，见扩展阅读


<a id="org5c503c4"></a>

## kube-proxy ipvs 模式设置


<a id="org486cd30"></a>

### centos 关闭 SELinux

```sh
# 编辑 /etc/selinux/config 文件；确保 SELINUX=disabled
docker1.node ➜  ~ cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```


<a id="orgb7f74da"></a>

### firewall 关闭

```sh
systemctl stop firewalld
systemctl disable firewalld

```


<a id="org061f978"></a>

### 内核参数调整

打开 ip forward 开关

```sh
docker1.node ➜  ~ cat /etc/sysctl.conf
# sysctl settings are defined through files in
# /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
#
# Vendors settings live in /usr/lib/sysctl.d/.
# To override a whole file, create a new file with the same in
# /etc/sysctl.d/ and put new settings there. To override
# only specific settings, add a file with a lexically later
# name in /etc/sysctl.d/ and put new settings there.
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
```

然后执行 sysctl -p 使之生效

```sh
# sysctl -p
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```


<a id="org405e0d6"></a>

### load kernel ko

虽然 ipvs 早已整合到 kernel 主线中，但默认 ipvs 的 kernel module 默认是不加载的，需 要手动挂载，见官方：<https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs#load-ipvs-kernel-modules>

```sh
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
```

执行完后，可通过以下命令验证是否挂载成功

```sh
lsmod | grep ip_vs
```


<a id="orgff2cbf8"></a>

### 修改 kube-proxy 配置

1.  由于 ipvs 处于 alpha 阶段，首先通过&#x2013;feature-gates 开启 ipvs kube-proxy 添加参数 &#x2013;feature-gates=SupportIPVSProxyMode=true
2.  增加 ipvs-min-sync-period、&#x2013;ipvs-sync-period、&#x2013;ipvs-scheduler 三个参数用 于调整 ipvs，具体参数值请自行查阅 ipvs 文档

修改后后的配置

```yaml
spec:
  containers:
  - command:
    - /usr/local/bin/kube-proxy
    - --kubeconfig=/var/lib/kube-proxy/kubeconfig.conf
    - --cluster-cidr=10.244.0.0/16
    - --feature-gates=SupportIPVSProxyMode=true
    - --proxy-mode=ipvs
    - --ipvs-scheduler=rr
    - --ipvs-sync-period=1m
    image: 192.168.1.55/tenx_containers/kube-proxy-amd64:v1.8.3
    imagePullPolicy: IfNotPresent

```

由于 kube-proxy 是由 daemonSet 方式启动，修改后 pod 会自动重启


<a id="org936dedf"></a>

### 错误以及解决方案

1.  在修改后，kube-proxy 重启后会存在以下错误

```sh
ERROR: ../libkmod/libkmod.c:557 kmod_search_moddep() could not open moddep file '/lib/modules/3.10.0-514.el7.x86_64/modules.dep.bin'`, error: exit status 1"
```

这时，通过 `hostPath` 挂载方式将/lib/modules 挂载到 pod 中即可解决


<a id="org3b43ce4"></a>

## 测试

下载镜像 docker pull cloudnativelabs/whats-my-ip， 网络不是很好的话， 可以推到 是有镜像中，接下来部署 deployment 和 service

1.  部署 deployment

```sh
kubectl run myip --image=192.168.1.55/tenx_containers/whats-my-ip:latest --replicas=3 --port=8080
```

1.  查看创建后的 pod
    
    ```sh
    [root@harbor-master ~]# kubectl get pods -o wide
    NAME                    READY     STATUS    RESTARTS   AGE       IP                NODE
    myip-7955c5cd44-k7vvb   1/1       Running   0          30s       192.168.240.170   harbor-slave1
    myip-7955c5cd44-vhn9q   1/1       Running   0          30s       192.168.240.169   harbor-slave1
    myip-7955c5cd44-x224m   1/1       Running   0          30s       192.168.100.12    harbor-slave
    ```
2.  pod 创建好了，接下来创建 service
    
    ```sh
    kubectl expose deployment myip --port=8080 --target-port=8080 --type=NodePort
    ```
3.  创建创建的 service 以及 endpoint 信息
    
    ```sh
    [root@harbor-master ~]# kubectl get svc
    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          3d
    myip         NodePort    10.109.172.94   <none>        8080:30651/TCP   4s
    -----------
    [root@harbor-master ~]# kubectl get ep -owide
    NAME         ENDPOINTS                                                       AGE
    kubernetes   192.168.1.11:6443                                               3d
    myip         192.168.100.12:8080,192.168.240.169:8080,192.168.240.170:8080   22s
    ```
    
    1.  到这一步，deployment 和服务都已经搭建好了，下面通过 curl 命令进行测试，是否 rr 模式
        
        ```sh
        [root@harbor-master ~]# curl harbor-master:30651
        HOSTNAME:myip-7955c5cd44-vhn9q IP:192.168.240.169
        [root@harbor-master ~]# curl harbor-master:30651
        HOSTNAME:myip-7955c5cd44-x224m IP:192.168.100.12
        [root@harbor-master ~]# curl harbor-master:30651
        HOSTNAME:myip-7955c5cd44-k7vvb IP:192.168.240.170
        [root@harbor-master ~]# curl harbor-master:30651
        HOSTNAME:myip-7955c5cd44-vhn9q IP:192.168.240.169
        ```
        
        通过第五步的输出可看出，当前是 rr 模式访问的各个 endpoint


<a id="org7dd1fe7"></a>

## 调试编译


<a id="org4d6d39e"></a>

### 编译

ipvs 的实现依赖 libnl 动态链接库，是 c 实现的，当想变易 kube-proxy 时，需要在 linux 环 境（centos/ubuntu）下，并安装 libln 动态链接库，安装命令

```sh
apt-get install libnl-dev
apt-get install libnl-genl-3-dev
```

centos 7.4 默认已经安装了


<a id="orga7c46a3"></a>

### debug 命令

ipvs 也有用户态的执行命令 ipvsadm，可通过 ipvsadm 查看当前链接情况

```sh
apt-get install ipvsadm
```


<a id="orge8e3134"></a>

## 扩展阅读

这里就不对 ipvs 的概念做过多的介绍，大家可以根据 nn 以下地址脑补下

-   "ipvs 负载均衡（三）ipvs 三种工作方式 - 绯浅 yousa 的笔记 - CSDN 博客" <http://blog.csdn.net/qq_15437667/article/details/50644594>
-   "ipvs 负载均衡（四）ipvs 三种工作方式之 tun 模式 - 绯浅 yousa 的笔记 - CSDN 博客" <http://blog.csdn.net/qq_15437667/article/details/50664786>
-   "LVS 详解系列：初识 LVS" <http://www.zsythink.net/archives/2134>
-   "netfilter/iptables 简介" <https://www.ibm.com/developerworks/cn/linux/network/s-netip/index.html>
-   "Scaling Kubernetes to Support 50000 Services.pptx - Google 幻灯片" <https://docs.google.com/presentation/d/1BaIAywY2qqeHtyGZtlyAp89JIZs59MZLKcFLxKE6LyM/edit#slide=id.p3>
-   "Proposal for alpha version IPVS load balancing mode.docx - <https://docs.google.com/document/d/1YEBWR4EWeCEWwxufXzRM0e82l_lYYzIXQiSayGaVQ8M/edit>
-   ipvsad "Ipvsadm - LVSKB" <http://kb.linuxvirtualserver.org/wiki/Ipvsadmm>
-   ipvs proposal <https://github.com/kubernetes/community/pull/692/files>
-   "使用 LVS 实现负载均衡原理及安装配置详解 - 肖邦 linux - 博客园" <http://www.cnblogs.com/liwei0526vip/p/6370103.html>
-   ipvs offical document "kubernetes/pkg/proxy/ipvs at master · kubernetes/kubernetes" <https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs#load-ipvs-kernel-modules>


<a id="orgd8c41f5"></a>

## Test validation


<a id="org4f4d143"></a>

### Functionality tests, all below traffic should be reachable

1.  Traffic accessing service IP

    -   container -> serviceIP -> container (same host)
    -   container -> serviceIP -> container (cross host)
    -   container -> serviceIP -> container (same container)
    -   host -> serviceIP -> container (same host)
    -   host -> serviceIP -> container (cross host)

2.  Access service via NodePort

3.  Access service via external IP

4.  Traffic between container and host (not via service IP)

    -   container -> container (same host)
    -   container -> container (cross host)
    -   container -> container (same container)
    -   host -> container (same host)
    -   host -> container (cross host)
    -   container -> host (same host)
    -   container -> host (cross host)


<a id="org62c56bc"></a>

### Test service with ServiceAffinity=ClientIP. Validate IPVS has persistence for the service.


<a id="org64125a5"></a>

## IPCS vs IPTables 对比


<a id="org7bd76ba"></a>

### IPTables:

-   Manipulates tables provided by the linux firewall
-   IPTables is more flexible and can manipulate packets at different stages: Pre-routing, post-routing, forward, input, output.
-   IPTables has more operations: SNAT, DNAT, reject packets, port translation etc.


<a id="orgfbe4d57"></a>

### Why use IPVS?

-   Better performance (Hashing vs. Chains)
-   More load balancing algorithms
-   Round robin, source/destination hashing.
-   Based on least load, least connection or locality, can assign weights to servers.
-   Supports server health checks and connection retries
-   Supports sticky sessions


<a id="orgb020cb9"></a>

## 注意

最后说一点: ipvs 尚未稳定，请慎用；而且 &#x2013;masquerade-all 选项与 Calico 安全策略控制不兼容，请酌情考虑使用(Calico 在做网络策略限制的时候要求不能开启此选项)


<a id="org97382f9"></a>

### ipvs 与 kube-dns

当 kube-proxy 设置为 ipvs 模式，重启设备时，需要将 ip<sub>vs</sub>\*.ko 设置为启动自动加载 ip<sub>vs</sub>\*.ko,否则 kube-dns 会一直报错 `dns.go:174] Waiting for services and endpoints to be initialized from apiserver`,解决办法：在目录 `/etc/sysconfig/modules/` 下创建 ipvs.modules 文件，将需要加载的 ko 写入，如：

```sh
#!/bin/bash
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4

```

重启设备后，会自动加载 ip<sub>vs</sub>\*.ko 文件，解决 kube-dns 报错问题
