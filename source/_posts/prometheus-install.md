---
title: Docker安装Prometheus和Grafana
categories: [Metrics]
tags: [Prometheus, Metrics]
author: Mingshan
date: 2020-07-17
---

最近学习HikariCP数据库连接池，从这本书[《HikariCP数据库连接池实战》](https://weread.qq.com/web/reader/c6d32170718f6391c6d5618kc81322c012c81e728d9d180)看到数据库连接池的监控部分，用到了Prometheus和Grafana，这里记录一下安装与配置过程（利用docker安装），下一篇文章详细描写利用Prometheus进行SpringBoot应用自定义埋点监控与HikariCP数据库连接池监控。

<!-- more -->

# 拉取镜像

```
docker pull prom/node-exporter
docker pull prom/prometheus
docker pull grafana/grafana
```

# 启动node-exporter

```
docker run -d -p 9100:9100 \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/rootfs:ro" \
  --net="host" \
  prom/node-exporter
```

访问：

http://172.17.10.235:9100/metrics

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 7
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.14.4"} 1
# HELP go_memstats_alloc_bytes Number of bytes allocated and still in use.
# TYPE go_memstats_alloc_bytes gauge
go_memstats_alloc_bytes 1.268352e+06
```

# 启动prometheus

新建目录，生成prometheus.yml

```
mkdir /opt/metrics/prometheus
cd /opt/metrics/prometheus
vim prometheus.yml
```


```
global:
  # 默认情况下，每15s拉取一次目标采样点数据。
  scrape_interval:     15s 
  # 我们可以附加一些指定标签到采样点度量标签列表中, 用于和第三方系统进行通信, 包括：federation, remote storage, Alertmanager
  external_labels:
    # 下面就是拉取自身服务采样点数据配置
    monitor: 'codelab-monitor'
scrape_configs:
  # job名称会增加到拉取到的所有采样点上，同时还有一个instance目标服务的host：port标签也会增加到采样点上
  - job_name: prometheus
    # 覆盖global的采样点，拉取时间间隔5s
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
 
  - job_name: linux
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: localhost
          
  - job_name: metrics-demo
    # 覆盖global的采样点，拉取时间间隔5s
    scrape_interval: 5s
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['172.17.12.91:8080']
```



```
docker run  -d \
  -p 9090:9090 \
  -v /opt/metrics/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml  \
  prom/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
```

访问：

http://172.17.10.235:9090/metrics

http://172.17.10.235:9090/graph


# 启动Grafana


```
mkdir /opt/metrics/grafana-storage
chmod 777 -R /opt/metrics/grafana-storage
```

```
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  -v /opt/metrics/grafana-storage:/var/lib/grafana \
  grafana/grafana
```

访问：

http://172.17.10.235:3000/，默认用户名和密码都是admin

# 参考：

- https://blog.csdn.net/kongmingxiaoxiao/article/details/103827722
- https://prometheus.io/docs/prometheus/latest/configuration/configuration
