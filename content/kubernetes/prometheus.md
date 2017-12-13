---
title: "Prometheus 实践"
subtitle: "Prometheus 监控相关"
date: 2017-12-13T11:13:30+08:00
tags: ["kubernetes","prometheus"]
type: "post"
categories: ["kubernetes","prometheus"]
description: "prometheus 监控相关"
---

- [prometheus 实践](#org4647fbc)
  - [节点连接数](#orgb2623ed)


<a id="org4647fbc"></a>

# prometheus 实践


<a id="orgb2623ed"></a>

## 节点连接数

在 netfilter 开启的情况下，部署 `Node exporter`, 见[官方网站](https://github.com/prometheus/node_exporter) 的列表，其 中 `node_nf_conntrack_entries` 记录当前设备上的连接数。这里汇总的这个设备上 所有状态的连接跟踪情况，包括（close<sub>wait</sub>，established，listen，time<sub>wait</sub>）等状态， 最新的 node<sub>exporter</sub> 可通过参数 -collector.tcpstat 收集不同状态的连接数情况，开启 tcpstat 后，可以看到 node<sub>exporter</sub> 的 metrics 中会有以下信息：

```sh
# HELP node_tcp_connection_states Number of connection states.
# TYPE node_tcp_connection_states gauge
node_tcp_connection_states{state="close_wait"} 24
node_tcp_connection_states{state="established"} 348
node_tcp_connection_states{state="listen"} 29
node_tcp_connection_states{state="time_wait"} 76
```

**注意：node<sub>exporter</sub> 在 0.15.2 以后开始支持参数 `collector.tcpstat`**
