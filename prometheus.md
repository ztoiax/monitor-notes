# Prometheus

- [Monitoring Linux Network Stack](http://arthurchiao.art/blog/monitoring-network-stack/)

    - 使用简单的shell脚本，配合prometheus监控网络

- [官方exporter的下载页面](https://prometheus.io/download/)

## exporter

- [exporters列表](https://prometheus.io/docs/instrumenting/exporters/)

- 白盒监控：exporter需要安装到目标机器

```sh
# 更新prometheus.yml配置后。可以发送SIGHUP信号，重启prometheus
kill -s SIGHUP $(pidof prometheus)
```

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
```sh
nginx-prometheus-exporter --nginx.scrape-uri=http://127.0.0.1:8080/stub_status

```

#### [nginx-vts-exporter](https://github.com/hnlq715/nginx-vts-exporter)

- 需要先安装[nginx-module-vts](https://github.com/vozlt/nginx-module-vts)
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

### blackbox_exporter

- 白盒监控：exporter需要安装到目标机器。
    - 除了`blackbox_exporter`外大多都是白盒监控

- 黑盒监控：无需安装到目标机器。通过http、https、ssh、dns、tcp、icmp等方式进行监控
    - `blackbox_exporter`就是黑盒监控

- 新建[blackbox.yml](./prometheus-config/blackbox.yml)

- 启动`blackbox_exporter`
```sh
prometheus-blackbox-exporter --config.file "blackbox.yml"
```

- 添加进`prometheus.yml`
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

- 打开`http://127.0.0.1:9115/`。点击`Probe prometheus.io for http_2xx`滚动到底部如果有`probe_success 0`表示失败


## PromQL
```
# 每个cpu每秒的空闲时间，并求出所有cpu的平均值
avg without (cpu,mode) (rate(node_cpu_seconds_total {mode = "idle"} [1m]))
```

# Grafana

# reference
- [《Prometheus监控技术与实践》配套资料](https://github.com/aiopstack/Prometheus-book)
- [《prometheus-book》](https://yunlzheng.gitbook.io/prometheus-book/)
