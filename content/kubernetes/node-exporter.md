---
title: "Node Exporter"
subtitle: "Node Exporter"
date: 2017-12-15T14:33:39+08:00
tags: ["node_exporter", "kuberntes", "prometheus"]
type: "post"
categories: ["node_exporter", "kuberntes", "prometheus"]
description: "node_exporter 实践"
---

- [cpu 监控](#orgc3ed17b)

node<sub>exporter</sub> 作为 prometheus 的监控插件，主要用于监控节点信息，如节点的 cpu，网络， 等等，详情可见[github 官网](https://github.com/prometheus/node_exporter)


<a id="orgc3ed17b"></a>

# cpu 监控

通过 node<sub>exporter</sub> 提供的 node<sub>cpu</sub> 这个 metrics 提供对 cpu 的监控，主要用于查看 linux 下的 `/proc/stat` 文件获取 cpu 的每个核在各个 mode 下所使用的秒数，mode 类型包 含如下：

-   user: The time spent in userland
-   system: The time spent in the kernel
-   iowait: Time spent waiting for I/O
-   idle: Time the CPU had nothing to do
-   irq&softirq: Time servicing interrupts
-   guest: If you are running VMs, the CPU they use
-   steal: If you are a VM, time other VMs “stole” from your CPUs

node<sub>exporter</sub> 提供的 metric 信息如下所示：

```sh
# HELP node_cpu Seconds the cpus spent in each mode.
# TYPE node_cpu counter
node_cpu{cpu="cpu0",mode="guest"} 0
node_cpu{cpu="cpu0",mode="guest_nice"} 0
node_cpu{cpu="cpu0",mode="idle"} 1.16190813e+06
node_cpu{cpu="cpu0",mode="iowait"} 7097.81
node_cpu{cpu="cpu0",mode="irq"} 0
node_cpu{cpu="cpu0",mode="nice"} 392.46
node_cpu{cpu="cpu0",mode="softirq"} 3133.33
node_cpu{cpu="cpu0",mode="steal"} 0
node_cpu{cpu="cpu0",mode="system"} 102660.99
node_cpu{cpu="cpu0",mode="user"} 221165.62
node_cpu{cpu="cpu1",mode="guest"} 0
node_cpu{cpu="cpu1",mode="guest_nice"} 0
node_cpu{cpu="cpu1",mode="idle"} 1.14791184e+06
node_cpu{cpu="cpu1",mode="iowait"} 6781.02
node_cpu{cpu="cpu1",mode="irq"} 0
node_cpu{cpu="cpu1",mode="nice"} 385.58
node_cpu{cpu="cpu1",mode="softirq"} 11671.78
node_cpu{cpu="cpu1",mode="steal"} 0
node_cpu{cpu="cpu1",mode="system"} 101436.52
node_cpu{cpu="cpu1",mode="user"} 221986.68
```

对于用户来说主要关系的应该是除 idle 的其他 mode cpu 占用率，可通过 prometheus 提供的 聚合方法计算出 cpu 的占用率，公式如下：

,#+BEGIN<sub>SRC</sub> sh 100 - (avg by (instance) (irate(node<sub>cpu</sub>{instance="node-1",mode="idle"}[5m])) \* 100) \\#+END<sub>SRC</sub> 这样就可以计算出节点 `node-1` 的 cpu 占用率
