
<!-- mtoc-start -->

* [zabbix](#zabbix)
  * [安装](#安装)
    * [yum 安装](#yum-安装)
      * [安装zabbix-server。服务器安装](#安装zabbix-server服务器安装)
      * [安装zabbix-agent。被监控的主机安装](#安装zabbix-agent被监控的主机安装)
  * [监控配置](#监控配置)
    * [基本监控操作](#基本监控操作)
    * [自定义监控项](#自定义监控项)
  * [zabbix-proxy部署](#zabbix-proxy部署)
    * [安装](#安装-1)
    * [配置](#配置)
    * [agent端配置](#agent端配置)
  * [自动发现](#自动发现)
  * [未读](#未读)
    * [Centos7 部署 Zabbix 5.4 高可用集群](#centos7-部署-zabbix-54-高可用集群)

<!-- mtoc-end -->

# zabbix

- [咸鱼运维杂谈：企业级监控方案——zabbix！（上）](https://mp.weixin.qq.com/s?__biz=MzkzNzI1MzE2Mw==&mid=2247484319&idx=1&sn=5c43df4716f63c35433c6325e1acf0a5&chksm=c29303dbf5e48acd55d771cdf50009f456cbf03ab031438198d0bcd0a4f2e844171c3b5ddabc&scene=21#wechat_redirect)

- [咸鱼运维杂谈：企业级监控方案——zabbix！（下）](https://mp.weixin.qq.com/s?__biz=MzkzNzI1MzE2Mw==&mid=2247484368&idx=1&sn=ebb8d885e42bb32559b617c8763fd92a&chksm=c2930394f5e48a822f647a60b5369c01e04c28d7cfc29547dfdf6e74e6c2fe0e415df5173b20&cur_album_id=2890712891962802179&scene=189#wechat_redirect)

- 主要组件

    - zabbix server是核心组件，所有配置信息、统计信息、操作数据的核心存储。
        - 负责接受客户端发送的报告和信息

    - zabbix数据存储：收集到的数据存储在数据库中。如mysql、oracle、sqlite

    - zabbix web界面：gui接口。通常（但不一定）与zabbix server运行在同一台物理机

    - zabbix proxy代理服务器（可选组件）：分布式监控，代理服务器代替zabbix server收集性能数据和可用性数据，汇总后统一发往zabbix server

    - zabbix agent监控代理：部署到被监控主机上，负责收集数据

    ![avatar](./Pictures/prometheus/zabbix架构.avif)

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

- zabbix有两种监控方式

    - 主动模式：agent主动向zabbix-server请求与自己相关的监控项配置，主动将监控数据发送给server（有proxy情况下发送给proxy）

    - 被动模式：server向agent请求获取监控数据，agent收到请求，之后获取数据并响应给server

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

### yum 安装

#### 安装zabbix-server。服务器安装

```sh
rpm -Uvh https://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-2.el7.noarch.rpm
yum clean all
yum install zabbix-server-mysql zabbix-web-mysql -y
```

- 安装数据库
```sh
yum install mariadb-server -y
systemctl start mariadb
systemctl enable mariadb
```

- 初始化数据库
```sh
mysql -uroot -p
Enter password: #按回车即可

#创建一个名为zabbix的库并设置为utf8的字符编码格式
create database zabbix character set utf8 collate utf8_bin;

#创建用户并授权设置密码为password
create user zabbix@localhost identified by 'password';
grant all privileges on zabbix.* to zabbix@localhost;
flush privileges;
quit;
```

- 导入表
```sh
zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix
```

- 对zabbix-server进行配置数据库密码
```sh
vim /etc/zabbix/zabbix_server.conf
DBPassword=password

systemctl start zabbix-server.service
systemctl enable zabbix-server.service
```

- 进行前端php配置
```sh
vim /etc/httpd/conf.d/zabbix.conf

#这里我们修改时区为亚洲上海即可
php_value date.timezone Asia/Shanghai

#启动服务
systemctl start httpd
systemctl enable httpd
```

- 至此完成zabbix 服务端的部署，我们可以在浏览器输入：ip/zabbix 进行访问。
    - 一开始连接时输入的是数据库密码
    - 之后登陆是，Admin为用户，zabbix为密码


#### 安装zabbix-agent。被监控的主机安装

- 安装zabbix-agent
```sh
rpm -Uvh https://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-2.el7.noarch.rpm
yum install -y zabbix-agent
```

- 修改配置
```sh
vim /etc/zabbix/zabbix_agentd.conf

###修改成zabbix-server端的ip地址###
Server=192.168.244.141

###修改成zabbix-server端的ip地址###
ServerActive=192.168.244.141

###修改成zabbix-agent端即本机的ip地址，不要用127.0.0.1###
Hostname=192.168.244.128
```

- 启动服务
```sh
systemctl start zabbix-agent.service
systemctl enable zabbix-agent.service
```

- 关闭防火墙和selinux
```sh
setenforce 0
systemctl stop firewalld
```

## 监控配置

### 基本监控操作

- 点击web界面的右上角的`用户`可以设置中文

- 在server端的web监控添加一些监控项来对agent1和agent2来进行监控

- 1.先将agent1、agent2添加到监控主机中

    - 配置—>主机—>创建主机

    ![avatar](./Pictures/prometheus/zabbix添加到监控主机1.avif)
    ![avatar](./Pictures/prometheus/zabbix添加到监控主机2.avif)
    ![avatar](./Pictures/prometheus/zabbix添加到监控主机3.avif)

- 2.接下来添加一下监控项对其进行监控

    - 监控项——>(左上角)创建监控项

    ![avatar](./Pictures/prometheus/zabbix添加监控项1.avif)
    ![avatar](./Pictures/prometheus/zabbix添加监控项2.avif)

    - 添加完后我们可以看到agent1、agent2监控项数量有了新增
    ![avatar](./Pictures/prometheus/zabbix添加监控项3.avif)

    - 别忘了链接至模板，否则获取不到zabbix agent状态，并且图标为灰色，而在zabbix-server端使用zabbix_get可以正常获取到数据。在正常情况下应为绿色或红色
    ![avatar](./Pictures/prometheus/zabbix添加监控项4.avif)
    ![avatar](./Pictures/prometheus/zabbix添加监控项5.avif)

- 3.对监控项添加图形界面

    - 点击名称——>图形——>创建图形

    ![avatar](./Pictures/prometheus/zabbix对监控项添加图形界面1.avif)
    ![avatar](./Pictures/prometheus/zabbix对监控项添加图形界面2.avif)
    ![avatar](./Pictures/prometheus/zabbix对监控项添加图形界面3.avif)

    - 添加完毕后我们在上方菜单栏：监测——>图形——>选择监控主机
    ![avatar](./Pictures/prometheus/zabbix对监控项添加图形界面4.avif)

### 自定义监控项

- 在zabbix-agent端配置：

    - 添加`iostat | grep sda | awk '{print $2}'`命令行
    ```sh
    vim /etc/zabbix/zabbix_agentd.d/userparameter_io.conf

    #sda_io为自定义监控项名字，逗号后面为得出数据的语句
    UserParameter=sda_io,iostat | grep sda | awk '{print $2}'
    ```

    - 重启zabbix-agent
    ```sh
    systemctl restart zabbix-agent
    ```

- 在server端进行验证，需要安装zabbix-get工具

    ```sh
    yum install zabbix-get.x86_64 -y
    #-s：指定agent ip地址 #-k：指定监控项名字
    zabbix_get -s 192.168.244.128 -k sda_io
    ```

    - 3.对监控项添加图形界面

        - 点击名称——>图形——>创建图形

        ![avatar](./Pictures/prometheus/zabbix对自定义监控项添加图形界面1.avif)
        ![avatar](./Pictures/prometheus/zabbix对自定义监控项添加图形界面2.avif)
        ![avatar](./Pictures/prometheus/zabbix对自定义监控项添加图形界面3.avif)

## zabbix-proxy部署

- 在一些分布式环境中，一台zabbix-server可能要监控许多agent，这会导致zabbix-server压力过大，容易出现资源紧张的情况

- zabbix-proxy 可以代替 zabbix-server 收集性能和可用性数据，然后把数据汇报给 zabbix-server，在一定程度上分担了zabbix-server 的压力。

- zabbix-proxy使用场景：

    - 监控远程区域主机
    - 监控本地网络不稳定区域
    - 当 zabbix 监控上千设备时,使用它来减轻 server 的压力
    - 便于分布式监控的维护

### 安装

- 安装
```sh
yum install zabbix-proxy-mysql -y
yum install mariadb-server -y
```

- 数据库配置
```sh
systemctl start mariadb
systemctl enable mariadb
mysql -uroot -p
Enter password: #按回车即可

#创建数据库zabbix_proxy，并设置为utf-8编码
create database zabbix_proxy character set utf8 collate
utf8_bin;

#创建用户并设置密码为password
create user zabbix_proxy@localhost identified by 'password';

#为用户授权
grant all privileges on zabbix_proxy.* to
zabbix_proxy@localhost;

#刷新权限
flush privileges;
```

- 数据库配置好之后我们导入表

```sh
zcat /usr/share/doc/zabbix-proxy-mysql*/schema.sql.gz | mysql -uzabbix_proxy -p zabbix_proxy
Enter password:
# 密码是password
```

- 验证

```sh
mysql -uzabbix_proxy -p zabbix_proxy

show tables;
+----------------------------+
| Tables_in_zabbix_proxy     |
+----------------------------+
| acknowledges               |
| actions                    |
| alerts                     |
| application_discovery      |
| application_prototype      |
| application_template       |
| applications               |
| auditlog                   |
| auditlog_details           |
| autoreg_host               |
```

### 配置

- 我们接下来修改proxy1的配置文件，让它能够与zabbix-server连接起来

    ```sh
    # 先备份配置文件，又因为配置文件里有很多注释和空行，我们可以先使用grep过滤掉
    cp /etc/zabbix/zabbix_proxy.conf /etc/zabbix/zabbix_proxy.conf.bak
    grep -Ev "^$|#" /etc/zabbix/zabbix_proxy.conf > /etc/zabbix/1.conf
    mv /etc/zabbix/1.conf /etc/zabbix/zabbix_proxy.conf
    ```

- vim /etc/zabbix/zabbix_proxy.conf

    ```sh
    #zabbix server服务器的地址或主机名，主动模式只能写一个IP，被动可以写多个IP
    Server=192.168.110.4
    # 本机ip
    Hostname=192.168.110.5
    #允许zabbix server执行远程命令
    EnableRemoteCommands=1
    LogFile=/var/log/zabbix/zabbix_proxy.log
    LogFileSize=0
    PidFile=/var/run/zabbix/zabbix_proxy.pid
    #数据库服务器地址
    DBHost=localhost
    #使用的数据库名称
    DBName=zabbix_proxy
    #连接数据库的用户名称
    DBUser=zabbix_proxy
    #数据库用户密码
    DBPassword=password
    ```

### agent端配置

- 同理，先备份配置文件再去除注释和空行

    ```sh
    cp /etc/zabbix/zabbix_agentd.conf /etc/zabbix/zabbix_agentd.conf.bak
    grep -Ev "^$|#" /etc/zabbix/zabbix_agentd.conf >  /etc/zabbix/1.conf
    mv /etc/zabbix/1.conf /etc/zabbix/zabbix_agentd.conf
    ```

- vim /etc/zabbix/zabbix_agentd.conf
    ```sh
    PidFile=/var/run/zabbix/zabbix_agentd.pid
    LogFile=/var/log/zabbix/zabbix_agentd.log
    LogFileSize=0
    # zabbix_proxy主机的ip
    Server=192.168.110.5
    ServerActive=192.168.110.5

    Hostname=192.168.110.6
    Include=/etc/zabbix/zabbix_agentd.d/*.conf
    ```

- 在server的web界面进行代理程序添加
    ![avatar](./Pictures/prometheus/zabbix添加代理程序1.avif)
    - 代理程序的ip为proxy组件所在主机的ip，192.168.110.5

    ![avatar](./Pictures/prometheus/zabbix添加代理程序2.avif)
    ![avatar](./Pictures/prometheus/zabbix添加代理程序3.avif)
    ![avatar](./Pictures/prometheus/zabbix添加代理程序4.avif)
    ![avatar](./Pictures/prometheus/zabbix添加代理程序5.avif)

## 自动发现

- [咸鱼运维杂谈：zabbix 自动发现](https://mp.weixin.qq.com/s?__biz=MzkzNzI1MzE2Mw==&mid=2247486575&idx=1&sn=474771d341fec657713ae095960aea5c&chksm=c2930c2bf5e4853d2e680abf0acd5b6401d97972be1b56171504e1b17d77ddb665caa13add40&cur_album_id=2890712891962802179&scene=190#rd)

## 未读

### [Centos7 部署 Zabbix 5.4 高可用集群](https://mp.weixin.qq.com/s/xfhIJ_gZ06izE5jGhbcK2g)
