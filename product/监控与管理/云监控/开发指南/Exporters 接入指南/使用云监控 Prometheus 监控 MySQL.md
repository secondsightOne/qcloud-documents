## 使用云监控 Prometheus 监控 MySQL 

MySQL Exporter 是社区专门为采集 MySQL/MariaDB 数据库监控指标而设计开发，通过 Exporter 上报核心的数据库指标，用于异常报警和监控大盘展示，云监控 Prometheus 提供了与 MySQL Exporter 集成及提供开箱即用的 Grafana 监控大盘。

目前，Exporter 支持高于5.6版本的 MySQL 和  高于 10.1版本的 MariaDB 。在 MySQL/MariaDB 低于 5.6 版本时，有一些监控指标可能无法采集。

> ？ 为了方便安装管理 Exporter，这里推荐使用腾讯云容器服务来统一管理。

## 前提条件

1. 在 Proemtheus 实例对应地域及私有网络（VPC）下，腾讯云容器服务中创建 Kubernetes 集群。详情请参考 [腾讯云容器服务—托管版集群](https://cloud.tencent.com/document/product/457/32189#.E4.BD.BF.E7.94.A8.E6.A8.A1.E6.9D.BF.E6.96.B0.E5.BB.BA.E9.9B.86.E7.BE.A4.3Cspan-id.3D.22templatecreation.22.3E.3C.2Fspan.3E)。
2. 登录 [云监控 Prometheus 控制台](https://console.cloud.tencent.com/monitor/prometheus)。
3. 在对应 Prometheus 实例 >【集成容器服务】中找到对应容器集群完成集成操作，详情请参考 [Agent 管理](https://cloud.tencent.com/document/product/248/48859)。

下列所有的操作都是在对应的容器服务集群下，并且基于 YAML 的方式创建资源。如图：

![](https://main.qcloudimg.com/raw/17aa7535a56c1e1e099bbed5bc2c375a.png)

> ？ 在操作前请确保该集群相应的命名空间已提前创建。如需创建命名空间请参考 [容器服务—管理命名空间](https://cloud.tencent.com/document/product/1141/41803)。

### 数据库授权

因为 MySQL Exporter 是通过查询数据库中状态数据来对其进行监控，所以需要对对应的数据库实例进行授权。账号和密码需根据实际情况而定，授权示例如下：
![](https://main.qcloudimg.com/raw/c60161e3f99b7fe6b36e4dc353d53ae7.png)
或者通过如下命令进行授权：

```
CREATE USER 'exporter'@'ip' IDENTIFIED BY 'XXXXXXXX' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'ip';
```

> ? 建议为该用户设置最大连接数限制，以避免因监控数据抓取对数据库带来影响。但并非所有的数据库版本中都可以生效，如MariaDB 10.1 版本不支持最大连接数设置，则无法生效。详情可参考 [MariaDB 说明](https://mariadb.com/kb/en/create-user/#resource-limit-options)。

### Exporter 部署

#### 使用 Secret 管理 MySQL 连接串

使用 Kubernetes 的 Secret 来管理连接串，并对连接串进行加密处理，在启动 MySQL Exporter 的时候直接使用 Secret Key，需要调整对应的 `连接串`，YAML 配置示例如下：

```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret-test
  namespace: mysql-demo
type: Opaque
stringData:
  datasource: "user:password@tcp(ip:port)/"  #对应 MySQL 连接串信息
```

#### 部署 MySQL Exporter

通过【工作负载】>【Deployment】进入 `Deployment` 管理页面，选择对应的 `命名空间` 来进行部署服务，可以通过控制台的方式创建。下面以 YAML 的方式部署 Exporter， 配置示例如下：

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  labels:
    k8s-app: mysql-exporter  # 根据业务需要调整成对应的名称，建议加上 MySQL 实例的信息
  name: mysql-exporter  # 根据业务需要调整成对应的名称，建议加上 MySQL 实例的信息
  namespace: mysql-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: mysql-exporter  # 根据业务需要调整成对应的名称，建议加上 MySQL 实例的信息
  template:
    metadata:
      labels:
        k8s-app: mysql-exporter  # 根据业务需要调整成对应的名称，建议加上 MySQL 实例的信息
    spec:
      containers:
      - env:
        - name: DATA_SOURCE_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-secret-test
              key: datasource
        image: ccr.ccs.tencentyun.com/k8s-comm/mysqld-exporter:0.12.1
        imagePullPolicy: IfNotPresent
        name: mysql-exporter
        ports:
        - containerPort: 9104
          name: metric-port 
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

#### 验证

通过【工作负载】>【Deployment】进入部署 MySQL Exporter时创建的`Deployment`，进入其【日志】页面可以看到 Exporter 成功启动，并暴露对应的访问地址，如图：
![](https://main.qcloudimg.com/raw/353be171da1dbdff76735a4b67a2055d.png)

【远程登录】到上面创建的 `POD` 中，在命令行中 curl 对应 Exporter 暴露的地址，可以正常得到对应的 MySQL 指标，如发现未能得到对应的数据，请检查一下 `连接串` 是否正确，具体如下：

```
curl localhost:9104/metrics
```

![](https://main.qcloudimg.com/raw/cc2feadce888950b8a94c9f7ae272abd.png)

### 添加采取任务

1. 登录 [云监控 Prometheus 控制台](https://console.cloud.tencent.com/monitor/prometheus)，选择对应 Prometheus 实例进入管理页面。
2. 通过集成容器服务列表点击【集群 ID】进入到容器服务集成管理页面。
3. 通过服务发现添加 `Pod Monitor` 来定义 Prometheus 抓取任务，YAML 配置示例如下：

```
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  # 填写一个唯一名称
  name: mysql-exporter
  # namespace固定，不要修改
  namespace: cm-prometheus
spec:
  podMetricsEndpoints:
  - interval: 30s
    # 填写pod yaml中Prometheus Exporter对应的Port的Name
    port: metric-port
    # 填写Prometheus Exporter对应的Path的值，不填默认/metrics
    path: /metrics
    relabelings:
    - action: replace
      sourceLabels: 
      - instance
      regex: (.*)
      targetLabel: instance
      replacement: 'crs-xxxxxx' # 调整成对应的 MySQL 实例 ID
    - action: replace
      sourceLabels: 
      - instance
      regex: (.*)
      targetLabel: ip
      replacement: '1.x.x.x' # 调整成对应的 MySQL 实例 IP
  # 选择要监控pod所在的namespace
  namespaceSelector:
    matchNames:
    - mysql-demo
  # 填写要监控pod的Label值，以定位目标pod
  selector:
    matchLabels:
      k8s-app: mysql-exporter
```

### 查看监控

在对应 Prometheus 实例 >【集成中心】中找到 `MySQL` 监控，安装对应的 Grafana Dashboard 即可开启 MySQL 监控大盘，查看实例相关的监控数据，如图：
![](https://main.qcloudimg.com/raw/bb67a605300a2abfa707fad036a489d5.png)

### 告警以及接入

腾讯云 Prometheus 托管服务内置了一些 MySQL 数据库的报警策略模板，可根据业务实际的情况调整对应的阈值来添加告警策略。
![MySQL Exporter alert](https://main.qcloudimg.com/raw/abb8680e123473f78c350201acac1abd.jpg)

### MySQL Exporter 采集参数详解

MySQL Exporter 使用各种 `Collector` 来控制采集数据的启停，具体参数如下：

| 名称                                                     | MySQL 版本 | 描述                                                         |
| -------------------------------------------------------- | ---------- | ------------------------------------------------------------ |
| collect.auto_increment.columns                           | 5.1        | 在 information_schema 中采集 auto_increment 和最大值。       |
| collect.binlog_size                                      | 5.1        | 采集所有注册的 binlog 文件大小。                             |
| collect.engine_innodb_status                             | 5.1        | 从 SHOW ENGINE INNODB STATUS 中采集状态数据。                |
| collect.engine_tokudb_status                             | 5.6        | 从 SHOW ENGINE TOKUDB STATUS 中采集状态数据。                |
| collect.global_status                                    | 5.1        | 从 SHOW GLOBAL STATUS (默认开启) 中采集状态数据。            |
| collect.global_variables                                 | 5.1        | 从 SHOW GLOBAL VARIABLES (默认开启) 中采集状态数据。         |
| collect.info_schema.clientstats                          | 5.5        | 如果设置了 userstat=1, 设置成 true 来开启用户端数据采集。    |
| collect.info_schema.innodb_metrics                       | 5.6        | 从 information_schema.innodb_metrics 中采集监控数据。        |
| collect.info_schema.innodb_tablespaces                   | 5.7        | 从 information_schema.innodb_sys_tablespaces 中采集监控数据。 |
| collect.info_schema.innodb_cmp                           | 5.5        | 从 information_schema.innodb_cmp 中采集 InnoDB 压缩表的监控数据。 |
| collect.info_schema.innodb_cmpmem                        | 5.5        | 从 information_schema.innodb_cmpmem 中采集 InnoDB buffer pool compression 的监控数据。 |
| collect.info_schema.processlist                          | 5.1        | 从 information_schema.processlist 中采集线程状态计数的监控数据。 |
| collect.info_schema.processlist.min_time                 | 5.1        | 线程可以被统计所维持的状态的最小时间。 (默认: 0)             |
| collect.info_schema.query_response_time                  | 5.5        | 如果 query_response_time_stats 被设置成 ON，采集查询相应时间的分布。 |
| collect.info_schema.replica_host                         | 5.6        | 从 information_schema.replica_host_status 中采集状态数据。   |
| collect.info_schema.tables                               | 5.1        | 从 information_schema.tables 中采集状态数据。                |
| collect.info_schema.tables.databases                     | 5.1        | 设置需要采集表状态的数据库, 或者设置成 '`*`' 来采集所有的。  |
| collect.info_schema.tablestats                           | 5.1        | 如果设置了 userstat=1, 设置成 true 来采集表统计数据。        |
| collect.info_schema.schemastats                          | 5.1        | 如果设置了 userstat=1, 设置成 true 来采集 schema 统计数据。  |
| collect.info_schema.userstats                            | 5.1        | 如果设置了 userstat=1, 设置成 true 来采集用户统计数据。      |
| collect.perf_schema.eventsstatements                     | 5.6        | 从 performance_schema.events_statements_summary_by_digest 中采集监控数据。 |
| collect.perf_schema.eventsstatements.digest_text_limit   | 5.6        | 设置正常文本语句的最大长度。 (默认: 120)                     |
| collect.perf_schema.eventsstatements.limit               | 5.6        | 事件语句的限制数量. (默认: 250)                              |
| collect.perf_schema.eventsstatements.timelimit           | 5.6        | 限制事件语句 'last_seen' 可以保持多久， 单位为秒。 (默认: 86400) |
| collect.perf_schema.eventsstatementssum                  | 5.7        | 从 performance_schema.events_statements_summary_by_digest summed 中采集监控数据。 |
| collect.perf_schema.eventswaits                          | 5.5        | 从 performance_schema.events_waits_summary_global_by_event_name 中采集监控数据。 |
| collect.perf_schema.file_events                          | 5.6        | 从 performance_schema.file_summary_by_event_name 中采集监控数据。 |
| collect.perf_schema.file_instances                       | 5.5        | 从 performance_schema.file_summary_by_instance 中采集监控数据。 |
| collect.perf_schema.indexiowaits                         | 5.6        | 从 performance_schema.table_io_waits_summary_by_index_usage 中采集监控数据。 |
| collect.perf_schema.tableiowaits                         | 5.6        | 从 performance_schema.table_io_waits_summary_by_table 中采集监控数据。 |
| collect.perf_schema.tablelocks                           | 5.6        | 从 performance_schema.table_lock_waits_summary_by_table 中采集监控数据。 |
| collect.perf_schema.replication_group_members            | 5.7        | 从 performance_schema.replication_group_members 中采集监控数据。 |
| collect.perf_schema.replication_group_member_stats       | 5.7        | 从 from performance_schema.replication_group_member_stats 中采集监控数据。 |
| collect.perf_schema.replication_applier_status_by_worker | 5.7        | 从 performance_schema.replication_applier_status_by_worker 中采集监控数据。 |
| collect.slave_status                                     | 5.1        | 从 SHOW SLAVE STATUS (默认开启) 中采集监控数据。             |
| collect.slave_hosts                                      | 5.1        | 从 SHOW SLAVE HOSTS 中采集监控数据。                         |
| collect.heartbeat                                        | 5.1        | 从 [heartbeat](#heartbeat) 中采集监控数据。                  |
| collect.heartbeat.database                               | 5.1        | 数据库心跳检测的数据源。(默认: heartbeat)                    |
| collect.heartbeat.table                                  | 5.1        | 表心跳检测的数据源。 (默认: heartbeat)                       |
| collect.heartbeat.utc                                    | 5.1        | 对当前的数据库服务器使用 UTC 时间戳 (`pt-heartbeat` is called with `--utc`)。(默认: false) |

### 全局配置参数

| 名称                       | 描述                                                         |
| -------------------------- | ------------------------------------------------------------ |
| config.my-cnf              | 用来读取数据库认证信息的配置文件 .my.cnf 位置。 (默认: `~/.my.cnf`) |
| log.level                  | 日志级别。 (默认: info)                                      |
| exporter.lock_wait_timeout | 为链接设置 lock_wait_timeout (单位：秒) 以避免对元数据的锁时间太长。(默认: 2) |
| exporter.log_slow_filter   | 添加 log_slow_filter 以避免抓取的慢查询被记录。  提示: 不支持 Oracle MySQL。 |
| web.listen-address         | web 端口监听地址。                                           |
| web.telemetry-path         | metrics 接口路径。                                           |
| version                    | 打印版本信息。                                               |

### heartbeat 心跳检测

如果开启了 `collect.heartbeat`， mysqld_exporter 会通过心跳检测机制抓取复制延迟数据。
