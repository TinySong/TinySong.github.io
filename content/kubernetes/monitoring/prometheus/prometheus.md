---
title: "Prometheus 实践"
subtitle: "Prometheus 监控相关"
date: 2017-12-13T11:13:30+08:00
tags: ["kubernetes","prometheus"]
type: "post"
categories: ["kubernetes","prometheus"]
description: "prometheus 监控相关"
---

- [prometheus 实践](#org1b4db6a)
  - [prometheus 高可用](#orgc487cb3)
    - [deployment](#org358beed)
    - [daemonSet 方式](#orgea0b598)
  - [支持 Node 上的磁盘 IO，网络吞吐等监控](#org0ffd271)
  - [tcp 连接数监控](#orge8c74a8)
  - [调优](#org9b324be)
    - [内存调优](#org9b835a4)
    - [块编码调优](#org45285c6)
  - [prometheus 查询](#orgb6d951f)
    - [metric type](#org1b0ed0a)
    - [job and instance](#org8c81b03)
    - [operator](#orgd8333b6)
  - [常见问题以及解决方式](#org1f5cc2a)
    - [prometheus container cannot get fs](#org16dd685)
  - [prometheus other exporter](#org729915f)



<a id="org1b4db6a"></a>

# prometheus 实践


<a id="orgc487cb3"></a>

## prometheus 高可用

可通过 deployment 和 DaementSet 方式实现高可用


<a id="org358beed"></a>

### deployment

1.  kubectl create -f prometheus-configmap.yaml
2.  kubectl create -f prometheus-config-rule.yaml
3.  kubectl create -f prometheus-deploy.yaml #其中 spec->replicas > 1
4.  kubectl create -f prometheus-service.yaml # service.spec.sessionAffinity 设置为 “ClientIP”

执行以上操作后，通过访问 <http://masterIP:30900／targets> 进入 targets 监控见面，查看 target 的 State 是否为 UP 状态，假如都为 UP，这时即可到 prometheus 总览界面查询所需要的数据。


<a id="orgea0b598"></a>

### daemonSet 方式

1.  kubectl create -f prometheus-configmap.yaml
2.  kubectl create -f prometheus-config-rule.yaml
3.  kubectl create -f prometheus-daemonset.yaml
4.  kubectl create -f prometheus-service.yaml # service.spec.sessionAffinity 设置为 "ClientIP"

执行以上操作后，通过访问 <http://masterIP:30900／targets> 进入 targets 监控见面，查看 target 的 State 是否为 UP 状态，假如都为 UP，这时即可到 prometheus 总览界面查询所需要的数据。


<a id="org0ffd271"></a>

## 支持 Node 上的磁盘 IO，网络吞吐等监控

-   目前默认支持的功能：<https://github.com/prometheus/node_exporter#enabled-by-default>
-   默认 diable 的功能：<https://github.com/prometheus/node_exporter#disabled-by-default>
-   node-exporter 使用 daemonSet 方式启动，配置文件见附录
-   kubectl create -f ./prometheus-node-exporter.yaml node-exporter 主要是通过在 pod 中添加 **"prometheus.io/scrape:"true"**,然后 configmap job<sub>name</sub> 为 kubernetes-pods 中，设置如下配置，prometheus 则会从凡是设置了"prometheus.io/scrape:"true 的 pod 都会 scrape 数据
    
    ```yaml
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    
    ```
    
    -   node-exporter 的配置
        
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          annotations:
            prometheus.io/scrape: "true"
          labels:
            app: node-exporter
            name: node-exporter
          name: node-exporter
          namespace: kube-system
        spec:
          clusterIP: None
          ports:
          - name: scrape
            port: 9100
            protocol: TCP
          selector:
            app: node-exporter
          type: ClusterIP
        #----
        apiVersion: extensions/v1beta1
        kind: DaemonSet
        metadata:
          name: node-exporter
          namespace: kube-system
        spec:
          template:
            metadata:
              labels:
                app: node-exporter
              name: node-exporter
              annotations:
                prometheus.io/scrape: "true"
            spec:
              containers:
              - image: 192.168.1.12/tenx_containers/node-exporter:v0.14.0
                name: node-exporter
                securityContext:
                  privileged: true
                ports:
                - containerPort: 9100
                  hostPort: 9100
                  name: scrape
              hostNetwork: true
              hostPID: true
        ```


<a id="orge8c74a8"></a>

## tcp 连接数监控

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


<a id="org9b324be"></a>

## 调优


<a id="org9b835a4"></a>

### 内存调优

<https://prometheus.io/docs/operating/storage/> v1.6 以后 Prometheus keeps all the currently used chunks in memory. In addition, it keeps as many most recently used chunks in memory as possible. You have to tell Prometheus how much memory it may use for this caching. The flag storage.local.target-heap-size allows you to set the heap size (in bytes) Prometheus aims not to exceed. As a rule of thumb, you should have at least 50% headroom in physical memory over the configured heap size. (Or, in other words, set storage.local.target-heap-size to a value of two thirds of the physical memory limit Prometheus should not exceed.)

使用 storage.local.target-heap-size 进行内存设置，如果一个节点专门用来运行 Prometheus，设置为总的物理内存的 2/3，如果有其他组件，适当估计其他组件损耗，使用剩余内存的 2/3 即可。 1.6 之前的版本参考以上文档


<a id="org45285c6"></a>

### 块编码调优

Prometheus currently offers three different types of chunk encodings. The chunk encoding for newly created chunks is determined by the -storage.local.chunk-encoding-version flag. The valid values are 0, 1, or 2. Type 0 is the simple delta encoding implemented for Prometheus's first chunked storage layer. Type 1 is the current default encoding, a double-delta encoding with much better compression behavior than type 0.

<https://prometheus.io/blog/2016/05/08/when-to-use-varbit-chunks/Type> 2 有更好的压缩比，但是会需要更多的 CPU 资源，IO 瓶颈时可以考虑使用 Type 2 Three times more samples in RAM, three times more samples on disk, only a third of disk ops, and since disk ops are currently the bottleneck for ingestion speed, it will also allow ingestion to be three times faster. In fact, the recently reported new ingestion record of 800,000 samples per second was only possible with varbit chunks – and with an SSD, obviously. With spinning disks, the bottleneck is reached far earlier, and thus the 3x gain matters even more.

1.  如果碰到 prometheus 的 apiserver endpoint UnKnown

增加 prometheus 内存再试试 其他调优、调试参数参考：<https://prometheus.io/docs/operating/storage/> 最下方


<a id="orgb6d951f"></a>

## prometheus 查询


<a id="org1b0ed0a"></a>

### metric type

-   Counter: 用于累计计数，例如用来记录请求次数。Counter 的特点是一直增加不会减少。
-   Gauge：用于记录常规数值，可以增加或减少。例如用来记录 CPU、内存的变化
-   Histogram：可理解为直方图，常用于跟踪事件发生的规模，如请求耗时、响应大小。可对记录的内容分组和聚合(count,sum 等)，例如响应时间小于 500 毫秒的多少次、500 毫秒~1000 毫秒之间多少次、1000 毫秒以上的多少次
-   Summary：与 Histogram 类似，但支持按百分比跟踪结果


<a id="org8c81b03"></a>

### job and instance

在 Prometheus 中任何被采集的目标 Target 被称为 Instance，通常对应单个进程。 相同类型的 Instance 被称为 Job，例如：

```yaml
- job: api-server
 - instance 1: 1.2.3.4:5670
 - instance 2: 1.2.3.4:5671
 - instance 3: 5.6.7.8:5670
 - instance 4: 5.6.7.8:5671
```

Prometheus 在从目标采集数据时会自动附加一些标签，用于识别被采集的目标：

-   job：配置的 job 名称
-   instance：<host>:<port> 或者 domain<sub>name</sub>


<a id="orgd8333b6"></a>

### operator

-   rate: method<sub>code</sub>:http<sub>errors</sub>:rate5m{code="500"} This returns a result vector containing the fraction of HTTP requests with status code of 500 for each method, as measured over the last 5 minutes.


<a id="org1f5cc2a"></a>

## 常见问题以及解决方式


<a id="org16dd685"></a>

### prometheus container cannot get fs

1.  "Report container FS metrics into prometheus /metrics by smarterclayton · Pull Request #1642 · google/cadvisor"

    <https://github.com/google/cadvisor/pull/1642>

2.  "no fs stats per container · Issue #1403 · google/cadvisor"

    <https://github.com/google/cadvisor/issues/1403>

3.  "/metrics container<sub>fs</sub><sub>reads</sub><sub>total</sub> is 0 · Issue #1669 · google/cadvisor"

    <https://github.com/google/cadvisor/issues/1669>


<a id="org729915f"></a>

## prometheus other exporter

-   Node/system metrics exporter
-   AWS CloudWatch exporter
-   Blackbox exporter
-   Collectd exporter
-   Consul exporter
-   Graphite exporter
-   HAProxy exporter
-   InfluxDB exporter
-   JMX exporter
-   Memcached exporter
-   Mesos task exporter
-   MySQL server exporter
-   SNMP exporter
-   StatsD exporter
