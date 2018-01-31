- [prometheus v1.6.x 升级到 v2.1 注意事项](#org5c1cb98)
  - [规则格式](#org740fffe)
  - [参数](#org7c209a8)
  - [迁移时，可参考官方迁移文档](#org33ce1dc)

&#x2014; title: "Prometheus2.0" subtitle: "Prometheus2.0" date: 2017-12-15T17:03:56+08:00 tags: [""] type: "post" categories: [""] description: "" &#x2014;


<a id="org5c1cb98"></a>

# prometheus v1.6.x 升级到 v2.1 注意事项

升级 prometheus2.0 时，会存在一些不兼容情况，具体如下


<a id="org740fffe"></a>

## 规则格式

prometheus 告警规则自定义格式迁移到 yaml 格式，当升级时 可通过 promtool 工具升级告警规则的格式

```sh
./promtool update rules /etc/prometheus_rules/*
```

转换完后的 yaml 格式的告警规则文件为 alert.rule.yml,通过脚本将起生成 configmap 文件

```sh
#!/bin/bash
# convert many rule file into configmap
cat <<-EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: kube-system
data:
EOF

for f in /etc/prometheus_rules/*.rule.y*ml
do
    suffix=.yml
    filename=$(basename "$f")
    echo "  ${filename%$suffix}: |+"
    cat $f | sed "s/^/    /g"
done
```


<a id="org7c209a8"></a>

## 参数

从 2.0 开始，prometheus 的命令参数必须以 `--` 开始，不支持 `-` 。 `-storage.local.*` 和 `-storage.remote.*` 参数丢弃， 本地存储参数改为以下格式：

```sh
- --storage.tsdb.path=/prometheus/data
- --storage.tsdb.retention=24h
```

自动加载规则文件默认不开启，需要通过参数 `--web.enable-lifecycle` 开启

如果需要保留历史的监控数据的话，需要启动一个 1.8.1 版本的 Prometheus 实例，该实例 不要去拉取监控数据，而只作为 2.0 版本的 remote data，在 2.0 的实例中配置:

```yaml
remote_read:
- url: "http://localhost:9094/api/v1/read"
```

`-alertmanager.url` 参数不在支持，可通过在 prometheus-config.yml 中添加 alertmanager 服务地址

```yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager:9093
```

或者也可以通过服务发现机制实现


<a id="org33ce1dc"></a>

## 迁移时，可参考官方迁移文档

<https://prometheus.io/docs/prometheus/2.0/migration/>
