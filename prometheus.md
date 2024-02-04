<!-- vim-markdown-toc GFM -->

* [Prometheus](#prometheus)
    * [exporter](#exporter)
        * [systemd](#systemd)
        * [node_exporter监控主机信息](#node_exporter监控主机信息)
        * [redis](#redis)
        * [mongodb](#mongodb)
        * [mysql](#mysql)
        * [nginx](#nginx)
            * [nginx-prometheus-exporter](#nginx-prometheus-exporter)
            * [nginx-vts-exporter](#nginx-vts-exporter)
        * [nvidia](#nvidia)
        * [github](#github)
        * [android](#android)
        * [cadvisor：docker监控](#cadvisordocker监控)
        * [blackbox_exporter](#blackbox_exporter)
            * [prometheus集成blackbox_exporter](#prometheus集成blackbox_exporter)
    * [PromQL](#promql)
        * [各种方法](#各种方法)
        * [metrics（指标）类型](#metrics指标类型)
        * [各种操作符](#各种操作符)
            * [通过curl在HTTP中使用PromQL](#通过curl在http中使用promql)
        * [Recoding Rules](#recoding-rules)
    * [服务发现](#服务发现)
        * [基于文件的服务发现](#基于文件的服务发现)
        * [consul](#consul)
            * [服务注册](#服务注册)
            * [集成prometheus](#集成prometheus)
        * [基于dns服务发现](#基于dns服务发现)
        * [服务发现与relabel_configs](#服务发现与relabel_configs)
    * [可视化](#可视化)
        * [Console Template](#console-template)
        * [Grafana](#grafana)
            * [dashboard](#dashboard)
    * [alertmanager告警](#alertmanager告警)
        * [告警](#告警)
        * [alertmanager配置文件](#alertmanager配置文件)
            * [route（路由规则）](#route路由规则)
            * [receivers（告警接收器）](#receivers告警接收器)
                * [PrometheusAlert：可以接入alertmanager的国内告警系统](#prometheusalert可以接入alertmanager的国内告警系统)
                * [gmail邮箱](#gmail邮箱)
                * [企业微信（失败）](#企业微信失败)
                * [钉钉dingtalk](#钉钉dingtalk)
                * [Slack](#slack)
            * [inhibit_rules](#inhibit_rules)
            * [silences（临时静默）](#silences临时静默)
            * [告警模板](#告警模板)
    * [集群](#集群)
        * [联邦集群](#联邦集群)
        * [通过goreman实现集群](#通过goreman实现集群)
            * [alertmanager](#alertmanager)
            * [prometheus](#prometheus-1)
    * [日志监控系统](#日志监控系统)
        * [mtail](#mtail)
    * [存储](#存储)
        * [远程存储](#远程存储)
    * [prophet：时序数据预测的 Python 库](#prophet时序数据预测的-python-库)
* [reference](#reference)

<!-- vim-markdown-toc -->

# Prometheus

- [Monitoring Linux Network Stack](http://arthurchiao.art/blog/monitoring-network-stack/)

    - 使用简单的shell脚本，配合prometheus监控网络

- [官方exporter的下载页面](https://prometheus.io/download/)

- 监控数据保留时间：Prometheus专注于短期监控、告警而设计，所以默认它只保存15天的时间序列数据。如果要更长期，建议考虑数据单独存储到其他平台。

## exporter

- [exporters列表](https://prometheus.io/docs/instrumenting/exporters/)

- 白盒监控：exporter需要安装到目标机器

```sh
# 更新prometheus.yml配置后。可以发送SIGHUP信号，重启prometheus
kill -s SIGHUP $(pidof prometheus)
```

- 一个job监控多个机器：

    - 可以是一个targets监控，多个机器。也可以是多个targets
        ```yml
          - job_name: "node"
            static_configs:
              - targets: ['localhost:9100', '192.168.1.111:9100']
              - targets: ['192.168.1.1:9100']
        ```

    - 如果机器太多，可以用json或yaml
        - 首次加载json和yaml文件需要重启prometheus。之后再修改json和yaml文件无需重启

        ```json
        [
          {
            "targets": [ "localhost:9100","192.168.1.111:9100"],
            "labels": {
              "env": "dev_webgame"
            }
          },
          {
            "targets": [ "192.168.1.1:9100" ],
            "labels": {
              "env": "dev_mgame",
              "job": "mysqld_node"
            }
          }
        ]
        ```

        ```yml
        - targets:
          - "192.168.1.2:9100"
        ```

        - 添加进`prometheus.yml`
            ```yml
              - job_name: "node"
                  - files:
                      - targets/*.json
                      refresh_interval: 5s
                  - files:
                      - targets/*.yaml
                      refresh_interval: 5s
            ```

- 查看exporter是否up`http://127.0.0.1:9090/targets?search=`

### systemd

- 以`nginx_exporter.service`为例

- service文件在本仓库的`./prometheus-config/systemd/`目录下

```sh
# 更新配置
systemctl daemon-reload
# 开机自启动
systemctl enable nginx_exporter.service
# 启动nginx_exporter.service
systemctl start nginx_exporter.service
```

### node_exporter监控主机信息

- `prometheus.yml`文件添加`node_exporter`

```yml
scrape_configs:
  - job_name: "node"
    static_configs:
      - targets: ['localhost:9100']
```

- Panel添加`up`：可以查看组件的上下线情况

- 现在可以添加`node_exporter`的metrics到新的Panel

    - 查看`node_exporter`的metrics：`http://127.0.0.1:9100/metrics`

    - `node_cpu_seconds_total`：所有cpu的占用时间
    - `node_disk_read_time_seconds_total`：所有磁盘的读取时间
    - `node_disk_read_time_seconds_total{device="sda"}`：指定sda
    - `node_network_receive_bytes_total`：网络总共接受的bytes

- 新建一个`rules`文件，监控cpu的5分钟内平均值

    ```yml
    # prometheus.rules.yml
    groups:
    - name: cpu-node
      rules:
      - record: job_instance_mode:node_cpu_seconds:avg_rate5m
        expr: avg by (job, instance, mode) (rate(node_cpu_seconds_total[5m]))
    ```

    - `prometheus.yml`文件添加刚才新建的rules文件
    ```yml
    rule_files:
      - "prometheus.rules.yml"
    ```

### [redis](https://github.com/oliver006/redis_exporter)

- 添加进`prometheus.yml`
```yml
scrape_configs:
  - job_name: "redis_exporter"
    scrape_interval: 10s
    static_configs:
    - targets: ['127.0.0.1:9121']
```

### [mongodb](https://github.com/percona/mongodb_exporter)

```sh
# 启动mongodb_exporter
mongodb_exporter --web.listen-address=:9103 --mongodb.uri=mongodb://127.0.0.1:27017
```

- 添加进`prometheus.yml`
```yml
  - job_name: "mongodb_exporter"
    scrape_interval: 10s
    static_configs:
    - targets: ['127.0.0.1:9103']

```

### [mysql](https://github.com/prometheus/mysqld_exporter)

- root用户运行

```sql
# 创建用户名为mysqld_exporter，密码为12345678
CREATE USER 'mysqld_exporter'@'localhost' IDENTIFIED BY '12345678' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqld_exporter'@'localhost';
FLUSH PRIVILEGES
# 查看是否有mysqld_exporter用户
select host,user from mysql.user
```

- 创建配置文件`mysqld_exporter.cnf`
```ini
[client]
user=mysqld_exporter
password=12345678
```

- 启动mysqld-exporter
```sh
mysqld-exporter --config.my-cnf="./mysqld_exporter.cnf"
```

- 添加进`prometheus.yml`
```yml
  - job_name: "mysql"
    scrape_interval: 10s
    static_configs:
    - targets: ['127.0.0.1:9104']
```

| 监控metrics                                  | 说明         | 对应的sql语句                                       |
|----------------------------------------------|--------------|-----------------------------------------------------|
| mysql_global_status_queries                  | 查询总次数   | SHOW GLOBAL STATUS LIKE "questions"                 |
| mysql_global_status_slow_queries             | 慢查询次数   | SHOW GLOBAL STATUS LIKE "Slow_queries"              |
| mysql_global_status_threads_connected        | 当前连接数   | SHOW GLOBAL STATUS LIKE 'Threads_connected'         |
| mysql_global_status_innodb_buffer_pool_reads | innodb缓冲池 | SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads'; |

### nginx

- [nginx-lua-prometheus](https://github.com/knyar/nginx-lua-prometheus)

#### [nginx-prometheus-exporter](https://github.com/nginxinc/nginx-prometheus-exporter)

- nginx.conf配置：需要开启stub_status

    ```sh
    http {
        server {
            listen       80;
            server_name example.com www.example.com localhost;

            # stub_status模块
            location = /basic_status {
                stub_status;

                # 只允许本机访问
                allow 127.0.0.1;
        	    deny all;
            }
    }
    ```

```sh
# 启动nginx-prometheus-exporter
nginx-prometheus-exporter --nginx.scrape-uri="http://127.0.0.1:80/basic_status"
```

#### [nginx-vts-exporter](https://github.com/hnlq715/nginx-vts-exporter)

- 需要先安装[nginx-module-vts](https://github.com/vozlt/nginx-module-vts)nginx模块
    ```nginx
    http {
        vhost_traffic_status_zone;

        ...

        server {

            ...

            location /status {
                vhost_traffic_status_display;
                vhost_traffic_status_display_format html;
            }
        }
    }
    ```

- 启动`nginx-vtx-exporter`
```sh
nginx-vtx-exporter -nginx.scrape_uri http:localhost/status/format/json
```

- 添加进`prometheus.yml`
```yml
  - job_name: "nginx-vts-exporter"
    scrape_interval: 10s
    static_configs:
    - targets: ['127.0.0.1:9913']
```

### [nvidia](https://github.com/mindprince/nvidia_gpu_prometheus_exporter)

```yml
  - job_name: "nvidia"
    static_configs:
    - targets: ['127.0.0.1:9445']
```

### [github](https://github.com/githubexporter/github-exporter)

```yml
  - job_name: "github"
    static_configs:
    - targets: ['127.0.0.1:9171']
```

### [android](https://github.com/birdthedeveloper/prometheus-android-exporter)

```yml
  - job_name: 'android'
    static_configs:
    - targets: ['192.168.1.111:9100']
      labels:
        machine: 'mi10'
```

### [cadvisor：docker监控](https://github.com/google/cadvisor)

```yml
  # docker
  - job_name: 'cadvisor'
    static_configs:
    - targets: ['127.0.0.1:8080']
```

### blackbox_exporter

- 白盒监控：exporter需要安装到目标机器。
    - 除了`blackbox_exporter`外大多都是白盒监控

- 黑盒监控：无需安装到目标机器。通过http、https、ssh、dns、tcp、icmp等方式进行监控
    - `blackbox_exporter`就是黑盒监控

- 配置文件`blockbox.yml`：在Blackbox Exporter每一个探针配置称为一个module

    - 每一个module包括：探针类型（prober）、验证访问超时时间（timeout）、以及当前探针的具体配置项

    ```yml
      # 探针类型：http、 tcp、 dns、 icmp.
      prober: <prober_string>

      # 超时时间
      [ timeout: <duration> ]

      # 探针的详细配置，最多只能配置其中的一个
      [ http: <http_probe> ]
      [ tcp: <tcp_probe> ]
      [ dns: <dns_probe> ]
      [ icmp: <icmp_probe> ]
    ```

- 以下一个简化的探针配置文件`blockbox.yml`，包含两个HTTP探针配置项：

    ```yml
    modules:
      http_2xx:
        prober: http
        http:
          method: GET
      http_post_2xx:
        prober: http
        http:
          method: POST
    ```

- 实验（不需要启动prometheus就可以使用）

    - 启动`blackbox_exporter`
    ```sh
    prometheus-blackbox-exporter --config.file "blackbox.yml"
    ```

    - 可以通过`http://127.0.0.1:9115/probe?module=http_2xx&target=www.bilibili.com`对www.bilibili.com进行探测。

        - 这里通过在URL中提供`module`参数指定了当前使用的探针，`target`参数指定探测目标，探针的探测结果通过`Metrics`的形式返回：

        - 从返回的样本中，用户可以获取站点的DNS解析耗时、站点响应时间、HTTP响应状态码等等和站点访问质量相关的监控指标，从而帮助管理员主动的发现故障和问题。
            - 滚动到底部如果有`probe_success 0`表示失败

#### prometheus集成blackbox_exporter

- [更多的探针配置](https://yunlzheng.gitbook.io/prometheus-book/part-ii-prometheus-jin-jie/exporter/commonly-eporter-usage/install_blackbox_exporter)

- 1.新建[blackbox.yml](./prometheus-config/blackbox.yml)

- 2.启动`blackbox_exporter`
```sh
prometheus-blackbox-exporter --config.file "blackbox.yml"
```

- 3.自定义一个探针。添加进`prometheus.yml`
```yml
  - job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]

    static_configs:
      - targets:
        - http://prometheus.io
        - https://prometheus.io
        - http://www.baidu.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115
  - job_name: 'blackbox_exporter'  # collect blackbox exporter's operational metrics.
    static_configs:
      - targets: ['127.0.0.1:9115']
```

- 4.重启prometheus

- 5.会对`http://prometheus.io` `https://prometheus.io` `http://www.baidu.com`不断探测

    ![avatar](./Pictures/prometheus/blackbox_exporter.avif)

## PromQL

### 各种方法

- 4个黄金指标：Four Golden Signals是Google针对大量分布式监控的经验总结，4个黄金指标可以在服务级别帮助衡量终端用户体验、服务中断、业务影响等层面的问题。主要关注与以下四种类型的指标：延迟，通讯量，错误以及饱和度：

    - 1.延迟：服务请求所需时间。
        - 记录用户所有请求所需的时间，重点是要区分成功请求的延迟时间和失败请求的延迟时间。 例如在数据库或者其他关键祸端服务异常触发HTTP 500的情况下，用户也可能会很快得到请求失败的响应内容，如果不加区分计算这些请求的延迟，可能导致计算结果与实际结果产生巨大的差异。除此以外，在微服务中通常提倡“快速失败”，开发人员需要特别注意这些延迟较大的错误，因为这些缓慢的错误会明显影响系统的性能，因此追踪这些错误的延迟也是非常重要的。

    - 2.通讯量：监控当前系统的流量，用于衡量服务的容量需求。
        - 流量对于不同类型的系统而言可能代表不同的含义。例如，在HTTP REST API中, 流量通常是每秒HTTP请求数；

    - 3.错误：监控当前系统所有发生的错误请求，衡量当前系统错误发生的速率。
    对于失败而言有些是显式的(比如, HTTP 500错误)，而有些是隐式(比如，HTTP响应200，但实际业务流程依然是失败的)。
        - 对于一些显式的错误如HTTP 500可以通过在负载均衡器(如Nginx)上进行捕获，而对于一些系统内部的异常，则可能需要直接从服务中添加钩子统计并进行获取。

    - 4.饱和度：衡量当前服务的饱和度。
        - 主要强调最能影响服务状态的受限制的资源。 例如，如果系统主要受内存影响，那就主要关注系统的内存状态，如果系统主要受限与磁盘I/O，那就主要观测磁盘I/O的状态。因为通常情况下，当这些资源达到饱和后，服务的性能会明显下降。同时还可以利用饱和度对系统做出预测，比如，“磁盘是否可能在4个小时候就满了”。

- RED方法：是Weave Cloud在基于Google的“4个黄金指标”的原则下结合Prometheus以及Kubernetes容器实践，细化和总结的方法论，特别适合于云原生应用以及微服务架构应用的监控和度量。主要关注以下三种关键指标：

    - (请求)速率：服务每秒接收的请求数。
    - (请求)错误：每秒失败的请求数。
    - (请求)耗时：每个请求的耗时。
    - 在“4大黄金信号”的原则下，RED方法可以有效的帮助用户衡量云原生以及微服务应用下的用户体验问题。


- USE方法全称"Utilization Saturation and Errors Method"：主要用于分析系统性能问题，可以指导用户快速识别资源瓶颈以及错误的方法。
    - 使用率：关注系统资源的使用情况。 这里的资源主要包括但不限于：CPU，内存，网络，磁盘等等。100%的使用率通常是系统性能瓶颈的标志。
    - 饱和度：例如CPU的平均运行排队长度，这里主要是针对资源的饱和度(注意，不同于4大黄金信号)。任何资源在某种程度上的饱和都可能导致系统性能的下降。
    - 错误：错误计数。例如：“网卡在数据包传输过程中检测到的以太网网络冲突了14次”。

### metrics（指标）类型

| metrics类型         | 说明                                                                                                                                       |
|---------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| 测量型（Gauge）     | 可增可减的数字（本质上是度量的快照）。常见的如内存使用率。                                                                                 |
| 计数型（counter）   | 只增不减，除非重置为0。比如某系统的HTTP请求量。                                                                                            |
| 直方图（histogram） | 通过对监控的指标点进行抽样，展示数据分布频率情况的类型。                                                                                   |
| 摘要（summary）     | 与Histogram非常类似，主要区别是summary在客户端完成聚合，而Histogram在服务端完成。因此summary只适合不需要集中聚合的单体指标(如GC相关指标)。 |


- 三条经验法则：

    - 如果需要多个采集节点的数据聚合、汇总，请选择直方图；
    - 如果需要观察多个采集节点数据的分布情况，请选择直方图；
    - 如果不需要考虑集群（如GC相关信息），可选择summary，它可以提供更加准确的分位数。


- 测量型（Gauge）
    ```promql
    rate(prometheus_http_requests_total)
    ```

- 计数型（counter）
    ```promql
    node_memory_MemFree_bytes
    ```

- Histogram和Summary

    - 如果大多数API请求都维持在100ms的响应时间范围内，而个别请求的响应时间需要5s，那么就会导致某些WEB页面的响应时间落到中位数的情况，而这种现象被称为长尾问题。为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，统计延迟在0~10ms之间的请求数有多少而10~20ms之间的请求数又有多少。

    ```promql
    prometheus_tsdb_compaction_chunk_range_bucket{le="100"}
    prometheus_tsdb_compaction_chunk_range_bucket{le="400"}
    ```


### 各种操作符

- Instant Vector selectors（即时向量选择器）：查询计算最新样本即时向量

```promql
# 指定label
up{job="mysql"}

# !=反向匹配
up{job!="mysql"}

# =~正则表达式
node_filesystem_size_bytes{mountpoint=~'.*home'}

# !~正则表达式
node_filesystem_size_bytes{mountpoint!~'.*home'}
```

- Range Vector selectors

|            |       |         |         |         |         |
|------------|-------|---------|---------|---------|---------|
| 时间区间： | s(秒) | h（小时 | d（天） | w（周） | y（年） |

```promql
# 1分钟内的
rate(process_cpu_seconds_total[1m])
rate(process_cpu_seconds_total{job="prometheus"}[1m])

# without忽略idle的cpu
avg without(cpu) (rate(node_cpu_seconds_total{mode="idle"}[1m]))

# 1分钟前的
rate(process_cpu_seconds_total) offset 1m

# 1小时前的1分钟内的
rate(process_cpu_seconds_total[1m] offset 1h)
```

- 聚合操作

| 聚合操作                       |
|--------------------------------|
| sum (求和)                     |
| min (最小值)                   |
| max (最大值)                   |
| avg (平均值)                   |
| stddev (标准差)                |
| stdvar (标准方差)              |
| count (计数)                   |
| count_values (对value进行计数) |
| bottomk (后n条时序)            |
| topk (前n条时序)               |
| quantile (分位数)              |

- `without`：用于从计算结果中移除列举的标签，而保留其它标签。
- `by`：则正好相反，结果向量中只保留列出的标签，其余标签则移除。

    ```promql
    sum(http_requests_total) without (instance)
    # 两者等价
    sum(http_requests_total) by (code,handler,job,method)
    ```

```promql
# sum总数
sum(prometheus_http_requests_total)

sum(sum(irate(prometheus_http_requests_total{mode!='idle'}[5m]))  / sum(irate(prometheus_http_requests_total[5m]))) by (instance)

# max最大值
max(prometheus_http_requests_total)

# avg平均值。每个cpu每秒的空闲时间，并求出所有cpu的平均值
avg without (cpu,mode) (rate(node_cpu_seconds_total {mode = "idle"} [1m]))

# 按照mode计算主机CPU的平均使用时间
avg(node_cpu_seconds_total) by (mode)

# count统计。统计有多少个设备
count(node_disk_read_bytes_total)

# count_values统计相同的值。

# topk排序。
topk(10, prometheus_http_requests_total)

quantile用于计算当前样本数据值的分布情况quantile(φ, express)其中0 ≤ φ ≤ 1。例如，当φ为0.5时，即表示找到当前样本数据中的中位数：
quantile(0.5, http_requests_total)
```

- 算术运算符

| 算术运算符 | 说明     |
|------------|----------|
| +          | (加法)   |
| -          | (减法)   |
| *          | (乘法)   |
| /          | (除法)   |
| %          | (求余)   |
| ^          | (幂运算) |

```promql
# MB单位
node_memory_MemFree_bytes / (1024 * 1024)

# 主机磁盘IO的总量
node_disk_written_bytes_total + node_disk_read_bytes_total

# 内存使用率
(node_memory_MemTotal_bytes - node_memory_MemFree_bytes) / node_memory_MemTotal_bytes * 100

# 内存使用率>95%
(node_memory_MemTotal_bytes - node_memory_MemFree_bytes) / node_memory_MemTotal_bytes > 0.95
```

- 集合运算符

    | 集合运算符    |
    |---------------|
    | and (并且)    |
    | or (或者)     |
    | unless (排除) |

- PromQL运算符中优先级由高到低依次为：
    - 1.`^`
    - 2.`*`, `/`, `%`
    - 3.`+`, `-`
    - 4.`==`, `!=`, `<=`, `<`, `>=`, `>`
    - 5.`and`, `unless`
    - 6.`or`

- count指标。只能在计数型（counter）的metrics使用

```promql
# increase统计增长率。3分钟内的每秒增长率
increase(process_cpu_seconds_total[3m]) / 180

# rate每秒平均律
rate(process_cpu_seconds_total[3m])

# irate比rate灵敏度更高，容易受干扰，机器重启会自动调整。分析长期趋势，告警规则适合使用rate
irate(process_cpu_seconds_total[3m])
```

- Gauge指标

```promql
# predict_linear线性回归预测。预测4小时内磁盘会不会占满
predict_linear(node_filesystem_size_bytes{job="node"}[1h], 4 * 3600) < 0
```

#### 通过curl在HTTP中使用PromQL

```sh
curl 'http://localhost:9090/api/v1/query?query=up' | json
# up在时间点2015-07-01T20:10:51.781Z
curl 'http://localhost:9090/api/v1/query?query=up&time=2015-07-01T20:10:51.781Z' | json
```

### Recoding Rules

- PromQL较为复杂且计算量较大时，直接使用PromQL可能会导致Prometheus响应超时的情况。这时需要一种能够类似于后台批处理的机制能够在后台完成这些复杂运算的计算，对于使用者而言只需要查询这些运算结果即可。

- Prometheus通过Recoding Rule规则支持这种后台计算的方式，可以实现对复杂查询的性能优化，提高查询效率。

- 通过`rule_files`定义recoding rule规则文件的访问路径
    ```yml
    rule_files:
      [ - <filepath_glob> ... ]
    ```

## 服务发现

- 通过服务发现的方式，管理员可以在不重启Prometheus服务的情况下动态的发现需要监控的Target实例信息。

### 基于文件的服务发现

- json文件中分别定义了3个采集任务，以及每个任务对应的Target列表：

    ```json
    [
      {
        "targets": [ "localhost:8080"],
        "labels": {
          "env": "localhost",
          "job": "cadvisor"
        }
      },
      {
        "targets": [ "localhost:9104" ],
        "labels": {
          "env": "prod",
          "job": "mysqld"
        }
      },
      {
        "targets": [ "localhost:9100"],
        "labels": {
          "env": "prod",
          "job": "node"
        }
      }
    ]
    ```

    - 并为Target实例添加了自定义标签`env`：
        ```
        # env="prod"
        node_cpu_seconds_total{cpu="cpu0",env="prod",instance="localhost:9100",job="node",mode="idle"}
        ```

- 定义了一个基于`file_sd_configs`的监控采集任务，其中模式的任务名称为`file_ds`。在JSON文件中可以使用job标签覆盖默认的job名称

```yml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
scrape_configs:
- job_name: 'file_ds'
  file_sd_configs:
  - files:
    - targets.json
```

- 默认每5m重新读取一次文件内容，当需要修改时，可以通过`refresh_interval`进行设置
```yml
- job_name: 'file_ds'
  file_sd_configs:
  - refresh_interval: 1m
    files:
    - targets.json
```

### consul

- Consul是由HashiCorp开发的一个支持多数据中心的分布式服务发现和键值对存储服务的开源软件，被大量应用于基于微服务的软件架构当中。

#### 服务注册

- 在`consul.d`目录下创建`/node_exporter.json`文件
```json
{
    "service": {
        "id": "node_exporter",
        "name": "node_exporter",
        "address": "127.0.0.1",
        "port": 9100
    }
}
```

```sh
# 启动consul
consul agent -dev -config-dir=/etc/consul.d

# 查看当前集群中的所有节点
consul members

# 获取服务列表：
curl http://localhost:8500/v1/catalog/service/node_exporter

# Consul还提供了内置的DNS服务。解析dns查询
dig @127.0.0.1 -p 8600 node_exporter.service.consul
```

#### 集成prometheus

- 添加进`prometheus.yml`
```yml
  - job_name: 'consul_sd_node_exporter'
    scheme: http
    consul_sd_configs:
      - server: 127.0.0.1:8500
        services: ['node_exporter']
```

- 不要忘记启动`node_exporter`

### 基于dns服务发现

- 不是所有应用环境都能使用promethues文件和consul服务发现

- 在不对外暴露ip地址，部署dns解析服务

- 添加进`prometheus.yml`
```yml
  - job_name: 'DNS_sd_node_exporter'
    dns_sd_configs:
      - names: ['_prometheus._tcp.363.com']
```

- 在`/etc/resolv.conf`上添加dns服务器地址`nameserver <dns服务器ip>`，然后重启prometheus

### 服务发现与relabel_configs

- 对于线上环境我们可能会划分为:dev, stage, prod不同的集群。每一个集群运行多个主机节点，每个服务器节点上运行一个Node Exporter实例。Node Exporter实例会自动注册到Consul中，而Prometheus则根据Consul返回的Node Exporter实例信息动态的维护Target列表，从而向这些Target轮询监控数据。

    ![avatar](./Pictures/prometheus/基于Consul的服务发现.avif)

    - 问题：希望Prometheus Server能够按照某些规则（比如标签）从服务发现注册中心返回的Target实例中有选择性的采集某些Exporter实例的监控数据。
        - 按照不同的环境dev, stage, prod聚合监控数据？
        - 对于研发团队而言，我可能只关心dev环境的监控数据，如何处理？
        - 如果为每一个团队单独搭建一个Prometheus Server。那么如何让不同团队的Prometheus Server采集不同的环境监控数据？

    - 解决方法：Prometheus采集的样本数据中都会包含一个名为`instance`的标签，该标签的内容对应到Target实例的`__address__`。这里实际上是发生了一次标签的重写处理。

        - 这种发生在采集样本数据之前，对Target实例的标签进行重写的机制在Prometheus被称为Relabeling。


- 在Prometheus所有的Target实例中，都包含一些默认的Metadata标签信息。下面图片中的`Discovered labels:`便是Metadata
![avatar](./Pictures/prometheus/Target-Metadata.avif)

- 除了这些默认的标签以外，还可以自定义Target的标签

    - 在[基于文件的服务发现](#基于文件的服务发现)一节中，就为Target实例添加了自定义标签`env`：
        ```promql
        # env="prod"
        node_cpu_seconds_total{cpu="cpu0",env="prod",instance="localhost:9100",job="node",mode="idle"}
        ```

- Relabeling最基本的应用场景就是基于Target实例中包含的metadata标签，动态的添加或者覆盖标签

    ```promql
    # 在默认情况下，从Node Exporter实例采集上来的样本数据如下所示：
    node_cpu_seconds_total{cpu="cpu0",instance="localhost:9100",job="node",mode="idle"} 93970.8203125

    # 我们希望能有一个额外的标签dc可以表示该样本所属的数据中心：
    node_cpu_seconds_total{cpu="cpu0",instance="localhost:9100",job="node",mode="idle", dc="dc1"} 93970.8203125
    ```

    - 一个最简单的`relabel_config`配置：添加dc便签
    ```yml
    scrape_configs:
      - job_name: node_exporter
        consul_sd_configs:
          - server: localhost:8500
            services:
              - node_exporter
        relabel_configs:
        - source_labels:  ["__meta_consul_dc"]
          target_label: "dc"
    ```

- 完整的`relabel_config`配置
    ```yml
    # The source labels select values from existing labels. Their content is concatenated
    # using the configured separator and matched against the configured regular expression
    # for the replace, keep, and drop actions.
    [ source_labels: '[' <labelname> [, ...] ']' ]

    # Separator placed between concatenated source label values.
    [ separator: <string> | default = ; ]

    # Label to which the resulting value is written in a replace action.
    # It is mandatory for replace actions. Regex capture groups are available.
    [ target_label: <labelname> ]

    # Regular expression against which the extracted value is matched.
    [ regex: <regex> | default = (.*) ]

    # Modulus to take of the hash of the source label values.
    [ modulus: <uint64> ]

    # Replacement value against which a regex replace is performed if the
    # regular expression matches. Regex capture groups are available.
    [ replacement: <string> | default = $1 ]

    # Action to perform based on regex matching.
    [ action: <relabel_action> | default = replace ]
    ```

    - `action`定义了当前relabel_config对Metadata标签的处理方式

        - `replace`（默认值）：会根据regex的配置匹配`source_labels`标签的值（多个`source_label`的值会按照separator进行拼接），并且将匹配到的值写入到`target_label`当中
            - 如果有多个匹配组，则可以使用${1}, ${2}确定写入的内容。如果没匹配到任何内容则不对`target_label`进行重新。

        - `labelmap`：与replace不同的是，labelmap会根据regex的定义去匹配Target实例所有标签的名称，并且以匹配到的内容为新的标签名称，其值作为新标签的值。

            - 例子：监控Kubernetes下所有的主机节点时，为将这些节点上定义的标签写入到样本中时，可以使用如下relabel_config配置：

                ```yml
                - job_name: 'kubernetes-nodes'
                  kubernetes_sd_configs:
                  - role: node
                  relabel_configs:
                  - action: labelmap
                    regex: __meta_kubernetes_node_label_(.+)
                ```

        - `labelkeep`或者`labeldrop`：对Target标签进行过滤

            - `labelkeep`：仅保留符合过滤条件的标签
                ```yml
                relabel_configs:
                  - regex: label_should_drop_(.+)
                    action: labeldrop
                ```

            - `labeldrop`：移除那些不匹配regex定义的所有标签

        - `keep`和`drop`：过滤Target实例。可以简单理解为keep用于选择，而drop用于排除。

            - `keep`：Prometheus会丢弃`source_labels`的值中没有匹配到regex正则表达式内容的Target实例
            - `drop`：则会丢弃那些`source_labels`的值匹配到regex正则表达式内容的Target实例。

            - 应用场景：不同职能（开发、测试、运维）的人员可能只关心其中一部分的监控数据，他们可能各自部署的自己的Prometheus Server用于监控自己关心的指标数据，如果让这些Prometheus Server采集所有环境中的所有Exporter数据显然会存在大量的资源浪费。（本节开头提到的问题）

            - 例子：只希望采集数据中心dc1中的Node Exporter实例的样本数据
                ```yml
                scrape_configs:
                  - job_name: node_exporter
                    consul_sd_configs:
                      - server: localhost:8500
                        services:
                          - node_exporter
                    relabel_configs:
                    - source_labels:  ["__meta_consul_dc"]
                      regex: "dc1"
                      action: keep
                ```

- 使用hashmod计算`source_labels`的Hash值

    - 当relabel_config设置为hashmod时，Prometheus会根据modulus的值作为系数，计算source_labels值的hash值。

        ```yml
        scrape_configs
        - job_name: 'file_ds'
          relabel_configs:
            - source_labels: [__address__]
              modulus:       4
              target_label:  tmp_hash
              action:        hashmod
          file_sd_configs:
          - files:
            - targets.json
        ```

    - 当前Target实例`__address__`的值以4作为系数，这样每个Target实例都会包含一个新的标签tmp_hash，并且该值的范围在1~4之间

## 可视化

### Console Template

- [《prometheus-book》使用Console Template](https://yunlzheng.gitbook.io/prometheus-book/part-ii-prometheus-jin-jie/grafana/use-console-template)

- Prometheus内置了一个简单的解决方案，它允许用户通过Go模板语言创建任意的控制台界面，并且通过Prometheus Server对外提供访问路径。

### Grafana

- [腾讯技术工程：上手开源数据可视化工具 Grafana](https://view.inews.qq.com/a/20221014A06MAQ00)

- 数据源（Data Source）：支持Graphite, InfluxDB, OpenTSDB, Prometheus, Elasticsearch, CloudWatch的支持

#### dashboard

- 插件：需要安装对应组件的插件，不然Dashboard无法选择对应的Data Source

    - 安装插件后需要重启grafana

```sh
# 安装饼图
grafana cli plugins install grafana-piechart-panel

# 安装gantt图
grafana cli plugins install marcusolsson-gantt-panel

# mongodb Data Source
grafana cli plugins install grafana-mongodb-datasource

# redis Data Source
grafana cli plugins install redis-datasource
```

- 如果对应的`exporter`组件没有配置好，则不会有数据

- [Grafana社区用户分享的Dashboard](https://grafana.com/grafana/dashboards/)

| id号  | 可视化的组件    |
|-------|-----------------|
| 8919  | node_exporter   |
| 1860  | node_exporter   |
| 7587  | blackbox        |
| 2583  | mongodb         |
| 7365  | MySQL Overview  |
| 14031 | MySQL Dashboard |
| 12776 | Redis           |
| 11835 | Redis           |


## alertmanager告警

![avatar](./Pictures/prometheus/alertmanager告警架构.avif)

- 告警规则：告警规则实际上主要由PromQL进行定义，其实际意义是当表达式（PromQL）查询结果持续多长时间（During）后出发告警

- Alertmanager除了提供基本的**告警通知**能力以外还有：

    ![avatar](./Pictures/prometheus/alertmanager特性.avif)

    - 分组：机制可以将详细的告警信息合并成一个通知。
        - 例子：系统宕机导致大量的告警被同时触发，在这种情况下分组机制可以将这些被触发的告警合并为一个告警通知，避免一次性接受大量的告警通知，而无法对问题进行快速定位。
        - 例子：当集群中有数百个正在运行的服务实例，并且为每一个实例设置了告警规则。假如此时发生了网络故障，可能导致大量的服务实例无法连接到数据库，结果就会有数百个告警被发送到Alertmanager。

    - 抑制：当某一告警发出后，可以停止重复发送由此告警引发的其它告警的机制。
        - 当集群不可访问时触发了一次告警，通过配置Alertmanager可以忽略与该集群有关的其它所有告警。这样可以避免接收到大量与实际问题无关的告警通知。

    - 静默：提供了一个简单的机制可以快速根据标签对告警进行静默处理。如果接收到的告警符合静默的配置，Alertmanager则不会发送告警通知。

        - 需要在Alertmanager的Werb页面上进行设置。

### 告警

- [awesome-prometheus-alerts：各种写好的组件告警规则](https://github.com/samber/awesome-prometheus-alerts)

- 启动alertmanager后打开网站：`http://127.0.0.1:9093/#/alerts`

- `prometheus.yml`中注销相应的字段

    ```yml
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              - 127.0.0.1:9093
    ```

- 添加进`prometheus.yml`

    ```yml
      - job_name: 'Alertmanager'
        static_configs:
        - targets: ['127.0.0.1:9093']
    ```

- 告警规则

    - 该网站复制配置文件，可以绘制出routing tree`https://www.prometheus.io/webtools/alerting/routing-tree-editor/`

- 实验：监控node_exporter的up状态

    - 1.新建一个`up-node_exporter.yml`rule文件。

        ```yml
        groups:
        - name: UP
          rules:
          - alert: node
            expr: up{job="node"} == 0
            for: 3m
            labels:
              severity: critical
            annotations:
              description: "Node has been down for more than 3 minutes."
              summary: "Node down"
        ```

        | 字段        | 说明                                                                                                                     |
        |-------------|--------------------------------------------------------------------------------------------------------------------------|
        | alert       | 告警规则的名称。                                                                                                         |
        | expr        | 基于PromQL表达式告警触发条件，用于计算是否有时间序列满足该条件。                                                         |
        | for         | 评估等待时间，可选参数。用于表示只有当触发条件持续一段时间后才发送告警。在等待期间新产生告警的状态为pending。            |
        | labels      | 自定义标签，允许用户指定要附加到告警上的一组附加标签。                                                                   |
        | annotations | 用于指定一组附加信息，比如用于描述告警详细信息的文字等，annotations的内容在告警产生时会一同作为参数发送到Alertmanager。 |

    - 2.配置告警规则文件。添加进`prometheus.yml`
        ```yml
        rule_files:
          - "/etc/prometheus/rules/up-node_exporter.yml"
        ```

        - 也可以是这样
            ```yml
            rule_files:
              - /etc/prometheus/rules/*.rules
            ```

        - 默认情况下Prometheus会每分钟对这些告警规则进行计算，如果用户想定义自己的告警计算周期，则可以通过`evaluation_interval`来覆盖默认的计算周期
            ```yml
            global:
              evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
            ```

    - 3.关联Prometheus与Alertmanager。添加进`prometheus.yml`
        ```yml
        alerting:
          alertmanagers:
            - static_configs:
                - targets: ['localhost:9093']
        ```

    - 4.重启prometheus
    - 5.启动alertmanager。可以在`http://127.0.0.1:9093/#/status`查看配置文件`/etc/alertmanager/alertmanager.yml`
    - 6.查看rules是否加载`http://127.0.0.1:9090/rules`
    - 7.停止`node_exporter`
    - 8.确认告警是否成功
        - prometheus：打开`http://127.0.0.1:9090/alerts`查看是否收到告警
            - 设置了`for: 3m`，此时的status为`PENDING`：表示满足触发条件，但没有满足告警持续时间
                ![avatar](./Pictures/prometheus/alertmanager告警实验.avif)

            - 3分钟后，status为`FIRING`：表示满足告警持续时间
                ![avatar](./Pictures/prometheus/alertmanager告警实验1.avif)

        - alertmanager：打开`http://127.0.0.1:9093/#/alerts`查看是否收到告警
            ![avatar](./Pictures/prometheus/alertmanager告警实验2.avif)

### alertmanager配置文件

| 配置组成部分              | 说明                                                                                                   |
|---------------------------|--------------------------------------------------------------------------------------------------------|
| 全局配置（global）        | 用于定义一些全局的公共参数，如全局的SMTP配置，Slack配置等内容；                                        |
| 模板（templates）         | 用于定义告警通知时的模板，如HTML模板，邮件模板等；                                                     |
| 告警路由（route）         | 根据标签匹配，确定当前告警应该如何处理；                                                               |
| 接收人（receivers）       | 接收人是一个抽象的概念，它可以是一个邮箱也可以是微信，Slack或者Webhook等，接收人一般配合告警路由使用； |
| 抑制规则（inhibit_rules） | 合理设置抑制规则可以减少垃圾告警的产生                                                                 |

```yml
global:
  [ resolve_timeout: <duration> | default = 5m ]
  [ smtp_from: <tmpl_string> ]
  [ smtp_smarthost: <string> ]
  [ smtp_hello: <string> | default = "localhost" ]
  [ smtp_auth_username: <string> ]
  [ smtp_auth_password: <secret> ]
  [ smtp_auth_identity: <string> ]
  [ smtp_auth_secret: <secret> ]
  [ smtp_require_tls: <bool> | default = true ]
  [ slack_api_url: <secret> ]
  [ victorops_api_key: <secret> ]
  [ victorops_api_url: <string> | default = "https://alert.victorops.com/integrations/generic/20131114/alert/" ]
  [ pagerduty_url: <string> | default = "https://events.pagerduty.com/v2/enqueue" ]
  [ opsgenie_api_key: <secret> ]
  [ opsgenie_api_url: <string> | default = "https://api.opsgenie.com/" ]
  [ hipchat_api_url: <string> | default = "https://api.hipchat.com/" ]
  [ hipchat_auth_token: <secret> ]
  [ wechat_api_url: <string> | default = "https://qyapi.weixin.qq.com/cgi-bin/" ]
  [ wechat_api_secret: <secret> ]
  [ wechat_api_corp_id: <string> ]
  [ http_config: <http_config> ]

templates:
  [ - <filepath> ... ]

route: <route>

receivers:
  - <receiver> ...

inhibit_rules:
  [ - <inhibit_rule> ... ]
```

- `resolve_timeout`：该参数定义了当Alertmanager持续多长时间未接收到告警后，标记告警状态为resolved（已解决）
    - 该参数的定义可能会影响到告警恢复通知的接收时间

#### route（路由规则）

- 最简单的route定义

    ```yml
    route:
      group_by: ['alertname']
      receiver: 'web.hook'
    receivers:
    - name: 'web.hook'
      webhook_configs:
      - url: 'http://127.0.0.1:5001/'
    ```

- route的完整定义
    ```yml
    route:
        [ receiver: <string> ]
        [ group_by: '[' <labelname>, ... ']' ]
        [ continue: <boolean> | default = false ]

        match:
          [ <labelname>: <labelvalue>, ... ]

        match_re:
          [ <labelname>: <regex>, ... ]

        [ group_wait: <duration> | default = 30s ]
        [ group_interval: <duration> | default = 5m ]
        [ repeat_interval: <duration> | default = 4h ]

        routes:
          [ - <route> ... ]
    ```

- 问题：对于不同级别的告警，我们可能会有完全不同的处理方式
- 解决方法：在`route`中，我们还可以定义更多的子`routes`，这些`routes`通过标签匹配告警的处理方式
    - 默认情况下，告警进入到顶级route后会遍历所有的子节点，直到找到最深的匹配routes，并将告警发送到该routes定义的receiver中。
        - 但如果route中设置continue的值为false，那么告警在匹配到第一个子节点之后就直接停止。如果continue为true，报警则会继续进行后续子节点的匹配。如果当前告警匹配不到任何的子节点，那该告警将会基于当前路由节点的接收器配置方式进行处理。

- 告警的匹配：2种方法
    - 1.字符串验证：通过设置`match`规则判断当前告警中是否存在标签labelname并且其值等于labelvalue。
    - 2.正则表达式：通过设置`match_re`验证当前告警标签的值是否满足正则表达式的内容。

- `repeat_interval`：如果警报已经成功发送通知, 如果想设置发送告警通知之前要等待时间

- 告警分组：对告警通知进行分组，将多条告警合合并为一个通知

    - `group_by`：基于告警中包含的标签，如果满足group_by中定义标签名称，那么这些告警将会合并为一个通知发送给receiver。
    - `group_wait`：设置等待时间。有的时候为了能够一次性收集和发送更多的相关信息时
    - `group_interval`：则用于定义相同的Group之间发送告警通知的时间间隔。

- 解读以下配置路由规则：

    ```yml
    route:
      receiver: 'default-receiver'
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      group_by: [cluster, alertname]
      routes:
      - receiver: 'database-pager'
        group_wait: 10s
        match_re:
          service: mysql|cassandra
      - receiver: 'frontend-pager'
        group_by: [product, environment]
        match:
          team: frontend
    ```

    - 默认情况：所有告警都会发送给`default-receiver`
    - 其中一个子路由：如果告警中包含service标签（并且service为MySQL或者Cassandra），则向`database-pager`发送告警通知
        - 注意：由于这里没有定义`group_by`等属性，这些属性的配置信息将从上级路由继承，`database-pager`将会接收到按`cluster`和`alertname`进行分组的告警通知。

    - 如果有team标签的告警，并且team的值为frontend。Alertmanager将会按照标签product和environment对告警进行分组。
        - 此时如果应用出现异常，开发团队就能清楚的知道哪一个环境(environment)中的哪一个应用程序出现了问题，可以快速对应用进行问题定位。

#### receivers（告警接收器）

- 告警接收器有：email、企业微信、webhooks的钉钉

- 如果不设置告警接收器，只能在浏览器查看告警通知。

##### [PrometheusAlert：可以接入alertmanager的国内告警系统](https://github.com/feiyu563/PrometheusAlert)

- PrometheusAlert：
    - 可以自定义告警模板
    - 可以模拟测试

- 1.安装并启动PrometheusAlert
    ```sh
    # 下载
    curl -LO https://github.com/feiyu563/PrometheusAlert/releases/download/v4.9/linux.zip

    # 解压linux.zip
    x linux.zip

    # 赋予执行权限
    chmod 755 PrometheusAlert

    # 启动PrometheusAlert
    ./PrometheusAlert
    ```

- 2.配置`app.conf`文件

- 3.添加进`alertmanager.yml`

    ```yml
    route:
      # 指定为receiver
      receiver: 'web.hook.prometheusalert'
    receivers:
      - name: 'web.hook.prometheusalert'
        webhook_configs:
        ## 钉钉
        - url: 'http://127.0.0.1:8080/prometheusalert?type=dd&tpl=prometheus-dd&ddurl=https://oapi.dingtalk.com/robot/send?access_token=0976ee8b220bd0a9d6e08761a95da0ee7dc14c1971fa8187d1d77094520ea199'
    ```

- 4.重启alertmanager

- 5.使用之前的关闭`node_exporter`告警实验。查看钉钉机器人是否收到告警
    ![avatar](./Pictures/prometheus/alertmanager-prometheusalert.avif)
    ![avatar](./Pictures/prometheus/alertmanager-prometheusalert1.avif)

##### gmail邮箱

```yml
global:
  smtp_smarthost: smtp.gmail.com:587
  smtp_from: <smtp mail from>
  smtp_auth_username: <usernae>
  smtp_auth_identity: <username>
  smtp_auth_password: <password>

route:
  group_by: ['alertname']
  receiver: 'default-receiver'

receivers:
  - name: default-receiver
    email_configs:
      - to: <mail to address>
        send_resolved: true
```

##### 企业微信（失败）

- 1.进入企业微信[官网](https://work.weixin.qq.com/)
- 2.登陆后。创建应用，并选择部门。
- 3.创建后：保存`AgentId`、`Secret`
- 4.进入通讯录页面找到并保存：`部门ID`（一般为1）
- 5.点击成员名字，并保存：`账号`
- 6.进入我的企业页面：找到并保存`企业ID`

- [weixin-alert：测试工具](https://github.com/OneOaaS/weixin-alert)

    | 参数         | 说明     |
    |--------------|----------|
    | --corpid     | 企业ID   |
    | --corpsecret | Secret   |
    | --user       | 成员账号 |
    | --agentid    | AgentId  |

    ```sh
    # 下载测试工具
    curl -LO https://raw.githubusercontent.com/OneOaaS/weixin-alert/master/weixin_linux_amd64
    chmod 755 weixin_linux_amd64

    # 测试
    ./weixin_linux_amd64 \
        -corpid= \
        -corpsecret= \
        -msg="您好告警测试" \
        -user= \
        -agentid=
    ```

- 添加进`alertmanager.yml`

    ```yml
    global:
      resolve_timeout: 5m

      # 这个值。无需修改
      wechat_api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'

      # 企业id
      wechat_api_corp_id: ''
      # Secret
      wechat_api_secret: ''
    ```

    ```yml
    receivers:
      - name: 'wechat'
        wechat_configs:
          - send_resolved: true
            # 部门id
            to_party: '1'
            # AgentId
            agent_id: ''
            # 企业id
            corp_id: ''
            # 成员账号。@all为所有成员
            to_user: '@all'
            # 这个值。无需修改
            api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'
            # Secret
            api_secret: ''
    ```

    ```sh
    # 检查配置
    amtool check-config /etc/alertmanager/alertmanager.yml
    ```

- 重启alertmanager

##### 钉钉dingtalk

- 1.在手机app上创建团队

- 2.登陆钉钉开放平台[官网](https://open-dev.dingtalk.com/)
    - 钉钉网页版、pc版、手机版都无法创建自定义机器人
- 3.点击`创建应用`后，可以创建机器人。
- 4.在手机app群里保存刚才创建机器人的`webhook`
- 5.下载[prometheus-webhook-dingtalk](https://github.com/timonwong/prometheus-webhook-dingtalk)
    ```sh
    # 下载
    curl -LO https://github.com/timonwong/prometheus-webhook-dingtalk/releases/download/v2.1.0/prometheus-webhook-dingtalk-2.1.0.linux-amd64.tar.gz
    # 解压
    ```

- 6.配置`prometheus-webhook-dingtalk`的`config.yml`后
    ```yml
    targets:
      webhook_dingding:
        url: https://oapi.dingtalk.com/robot/send?access_token=0976ee8b220bd0a9d6e08761a95da0ee7dc14c1971fa8187d1d77094520ea199
    ```

- 7.运行`prometheus-webhook-dingtalk`。默认端口为8086
    ```yml
    ./prometheus-webhook-dingtalk
    ```

- 8.添加进`alertmanager.yml`
    ```yml
    route:
      # 指定为web.hook.dingtalk
      receiver: 'web.hook.dingtalk'
    receivers:
      # 设置web.hook.dingtalk
      - name: web.hook.dingtalk
        webhook_configs:
        - url: http://localhost:8060/dingtalk/webhook_dingding/send
          send_resolved: true
    ```

- 9.重启alertmanager

- 10.测试
    ```sh
    # 通过curl发送告警。记得修改access_token
    curl -l -H "Content-type: application/json" -X POST -d '{"msgtype": "markdown","markdown": {"title":"Prometheus告警信息","text": "#### 监控指标\n> 监控描述信息\n\n> ###### 告警时间 \n"},"at": {"isAtAll": false}}' 'https://oapi.dingtalk.com/robot/send?access_token=xxxx'
    ```

    ![avatar](./Pictures/prometheus/alertmanager-prometheus-webhook-dingtalk.avif)

- 11.使用之前的关闭`node_exporter`告警实验。查看钉钉机器人是否收到告警
    - 恢复`node_exporter`后，不知道为什么没有信息

    ![avatar](./Pictures/prometheus/alertmanager-prometheus-webhook-dingtalk1.avif)

##### Slack

- [《prometheus-book》集成Slack](https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/alert/alert-manager-use-receiver/alert-with-slack)

#### inhibit_rules

- 抑制机制：当集群不可用时，用户可能只希望接收到一条告警，告诉他这时候集群出现了问题，而不是大量的如集群中的应用异常、中间件服务异常的告警通知。

```yml
inhibit_rules:
  target_match:
    [ <labelname>: <labelvalue>, ... ]
  target_match_re:
    [ <labelname>: <regex>, ... ]

  source_match:
    [ <labelname>: <labelvalue>, ... ]
  source_match_re:
    [ <labelname>: <regex>, ... ]

  [ equal: '[' <labelname>, ... ']' ]
```

- 告警满足到`target_match`和`target_match_re`规则 and 满足`source_match`或者定义的匹配规则 and 已发送的告警与新产生的告警中`equal`定义的标签完全相同，则启动抑制机制，新的告警不会发送。

- 解读以下配置

```yml
- source_match:
    alertname: NodeDown
    severity: critical
  target_match:
    severity: critical
  equal:
    - node
```

- 某一个主机节点异常宕机导致告警`NodeDown`被触发，同时在告警规则中定义了告警级别`severity=critical`。由于主机异常宕机，该主机上部署的所有服务，中间件会不可用并触发报警。

- 如果有新的告警级别为`severity=critical`，并且告警中标签`node`的值与`NodeDown`告警的相同，则说明新的告警是由`NodeDown`导致的，则启动抑制机制停止向接收器发送通知。

#### silences（临时静默）

- silences配置网址：`http://127.0.0.1:9093/#/silences`

- 除了基于抑制机制可以控制告警通知的行为以外，用户或者管理员还可以直接通过Alertmanager的UI临时屏蔽特定的告警通知。

- 通过定义标签的匹配规则(字符串或者正则表达式)，如果新的告警通知满足静默规则的设置，则停止向receiver发送通知。

#### 告警模板

- [《prometheus-book》告警模板详解](https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/alert/alert-template)

## 集群

- 高可用方案

    | 高可用方案 | 服务可用性 | 数据持久化 | 水平扩展 |
    |------------|------------|------------|----------|
    | 基本HA     | yes        | no         | no       |
    | 远程存储   | no         | yes        | no       |
    | 联邦集群   | no         | no         | yes      |

    - 1.基本HA：服务可用性

        ![avatar](./Pictures/prometheus/集群-基本HA.avif)

        - 基本的HA模式只能确保Promthues服务的可用性问题，但是不解决Prometheus Server之间的数据一致性问题以及持久化问题(数据丢失后无法恢复)，也无法进行动态的扩展。因此这种部署方式适合监控规模不大，Promthues Server也不会频繁发生迁移的情况，并且只需要保存短周期监控数据的场景。

    - 2.基本HA + 远程存储：在基本HA模式的基础上通过添加Remote Storage存储支持

        ![avatar](./Pictures/prometheus/集群-基本HA+远程存储.avif)

        - 在解决了Promthues服务可用性的基础上，同时确保了数据的持久化，当Promthues Server发生宕机或者数据丢失的情况下，可以快速的恢复。 同时Promthues Server可能很好的进行迁移。因此，该方案适用于用户监控规模不大，但是希望能够将监控数据持久化，同时能够确保Promthues Server的可迁移性的场景。

    - 3.基本HA + 远程存储 + 联邦集群：当单台Promthues Server无法处理大量的采集任务时，用户可以考虑基于Prometheus联邦集群的方式将监控采集任务划分到不同的Promthues实例当中即在任务级别功能分区。

        ![avatar](./Pictures/prometheus/集群-基本HA+远程存储+联邦集群.avif)

        - 适用于两种场景：

            - 1.单数据中心 + 大量的采集任务
                - 这种场景下Promthues的性能瓶颈主要在于大量的采集任务，因此用户需要利用Prometheus联邦集群的特性，将不同类型的采集任务划分到不同的Promthues子服务中，从而实现功能分区。
                - 例如一个Promthues Server负责采集基础设施相关的监控指标，另外一个Prometheus Server负责采集应用监控指标。再有上层Prometheus Server实现对数据的汇聚。

            - 2.多数据中心
                - 当Promthues Server无法直接与数据中心中的Exporter进行通讯时，在每一个数据中部署一个单独的Promthues Server负责当前数据中心的采集任务是一个不错的方式。这样可以避免用户进行大量的网络配置，只需要确保主Promthues Server实例能够与当前数据中心的Prometheus Server通讯即可。 中心Promthues Server负责实现对多数据中心数据的聚合。

    - 4.按照实例进行功能分区：极端情况，即单个采集任务的Target数也变得非常巨大。这时简单通过联邦集群进行功能分区，Prometheus Server也无法有效处理时。这种情况只能考虑继续在实例级别进行功能划分。

        ![avatar](./Pictures/prometheus/集群-按照实例进行功能分区.avif)

        - 通过`relabel`设置，我们可以确保当前Prometheus Server只收集当前采集任务的一部分实例的监控指标。
            ```yml
            global:
              external_labels:
                slave: 1  # This is the 2nd slave. This prevents clashes between slaves.
            scrape_configs:
              - job_name: some_job
                relabel_configs:
                - source_labels: [__address__]
                  modulus:       4
                  target_label:  __tmp_hash
                  action:        hashmod
                - source_labels: [__tmp_hash]
                  regex:         ^1$
                  action:        keep
            ```

        - 并且通过当前数据中心的一个中心Prometheus Server将监控数据进行聚合到任务级别。

            ```yml
            - scrape_config:
              - job_name: slaves
                honor_labels: true
                metrics_path: /federate
                params:
                  match[]:
                    - '{__name__=~"^slave:.*"}'   # Request all slave-level time series
                static_configs:
                  - targets:
                    - slave0:9090
                    - slave1:9090
                    - slave3:9090
                    - slave4:9090
            ```

### 联邦集群

- 联邦集群：在每个数据中心部署单独的Prometheus Server，用于采集当前数据中心监控数据。并由一个中心的Prometheus Server负责聚合多个数据中心的监控数据。
    ![avatar](./Pictures/prometheus/集群-联邦集群.avif)

- 配置
    ```yml
    scrape_configs:
      - job_name: 'federate'
        scrape_interval: 15s
        honor_labels: true
        metrics_path: '/federate'
        params:
          'match[]':
            - '{job="prometheus"}'
            - '{__name__=~"job:.*"}'
            - '{__name__=~"node.*"}'
        static_configs:
          - targets:
            - '192.168.77.11:9090'
            - '192.168.77.12:9090'
    ```

### 通过[goreman](https://github.com/mattn/goreman)实现集群

- 可以通过goreman同时部署alertmanager和prometheus集群。部署的方式也大同小异。

- 安装goreman
```sh
go install github.com/mattn/goreman@latest
```

- 安装并启动webhook
```sh
## 安装本地webhook服务
go install github.com/prometheus/alertmanager/examples/webhook@latest

## 启动webhook
webhook
```

#### alertmanager

- alertmanager通过gossip机制同步告警通知状态
    ![avatar](./Pictures/prometheus/集群-alertmanager.avif)

- 当Alertmanager接收到来自Prometheus的告警消息后，会按照以下流程对告警进行处理：

    ![avatar](./Pictures/prometheus/集群-alertmanager-gossip.avif)

    - 1.在第一个阶段Silence中，判断当前通知是否匹配到任何的静默规则，如果没有则进入下一个阶段，否则则中断流水线不发送通知。
    - 2.在第二个阶段Wait中，根据当前Alertmanager在集群中所在的顺序(index)等待index * 5s的时间。
    - 3.当前Alertmanager等待阶段结束后，Dedup阶段则会判断当前Alertmanager数据库中该通知是否已经发送，如果已经发送则中断流水线，不发送告警，否则则进入下一阶段Send对外发送告警通知。
    - 4.告警发送完成后该Alertmanager进入最后一个阶段Gossip，Gossip会通知其他Alertmanager实例当前告警已经发送。其他实例接收到Gossip消息后，则会在自己的数据库中保存该通知已发送的记录。

- 创建文件`/etc/alertmanager/alertmanager-ha.yml`

```yml
route:
receiver: 'default-receiver'
receivers:
  - name: default-receiver
webhook_configs:
  - url: 'http://127.0.0.1'
```

- 创建文件`/etc/alertmanager/alertmanager.procfile`
```
a1: alertmanager --web.listen-address=":9093" --cluster.listen-address="127.0.0.1:8001" --config.file=/etc/alertmanager/alertmanager.yml --storage.path=~/alertmanager/data1

a2: alertmanager --web.listen-address=":9094" --cluster.listen-address="127.0.0.1:8002" --config.file=/etc/alertmanager/alertmanager.yml --storage.path=~/alertmanager/data2

webhook: webhook
```

- 启动2个集群
```sh
goreman -f /etc/alertmanager/alertmanager.procfile start
```

- 添加进`prometheus.yml`
```yml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 127.0.0.1:9093
          - 127.0.0.1:9094
```

- 重启prometheus

- 通过`http://localhost:9093/#/status`可以查看当前Alertmanager集群的状态。

#### prometheus

- 创建文件`prometheus.procfile`
```
p1: /etc/prometheus/prometheus --config.file "/etc/prometheus/prometheus.yml" --storage.tsdb.path "/etc/prometheus/data1" --storage.tsdb.retention=15d --web.console.templates="/etc/prometheus/consoles" --web.console.libraries="/etc/prometheus/console_libraries" --web.max-connections=512 --web.listen-address "0.0.0.0:9090"


p2: /etc/prometheus/prometheus --config.file "/etc/prometheus/prometheus.yml" --storage.tsdb.path "/etc/prometheus/data2" --storage.tsdb.retention=15d --web.console.templates="/etc/prometheus/consoles" --web.console.libraries="/etc/prometheus/console_libraries" --web.max-connections=512 --web.listen-address "0.0.0.0:9091"

webhook: webhook
```

- 启动2个集群
```sh
goreman -f prometheus.procfile start
```

## 日志监控系统

### [mtail](https://github.com/google/mtail)

- 安装mtail

- 添加进`/etc/mtail.conf`
```
## simple line counter
counter lines_total
/$/ {
  lines_total++
}
```

- 启动mtail。启动后可以打开`http://127.0.0.1:3903/`
```sh
sudo mtail --progs /etc/mtail.conf --logs '/var/log/*'
```

- 集成于prometheus

    - `http://127.0.0.1:3903/metrics`有prometheus指标

    - 添加进`prometheus.yml`
        ```yml
          - job_name: 'mtail'
            static_configs:
            - targets: ['127.0.0.1:3903']
        ```

## 存储

- 按照两个小时为一个时间窗口，将两小时内产生的数据存储在一个块(Block)中，每一个块中包含该时间窗口内的所有样本数据(chunks)，元数据文件(meta.json)以及索引文件(index)。

- 为了确保此期间如果Prometheus发生崩溃或者重启时能够恢复数据，Prometheus启动时会从写入日志(WAL)进行重播，从而恢复数据。

    - 此期间如果通过API删除时间序列，删除记录也会保存在单独的逻辑文件当中(tombstone)。

- 通过时间窗口的形式保存所有的样本数据，可以明显提高Prometheus的查询效率，当查询一段时间范围内的所有样本数据时，只需要简单的从落在该范围内的块中查询数据即可。

- Prometheus中存储的每一个样本大概占用1-2字节大小

```
./data
|- 01BKGV7JBM69T2G1BGBGM6KB12 # 块
  |- meta.json  # 元数据
  |- wal        # 写入日志
    |- 000002
    |- 000001
|- 01BKGTZQ1SYQJTR4PB43C8PD98  # 块
  |- meta.json  #元数据
  |- index   # 索引文件
  |- chunks  # 样本数据
    |- 000001
  |- tombstones # 逻辑数据
|- 01BKGTZQ1HHWHV8FBJXW1Y3W0K
  |- meta.json
  |- wal
    |-000001
```

- 如果需要对Prometheus Server的本地磁盘空间做容量规划时，可以通过以下公式计算：

    ```
    needed_disk_space = retention_time_seconds（保留时间） * ingested_samples_per_second（每秒获取样本数） * bytes_per_sample（样本大小）
    ```

    - 如果想减少本地磁盘的容量需求：在保留时间(retention_time_seconds)和样本大小(bytes_per_sample)不变的情况下，只能通过减少每秒获取样本数(ingested_samples_per_second)的方式。

    - 因此有两种手段
        - 1.减少时间序列的数量
        - 2.增加采集样本的时间间隔。
        - 考虑到Prometheus会对时间序列进行压缩效率，减少时间序列的数量效果更明显。

### 远程存储

- Remote Write：在Prometheus配置文件中指定Remote Write(远程写)的URL地址，Prometheus将采集到的样本数据通过HTTP的形式发送给适配器(Adaptor)。

    - 而用户则可以在适配器中对接外部任意的服务。外部服务可以是真正的存储系统，公有云的存储服务，也可以是消息队列等任意形式。

    ![avatar](./Pictures/prometheus/远程存储-Remote_Write.avif)

- Remote Read：当用户发起查询请求后，Promthues将向remote_read中配置的URL发起查询请求(matchers,ranges)，Adaptor根据请求条件从第三方存储服务中获取响应的数据。同时将数据转换为Promthues的原始样本数据返回给Prometheus Server。

    ![avatar](./Pictures/prometheus/远程存储-Remote_Read.avif)

- remote_write和remote_read配置：

    ```yml
    remote_write:
        url: <string>
        [ remote_timeout: <duration> | default = 30s ]
        write_relabel_configs:
        [ - <relabel_config> ... ]
        basic_auth:
        [ username: <string> ]
        [ password: <string> ]
        [ bearer_token: <string> ]
        [ bearer_token_file: /path/to/bearer/token/file ]
        tls_config:
        [ <tls_config> ]
        [ proxy_url: <string> ]

    remote_read:
        url: <string>
        required_matchers:
        [ <labelname>: <labelvalue> ... ]
        [ remote_timeout: <duration> | default = 30s ]
        [ read_recent: <boolean> | default = false ]
        basic_auth:
        [ username: <string> ]
        [ password: <string> ]
        [ bearer_token: <string> ]
        [ bearer_token_file: /path/to/bearer/token/file ]
        [ <tls_config> ]
        [ proxy_url: <string> ]
    ```

    - `url`：用于指定远程读/写的HTTP服务地址。
    - `basic_auth`：如果该URL启动了认证则此参数进行安全认证配置。
    - `tls_concig`：对https的支持需要设定
    - `proxy_url`：主要用于Prometheus无法直接访问适配器服务的情况下


- docker-compose启动influxdb
    ```yml
    version: '2'
    services:
      influxdb:
        image: influxdb:1.3.5
        command: -config /etc/influxdb/influxdb.conf
        ports:
          - "8086:8086"
        environment:
          - INFLUXDB_DB=prometheus
          - INFLUXDB_ADMIN_ENABLED=true
          - INFLUXDB_ADMIN_USER=admin
          - INFLUXDB_ADMIN_PASSWORD=admin
          - INFLUXDB_USER=prom
          - INFLUXDB_USER_PASSWORD=prom
    ```

- 安装[remote_storage_adapter](https://github.com/weetime/remote_storage_adapter)
    ```sh
    git clone https://github.com/weetime/remote_storage_adapter.git
    go build
    ```

- 启动`remote_storage_adapter`
    ```sh
    ./remote_storage_adapter --influxdb-url=http://localhost:8086/ --influxdb.database=prometheus --influxdb.retention-policy=autogen --influxdb.username=prom
    ```

- 添加进`prometheus.yml`
    ```yml
    # Remote write configuration (for Graphite, OpenTSDB, or InfluxDB).
    remote_write:
      - url: "http://localhost:9201/write"

    # Remote read configuration (for InfluxDB only at the moment).
    remote_read:
      - url: "http://localhost:9201/read"
    ```

- 重启prometheus

- 查看数据是否正常写入

    - 非docker启动
        ```sh
        # 启动influx shell
        influx v1 shell
        > use prometheus
        > show MEASUREMENTS
        ```

    - 以docker启动
        ```sh
        docker exec -it 795d0ead87a1 influx
        Connected to http://localhost:8086 version 1.3.5
        InfluxDB shell version: 1.3.5
        > auth
        username: prom
        password:

        > use prometheus
        > SHOW MEASUREMENTS
        ```

## [prophet：时序数据预测的 Python 库](https://github.com/facebook/prophet)

# reference

- [《Prometheus监控技术与实践》配套资料](https://github.com/aiopstack/Prometheus-book)
- [《prometheus-book》](https://yunlzheng.gitbook.io/prometheus-book/)
