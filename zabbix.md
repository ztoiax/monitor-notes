# zabbix

- 主要组件

    - zabbix server是核心组件，所有配置信息、统计信息、操作数据的核心存储。
        - 负责接受客户端发送的报告和信息

    - zabbix数据存储：收集到的数据存储在数据库中。如mysql、oracle、sqlite

    - zabbix web界面：gui接口。通常（但不一定）与zabbix server运行在同一台物理机

    - zabbix proxy代理服务器（可选组件）：分布式监控，代理服务器代替zabbix server收集性能数据和可用性数据，汇总后统一发往zabbix server

    - zabbix agent监控代理：部署到被监控主机上，负责收集数据

- zabbix服务进程：默认情况包含5个进程：zabbix_agentd、zabbix_get、zabbix_proxy、zabbix_sender、zabbix_server。还有一个zabbix_java_gateway是可选功能，需要另外安装

    - 1.zabbix_agentd：是zabbix agent监控代理端守护进程

    - 2.zabbix_get：是zabbix提供的一个工具，通常在zabbix server或者zabbix proxy端执行用来获取远程客户端信息。

        - 其实是zabbix server去zabbix agent端拉取数据的过程，此工具主要用来进行用户排错。例如zabbix server获取不到客户端数据时，可以使用zabbix_get命令获取数据，做故障排查

    - 3.zabbix_sender：是zabbix提供的一个工具，用于发送数据给zabbix server或者zabbix proxy，这其实是zabbix agent端主动推送监控数据到zabbix server端的过程，通常用于耗时比较长的检查或者有大量主机（千台以上）需要监控的场景。
        - 主动推送减轻了zabbix server的压力

    - 4.zabbix_proxy：zabbix的代理守护进程，功能类似zabbix server，唯一不同的是它只是一个中转站，它需要把收集到的数据提交到zabbix server

    - 5.zabbix_java_gateway：zabbix2.0引入的功能。java网关主要用来监控java应用环境，类似于zabbix_agentd进程。
        - 它只能主动推送数据，而不能等待zabbix server或zabbix proxy来拉取数据。它的数据会最终发送给zabbix server或zabbix proxy

    - 6.zabbix_server：整个zabbix系统的核心进程。其他进程zabbix_agentd、zabbix_get、zabbix_proxy、zabbix_sender、zabbix_server、zabbix_java_gateway都需要交给它来统一处理

- 监控术语
    - 1.主机（host）：监控的一台服务器或网络设备，可以通过IP或主机名指定
    - 2.主机组（host group）：主机组是主机的逻辑组。包含主机和模板，但同一个主机组内的主机和模板没有任何直接的关联
    - 3.监控项（item）：如cpu负载。由key来标识
    - 4.触发器（trigger）：监控阈值。
        - 如果大于阈值：触发器状态会从“OK”转变为“Problem”
        - 如果小于阈值：触发器状态有转变为“OK”
    - 5.应用集（application）：一组监控项组成的逻辑集合
    - 6.动作（action）：对于监控出现问题，事先定义的处理方法。如发送通知、何时执行操作、执行的频率等
    - 7.报警媒介类型（media）：发送通知的手段。如email、jabber或者sms等
    - 8.模板（template）：一组可以被应用到一个或多个主机上的实体集合，一个模板通常包含了应用集、监控项、触发器、图形、聚合图形、自动发现规则、web场景等几个项目。
        - 模板可以直接链接到某个主机上
        - 是学习zabbix的一个难点

## 安装

- zabbix web端是apache或nginx和php进行构建的

- [官网教程](https://www.zabbix.com/download?utm_campaign=mainpage&utm_source=website&utm_medium=header)

- 安装lnmp

    - `php.ini`设置
        ```ini
        extension=bcmath
        extension=gd
        extension=sockets
        extension=mysqli
        extension=gettext
        post_max_size = 16M
        max_execution_time = 300
        max_input_time = 300
        date.timezone = "UTC"
        ```

- 安装zabbix
```sh
# 安装前需要一些系统必须的依赖库
yum install -y net-snmp net-snmp-devel curl curl-devel libxm12 libevent libevent-devel

# 创建用户和组。用于启动zabbix守护进程
groupadd zabbix
useradd -g zabbix zabbix
```

```sh
# 创建数据库
# 如果root有密码
mariadb -u root -p -e "create database zabbix character set utf8 collate utf8_bin"
mariadb -u root -p -e "grant all on zabbix.* to zabbix@localhost identified by 'test'"
mariadb -u root -p -e "set global log_bin_trust_function_creators = 1"
# 如果root没有密码
mariadb -e "create database zabbix character set utf8 collate utf8_bin"
mariadb -e "grant all on zabbix.* to zabbix@localhost identified by 'password'"
mariadb -e "set global log_bin_trust_function_creators = 1"

# 导入模板。需要输入刚才设置的zabbix的密码
mariadb -u zabbix -p -D zabbix < /usr/share/zabbix-server/mysql/schema.sql
mariadb -u zabbix -p -D zabbix < /usr/share/zabbix-server/mysql/images.sql
mariadb -u zabbix -p -D zabbix < /usr/share/zabbix-server/mysql/data.sql
```

- 修改配置文件`/etc/zabbix/zabbix_server.conf`

```
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=password

# 监听的ip，也就是对那些ip开放
ListenIP=0.0.0.0
ListenPort=10051

# 日志目录
LogFile=/var/log/zabbix_server.log

# pollers的进程数量。zabbix server启动时会启动pollers
StartPollers=5
# Trappers的进程数量。zabbix server启动时会启动Trappers。agentd为主动模式时，这个值可以设置大一些
StartTrappers=10
# Discoverers的进程数量。zabbix server启动时会启动Discoverers。如果zabbix监控报告Discoverers进程忙碌时，需要提高该值
StartDiscoverers=10

# 存放脚本的目录
# AlertScriptsPath=/usr/local/zabbix/share/zabbix/alertscripts
```

- 启动zabbix_server
