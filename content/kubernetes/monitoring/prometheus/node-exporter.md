---
title: "Node Exporter"
subtitle: "Node Exporter"
date: 2017-12-15T14:33:39+08:00
tags: ["node_exporter", "kuberntes", "prometheus"]
type: "post"
categories: ["node_exporter", "kuberntes", "prometheus"]
description: "node_exporter 实践"
---

- [cpu 监控](#org208b27f)
  - [扩展阅读](#org292d01b)
- [memory 监控](#org19f93d7)


node<sub>exporter</sub> 作为 prometheus 的监控插件，主要用于监控节点信息，如节点的 cpu，网络， 等等，详情可见[github 官网](https://github.com/prometheus/node_exporter)


<a id="org208b27f"></a>

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


<a id="org292d01b"></a>

## 扩展阅读

-   Understanding Linux CPU stats

<http://blog.scoutapp.com/articles/2015/02/24/understanding-linuxs-cpu-stats>


<a id="org19f93d7"></a>

# memory 监控

node<sub>exporter</sub> 通过读取 `/proc/meminfo` 中的信息获取内存的相关指标， node<sub>exporter对外暴露的指标如下</sub>：

```sh
# HELP node_memory_Active Memory information field Active.
# TYPE node_memory_Active gauge
node_memory_Active 1.742860288e+09
# HELP node_memory_Active_anon Memory information field Active_anon.
# TYPE node_memory_Active_anon gauge
node_memory_Active_anon 7.31860992e+08
# HELP node_memory_Active_file Memory information field Active_file.
# TYPE node_memory_Active_file gauge
node_memory_Active_file 1.010999296e+09
# HELP node_memory_AnonHugePages Memory information field AnonHugePages.
# TYPE node_memory_AnonHugePages gauge
node_memory_AnonHugePages 3.73293056e+08
# HELP node_memory_AnonPages Memory information field AnonPages.
# TYPE node_memory_AnonPages gauge
node_memory_AnonPages 1.292787712e+09
# HELP node_memory_Bounce Memory information field Bounce.
# TYPE node_memory_Bounce gauge
node_memory_Bounce 0
# HELP node_memory_Buffers Memory information field Buffers.
# TYPE node_memory_Buffers gauge
node_memory_Buffers 585728
# HELP node_memory_Cached Memory information field Cached.
# TYPE node_memory_Cached gauge
node_memory_Cached 2.308968448e+09
# HELP node_memory_CommitLimit Memory information field CommitLimit.
# TYPE node_memory_CommitLimit gauge
node_memory_CommitLimit 4.21963776e+09
# HELP node_memory_Committed_AS Memory information field Committed_AS.
# TYPE node_memory_Committed_AS gauge
node_memory_Committed_AS 5.110943744e+09
# HELP node_memory_DirectMap1G Memory information field DirectMap1G.
# TYPE node_memory_DirectMap1G gauge
node_memory_DirectMap1G 2.147483648e+09
# HELP node_memory_DirectMap2M Memory information field DirectMap2M.
# TYPE node_memory_DirectMap2M gauge
node_memory_DirectMap2M 4.181721088e+09
# HELP node_memory_DirectMap4k Memory information field DirectMap4k.
# TYPE node_memory_DirectMap4k gauge
node_memory_DirectMap4k 1.13180672e+08
# HELP node_memory_Dirty Memory information field Dirty.
# TYPE node_memory_Dirty gauge
node_memory_Dirty 32768
# HELP node_memory_HardwareCorrupted Memory information field HardwareCorrupted.
# TYPE node_memory_HardwareCorrupted gauge
node_memory_HardwareCorrupted 0
# HELP node_memory_HugePages_Free Memory information field HugePages_Free.
# TYPE node_memory_HugePages_Free gauge
node_memory_HugePages_Free 0
# HELP node_memory_HugePages_Rsvd Memory information field HugePages_Rsvd.
# TYPE node_memory_HugePages_Rsvd gauge
node_memory_HugePages_Rsvd 0
# HELP node_memory_HugePages_Surp Memory information field HugePages_Surp.
# TYPE node_memory_HugePages_Surp gauge
node_memory_HugePages_Surp 0
# HELP node_memory_HugePages_Total Memory information field HugePages_Total.
# TYPE node_memory_HugePages_Total gauge
node_memory_HugePages_Total 0
# HELP node_memory_Hugepagesize Memory information field Hugepagesize.
# TYPE node_memory_Hugepagesize gauge
node_memory_Hugepagesize 2.097152e+06
# HELP node_memory_Inactive Memory information field Inactive.
# TYPE node_memory_Inactive gauge
node_memory_Inactive 1.885036544e+09
# HELP node_memory_Inactive_anon Memory information field Inactive_anon.
# TYPE node_memory_Inactive_anon gauge
node_memory_Inactive_anon 8.16078848e+08
# HELP node_memory_Inactive_file Memory information field Inactive_file.
# TYPE node_memory_Inactive_file gauge
node_memory_Inactive_file 1.068957696e+09
# HELP node_memory_KernelStack Memory information field KernelStack.
# TYPE node_memory_KernelStack gauge
node_memory_KernelStack 1.6941056e+07
# HELP node_memory_Mapped Memory information field Mapped.
# TYPE node_memory_Mapped gauge
node_memory_Mapped 1.9529728e+08
# HELP node_memory_MemAvailable Memory information field MemAvailable.
# TYPE node_memory_MemAvailable gauge
node_memory_MemAvailable 2.10018304e+09
# HELP node_memory_MemFree Memory information field MemFree.
# TYPE node_memory_MemFree gauge
node_memory_MemFree 1.58670848e+08
# HELP node_memory_MemTotal Memory information field MemTotal.
# TYPE node_memory_MemTotal gauge
node_memory_MemTotal 4.144316416e+09
# HELP node_memory_Mlocked Memory information field Mlocked.
# TYPE node_memory_Mlocked gauge
node_memory_Mlocked 0
# HELP node_memory_NFS_Unstable Memory information field NFS_Unstable.
# TYPE node_memory_NFS_Unstable gauge
node_memory_NFS_Unstable 0
# HELP node_memory_PageTables Memory information field PageTables.
# TYPE node_memory_PageTables gauge
node_memory_PageTables 1.5347712e+07
# HELP node_memory_SReclaimable Memory information field SReclaimable.
# TYPE node_memory_SReclaimable gauge
node_memory_SReclaimable 1.43306752e+08
# HELP node_memory_SUnreclaim Memory information field SUnreclaim.
# TYPE node_memory_SUnreclaim gauge
node_memory_SUnreclaim 7.0254592e+07
# HELP node_memory_Shmem Memory information field Shmem.
# TYPE node_memory_Shmem gauge
node_memory_Shmem 2.29597184e+08
# HELP node_memory_Slab Memory information field Slab.
# TYPE node_memory_Slab gauge
node_memory_Slab 2.13561344e+08
# HELP node_memory_SwapCached Memory information field SwapCached.
# TYPE node_memory_SwapCached gauge
node_memory_SwapCached 2.6058752e+07
# HELP node_memory_SwapFree Memory information field SwapFree.
# TYPE node_memory_SwapFree gauge
node_memory_SwapFree 2.013597696e+09
# HELP node_memory_SwapTotal Memory information field SwapTotal.
# TYPE node_memory_SwapTotal gauge
node_memory_SwapTotal 2.147479552e+09
# HELP node_memory_Unevictable Memory information field Unevictable.
# TYPE node_memory_Unevictable gauge
node_memory_Unevictable 0
# HELP node_memory_VmallocChunk Memory information field VmallocChunk.
# TYPE node_memory_VmallocChunk gauge
node_memory_VmallocChunk 3.5183965237248e+13
# HELP node_memory_VmallocTotal Memory information field VmallocTotal.
# TYPE node_memory_VmallocTotal gauge
node_memory_VmallocTotal 3.5184372087808e+13
# HELP node_memory_VmallocUsed Memory information field VmallocUsed.
# TYPE node_memory_VmallocUsed gauge
node_memory_VmallocUsed 2.002944e+08
# HELP node_memory_Writeback Memory information field Writeback.
# TYPE node_memory_Writeback gauge
node_memory_Writeback 0
# HELP node_memory_WritebackTmp Memory information field WritebackTmp.
# TYPE node_memory_WritebackTmp gauge
node_memory_WritebackTmp 0
```

指标是相当的完善，基本把内存的所有信息都已经暴露出来，普通用户主要关心 `node_memory_MemFree` 和 `node_memory_MemTotal`,可通过prometheus提供的计算方 法可计算出当前主机上内存使用情况,如：

```sh
(node_memory_MemTotal{instance="node-4"} -node_memory_MemFree{instance="node-4"})/1024/1024
```

即可计算出node-4节点的目前主机使用的内存数（单位M）
