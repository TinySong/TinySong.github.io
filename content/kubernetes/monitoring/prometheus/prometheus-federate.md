---
title: "Prometheus 联邦集群"
subtitle: "Prometheus 联邦集群"
date: 2017-10-13T11:13:30+08:00
tags: ["kubernetes","prometheus"]
type: "post"
categories: ["kubernetes","prometheus"]
description: "prometheus 联邦集群"
---

- [prometheus 联邦集群](#orga5cc21a)
  - [介绍](#org527d756)
    - [主要适用场景](#org77a28cf)
  - [操作](#org9a860d3)
  - [配置](#org5b20391)
    - [promethus federation](#orgba78942)
    - [自定义服务 metrics,主要设置三点，metric<sub>path</sub>, source<sub>labels</sub>, 和 regex；](#org2d8d391)
  - [注意点](#orgc2439da)
  - [扩展阅读](#orgcd8de1c)
  - [监测第三方的 metrics](#orgfc6c3f5)


<a id="orga5cc21a"></a>

# prometheus 联邦集群


<a id="org527d756"></a>

## 介绍

prometheus 最新支持联邦集群功能，但联邦集群的监控方式不同于 mysql 或 es 等集群的一主多备。见官方介绍： **Use case: There are different use cases for federation. Commonly, it is used to either achieve scalable Prometheus monitoring setups or to pull related metrics from one service's Prometheus into another.** 测试地址：

```sh
federation prometheus: http://192.168.1.11:30901, slave prometheus: http://192.168.1.11:30900
```

-   slave prometheus: 根据 configmap 中的配置抓取 node, api-server, sevice 等数据
-   federation prometheus: 从 slave prometheus 拉取数据


<a id="org77a28cf"></a>

### 主要适用场景

一个 datacenter 一个 prometheus,多个 datacenter 通过 global prometheus 联邦聚合所有 datacenter 中 prometheus 收集 到的数据，主要体现的功能点是聚合


<a id="org9a860d3"></a>

## 操作

可通过一下命令查看当前 prometheus 监控了哪些 target：

```sh
curl http://localhost:9090/api/v1/targets
```


<a id="org5b20391"></a>

## 配置


<a id="orgba78942"></a>

### promethus federation

1.  configmap

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      namespace: kube-system
      name: prometheus-config-federate
    data:
      prometheus.yaml: |
        scrape_configs:
        - job_name: 'prom-federation'  # job name
          honor_labels: true
          metrics_path: /federate   # scrape url path
          params:
            match[]:
              - '{job=~".+"}'   # Request all job-level time series
          static_configs:
            - targets:
                - 'prometheus:9090'  # 抓取的 service 地址
    ```

2.  federation deployment yaml

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      labels:
        name: prometheus-deployment
        plugin: prometheus-federate
        app: prometheus-federate
      name: prometheus-federate
      namespace: kube-system
    spec:
      replicas: 1
      selector:
        matchLabels:
          plugin: prometheus-federate
      template:
        metadata:
          labels:
            plugin: prometheus-federate
        spec:
          containers:
          - image: 192.168.1.52/tenx_containers/prometheus:v1.6.1
            resources:
              requests:
                cpu: 200m
                memory: 300Mi
              limits:
                cpu: 200m
                memory: 300Mi
            name: prometheus-federate
            command:
            - "/bin/prometheus"
            args:
            - "-config.file=/etc/prometheus/prometheus.yaml"
            - "-storage.local.path=/prometheus"
            - "-storage.local.retention=24h"
            - "-alertmanager.url=http://alertmanager:9093"
            ports:
            - containerPort: 9090
              protocol: TCP
            volumeMounts:
            - mountPath: "/etc/prometheus"
              name: config-volume
            - name: tenxcloud-time-localtime
              readOnly: true
              mountPath: "/etc/localtime"
            - name: tenxcloud-time-zone
              readOnly: true
              mountPath: "/etc/timezone"
          - image: 192.168.1.52/tenx_containers/inotify:1.0
            resources:
              requests:
                cpu: 200m
                memory: 200Mi
              limits:
                cpu: 200m
                memory: 200Mi
            name: notify
            args:
            - 'while inotifywait -qq -e modify,create,delete /etc/prometheus/;
              do sh -c  "curl -X POST http://localhost:9090/-/reload"; done; '
          volumes:
          - emptyDir: {}
            name: data
          - configMap:
              name: prometheus-config-federate
            name: config-volume
          - name: tenxcloud-time-localtime
            hostPath:
              path: "/etc/localtime"
          - name: tenxcloud-time-zone
            hostPath:
              path: "/etc/timezone"
    
    ```

3.  federation service config

    ```yaml
    kind: Service
    apiVersion: v1
    metadata:
      name: prometheus-federate
      namespace: kube-system
      labels:
        plugin: prometheus-federate
    spec:
      type: NodePort
      ports:
      - name: prometheus-federate
        protocol: TCP
        port: 9090
        nodePort: 30901
      selector:
        plugin: prometheus-federate
    ```


<a id="org2d8d391"></a>

### 自定义服务 metrics,主要设置三点，metric<sub>path</sub>, source<sub>labels</sub>, 和 regex；

```yaml
- job_name: 'kubernetes-services'
  kubernetes_sd_configs:
    - role: service
      relabel_configs:
        metrics_path: /probe   # 自定义 metrics path
        - source_labels: [__meta_kubernetes_service_name]  # 要设置成__meta_kubernetes_service_name
          action: keep
          regex: serviceName                # service Name

```


<a id="orgc2439da"></a>

## 注意点

-   <regex> is any valid RE2 regular expression. It is required for the **replace, keep, drop, labelmap,labeldrop and labelkeep actions**. The regex is anchored on both ends. To un-anchor the regex, use .\*<regex>.\*. Federation is intended for aggregated stats, not pulling the content of entire Prometheus servers.
-   kubernetes<sub>sd</sub><sub>config</sub>: <https://prometheus.io/docs/operating/configuration/#kubernetes_sd_config>
-   relabel<sub>config</sub>: <https://prometheus.io/docs/operating/configuration/#relabel_config>

Prometheus will use metrics provided by cAdvisor via kubelet service (runs on each node of Kubernetes cluster by default) and via kube-apiserver service only.


<a id="orgcd8de1c"></a>

## 扩展阅读

-   KubeCon 2017 - Prometheus Takeaways <https://pracucci.com/kubecon-2017-prometheus-takeaways.html>
-   Federation | Prometheus <https://prometheus.io/docs/operating/federation/>
-   Prometheus 实战于源码分析之 API 与联邦 <http://blog.csdn.net/u010278923/article/details/70891379>
-   "Prometheus and Kubernetes - Movio blog | Movio Blog" <https://movio.co/en/blog/prometheus-service-discovery-kubernetes/>
-   "Prometheus and Kubernetes up and running | CoreOS" <https://coreos.com/blog/prometheus-and-kubernetes-up-and-running.html>
-   Configuration | Prometheus <https://prometheus.io/docs/operating/configuration/#relabel_config>


<a id="orgfc6c3f5"></a>

## 监测第三方的 metrics

-   how to instrument your code to expose metrics to prometheus <http://marselester.com/prometheus-on-kubernetes.html>
