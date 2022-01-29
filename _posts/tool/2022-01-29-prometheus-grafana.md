---
layout: post
title: "Prometheus&Grafana的使用"
date: 2021-01-29 16:06:27
categories: Tool
---

## 一、Prometheus简介

Prometheus是一个开源监控和警报系统。Prometheus将其指标收集并存储在时序数据库中，用于灵活展示当前监控应用的状态。

Prometheus其对传统监控系统的测试和告警模型进行了彻底的颠覆，形成了基于<span style="color:red">中央化</span>的规则计算、统一分析和告警的新模型。 相比于传统监控系统，Prometheus具有以下优点

- 易于管理

    Prometheus核心部分只有一个单独的二进制文件，不存在任何的第三方依赖（数据库，缓存等）。因此不会有潜在级联故障的风险

    Prometheus基于<span style="color:red">Pull</span>模型的架构方式，可以在任何地方（本地电脑，开发环境，测试环境）搭建我们的监控系统

    对于一些复杂的情况，还可以使用Prometheus<span style="color:red">服务发现（Service Discovery）</span>的能力动态管理监控目标

- 监控服务的内部运行状态

​		Prometheus有丰富的Client库，可以轻松的在应用中添加对Prometheus的支持，从而可以获取服务和应用内部真正的运行状态

- 强大的数据模型

​		所有采集的监控数据均以指标和标签的形式保存在内置的<span style="color:red">时序数据库（TSDB）</span>当中。

- 强大的查询语言PromQL

    Prometheus内置了一个强大的数据查询语言PromQL。通过 PromQL可以实现对监控数据的查询、聚合

- 高效

    Prometheus可以高效地处理收集的数据，对于单一实例而言它可以处理数以百万的监控指标和每秒处理数十万的数据

- 可扩展性

    去中心化设计，便于扩展部署，任务量过大时可以对其进行扩展

## 二、Prometheus的架构

![普罗米修斯架构](/img/prometheus-grafana/普罗米修斯架构.png)

- 采集层

    | 组件        | 作用                                                         |
    | ----------- | ------------------------------------------------------------ |
    | Pushgateway | 当Prometheus与监控服务不在同一网段，或者监控的是任务类型的数据（可能只运行一会）使用将数据推送到网关的方式 |
    | Exporters   | 主要用来采集数据，并通过HTTP服务的形式暴露给Prometheus Server，Prometheus Server通过访问该Exporter提供的接口，即可获取到需要采集的监控数据 |

    

- 存储计算层

    | 组件              | 作用                                                         |
    | ----------------- | ------------------------------------------------------------ |
    | Prometheus Server | 面包含了存储引擎和计算引擎                                   |
    | Retrieval         | 组件为取数组件，它会主动从pushgateway或者exporter拉取指标数据 |
    | Service Discovery | 可以动态发现要监控的目标                                     |
    | TSDB              | 数据核心存储与查询                                           |
    | HTTP Server       | 对外提供HTTP服务                                             |

- 应用层

    | 组件         | 作用                                                      |
    | ------------ | --------------------------------------------------------- |
    | AlertManager | 报警设置，可以设置email、webhook等等告警通知              |
    | 数据可视化   | 可于各种数据可视化组件集成例如grafana，达到专业的展示效果 |

## 三、Prometheus及其组件的安装

### Prometheus Server的安装

1、从[官网地址][https://prometheus.io/]下载安装包

```shell
wget https://github.com/prometheus/prometheus/releases/download/[version]/prometheus-[version].linux-amd64.tar.gz
```

2、解压

```shell
tar -zxvf prometheus-[version].linux-amd64.tar.gz -C /opt/module/
mv /opt/module/prometheus-[version].linux-amd64 /opt/module/prometheus
```

3、修改配置文件prometheus.yml

```yaml
# 全局设置
global:
  scrape_interval: 15s     # 配置拉取数据的时间间隔，默认为1分钟。
  evaluation_interval: 15s # 规则验证(生成alert)的时间间隔，默认为1分钟。
  # scrape_timeout is set to the global default (10s).

# 报警配置，没有可视化界面，也可使用grafana的报警机制
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# 报警规则
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"


# 修改如下配置，此台服务器作为Prometheus Server
scrape_configs:
  - job_name: "prometheus"  # 监控作业的名称
    static_configs:         # 表示静态目标配置，就是固定从某个target拉取数据
      - targets: ["hostName:9090"]  # 会从[http://hostname:9090/metrics]上拉取数据
```

4、添加系统服务，或者直接启动

- 添加系统服务

```shell
vim /etc/systemd/system/prometheus.service
```

```txt
[Unit]
Description=Prometheus Monitoring System
Documentation=Prometheus Monitoring System
 
[Service]
ExecStart=/opt/module/prometheus/prometheus \
  --config.file=/opt/module/prometheus/prometheus.yml \
  --web.listen-address=:9090
 
[Install]
WantedBy=multi-user.target
```

- 启动

```shell
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus
```

- 访问【http://hostname:9090/】能打开web界面即可

### 安装Exporter（选择性安装）

在Prometheus的架构设计中，Prometheus Server主要负责数据的收集，存储并且对外提供数据查询支持，而实际的监控样本数据的收集则是由Exporter直接上报，或者Pushgateway收集上报完成。因此为了能够监控到某些东西，如主机CPU使用率，需要使用Exporter。Prometheus周期性的从Exporter暴露的HTTP服务地址（通常是/metrics）拉取监控样本数据。Exporter可以是一个相对开放的概念，有很多实现的Exporter如mysqld_exporter。其可以是一个独立运行的程序独立于监控目标以外，也可以是直接内置在监控目标中。只要能够向Prometheus提供标准格式的监控样本数据即可。

#### 安装node_exporter

能够采集到主机的运行指标如CPU, 内存，磁盘等信息。我们可以使用Node Exporter。Node Exporter同样采用Golang编写，并且不存在任何的第三方依赖，只需要下载，解压即可运行。

1、从[官网地址][https://prometheus.io/]下载安装包到需要监控的主机

```shell
wget https://github.com/prometheus/node_exporter/releases/download/[version]/node_exporter-[version].linux-amd64.tar.gz
```

2、解压

```shell
tar -zxvf node_exporter-[version].linux-amd64.tar.gz -C /opt/module/
mv /opt/module/node_exporter-[version].linux-amd64 /opt/module/node_exporter
```

3、添加系统服务，或者直接启动

- 添加系统服务

```shell
vim /etc/systemd/system/node_exporter.service
```

```txt
[Unit]
Description=node_exporter
After=network.target 

[Service]
User=root
Group=root
ExecStart=/opt/module/node_exporter/node_exporter\
          --web.listen-address=:9100
          
[Install]
WantedBy=multi-user.target
```

- 启动服务

```shell
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter
systemctl status node_exporter
```

4、配置Prometheus Server，修改prometheus.yml

```yaml
# 添加如下信息
scrape_configs:
  - job_name: "node_exporter"
    static_configs:
    - targets: 
      - "targetHostName:9100"
```

5、重启Prometheus Server，查看http://prometheus_server:9090，下status/targets

#### 安装mysqld_exporter

1、从[官网地址][https://prometheus.io/]下载安装包到需要监控的主机

```shell
wget https://github.com/prometheus/mysqld_exporter/releases/download/[version]/mysqld_exporter-[version].linux-amd64.tar.gz
```

2、解压

```shell
tar -zxvf mysqld_exporter-[version].linux-amd64.tar.gz -C /opt/module/
mv /opt/module/mysqld_exporter-[version].linux-amd64 /opt/module/mysqld_exporter
```

3、配置，创建my.conf配置文件

```txt
[client]
host=[hostName]
port=[port]
user=[monitorUser]
password=[monitorPassword]
```

4、添加系统服务，或者直接启动

```shell
nohup /opt/module/mysqld_exporter/mysqld_exporter --web.listen-address=":9104" --config.my-cnf=/opt/module/mysqld_exporter/my.conf &
```

5、配置Prometheus Server，修改prometheus.yml

```yaml
# 添加如下信息
scrape_configs:
  - job_name: "mysqld_exporter"
    static_configs:
    - targets: 
      - "targetHostName:9104"
```

6、重启Prometheus Server，查看http://prometheus_server:9090,下status/targets

### 安装 Pushgateway（选择性安装）

Prometheus在正常情况下是采用拉模式从exporter（比如专门监控主机的NodeExporter）拉取监控数据，但当遇到定时性任务，或者网络不在同一网段上时需要采用Pushgateway。将需要监控的数据上传到Pushgateway，之后再由Prometheus Server拉取数据。

1、从[官网地址][https://prometheus.io/]下载安装包到需要监控的主机

```shell
wget https://github.com/prometheus/pushgateway/releases/download/[version]/pushgateway-[version].linux-amd64.tar.gz
```

2、解压

```shell
tar -zxvf pushgateway-[version].linux-amd64.tar.gz -C /opt/module/
mv /opt/module/pushgateway-[version].linux-amd64 /opt/module/pushgateway
```

3、添加系统服务，或者直接启动

- 添加系统服务

```shell
vim /etc/systemd/system/pushgateway.service
```

```txt
[Unit]
Description=pushgateway
After=network.target 

[Service]
User=root
Group=root
ExecStart=/opt/module/pushgateway/pushgateway\
          --web.listen-address=:9091
          
[Install]
WantedBy=multi-user.target
```

- 启动服务

```shell
systemctl daemon-reload
systemctl enable pushgateway
systemctl start pushgateway
systemctl status pushgateway
```

4、配置Prometheus Server，修改prometheus.yml

```yaml
# 添加如下信息
scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true  #加上此配置exporter节点上传数据中的一些标签将不会被pushgateway节点的相同标签覆盖
    static_configs:
      - targets: 
        - "hostName:9091"
```

5、重启Prometheus Server，查看http://prometheus_server:9090，下status/targets

### Alertmanager模块
安装十分简单，但是没有可视化界面提供实时编辑，灵活读较低，编写较为麻烦，可使用Grafana的报警


## 四、PromQL使用

Prometheus通过指标名称（metrics name）以及对应的一组标签（label set）唯一定义一条时间序列。指标名称反映了监控样本的基本标识，而label则在这个基本特征上为采集到的数据提供了多种特征维度。用户可以基于这些特征维度<span style="color:red">过滤，聚合，统计</span>从而产生新的计算后的一条时间序列。PromQL是Prometheus内置的数据查询语言，其提供对时间序列数据丰富的查询，聚合以及逻辑运算能力的支持。并且被广泛应用在Prometheus的日常应用当中，包括对数据查询、可视化、告警处理当中。

### 基本用法

- 查询时间序列[每个时间序列包含单个样本，它们共享相同的时间戳。即返回值只会包含该时间序列中的最新的一个样本值]

```promql
# 查询mysql链接超时时长，此指标只需要查询当前状态，时间段内的是没有意义的
mysql_global_variables_connect_timeout
mysql_global_variables_connect_timeout{}
```

- 过滤标签的匹配模式

```promql
# 等值查询
mysql_global_variables_connect_timeout{instance="centos161:9104"}
# 不等值查询
mysql_global_variables_connect_timeout{instance!="centos161:9104"}
# 正则表达式查询
mysql_global_variables_connect_timeout{instance=~".*:9104"}
```

- 范围查询，一段时间序列的集合

```promql
# 查询5min内的所有IO设备的取样
node_disk_io_now[5m]
```

- 时间位移操作

```promql
# 例如当前是12点，此查询查询的是11:50-11:55的数据
node_disk_io_now[5m] offset 5m
```

- 聚合操作

支持sum（求和）、min（最小值）、max（最大值）、avg（平均值）、stddev（标准差）、stdvar（标准差异）、count（计数）、count_values（对value进行计数）、bottomk（后n条时序）、topk（前n条时序）、quantile（分布统计）

```promql
# 查询系统所有 http 请求的总量
sum(prometheus_http_requests_total)

# 按照 mode 计算主机 CPU 的平均使用时间
avg(node_cpu_seconds_total) by (mode)
```

### 操作符

PromQL支持的所有数学运算符：+（加法）、-（减法）、*（乘法）、/（除法）、%（求余）、^（幂运算）

PromQL支持的所有布尔运算符：=（相等）、!=（不相等）、>（大于）、<（小于）、>=（大于等于）、<=（小于等于）

PromQL支持的集合运算符：and（并且）、or（或者）、unless（排除）

## 五、自定义上传监控指标

### Export/metrice方式

可使用`Micrometer`和`Prometheus Java Simpleclient`上传监控指标，Spring Boot集成了`Micrometer`

- `Micrometer`可对接多种监控系统，如果不再使用Prometheus也可直接切换未别的监控系统，是通用型组件
- `Prometheus Java Simpleclient`是Prometheus的Java客户端，用于上传数据

如下以`Spring Micrometer`为例

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
public class PrometheusMetricApplicationTests {

  @Autowired
  private MeterRegistry registry;

  /**
    * 用于统计次数，例如MQ消费失败次数
    * Counter接口允许以固定的数值递增，该数值必须为正数
    */
  @Test
  public void countersTest() throws Exception {
    // 写入实例名称
    Tag instanceTag = Tag.of("instance", InetAddress.getLocalHost().getHostName());
    // 消费者组名称
    Tag groupNameTag = Tag.of("groupName", "publishGoodsStateChange");
    // 错误信息
    Tag errorMsgTag = Tag.of("errorMsg", "error: error msg is long long text");
    // description 相当于 HELP信息
    Counter.builder("c2c_consume_exception")
      .description("统计失败个数。。。")
      .tags(Lists.list(instanceTag, groupNameTag, errorMsgTag))
      .register(registry)
      .increment();
    Thread.sleep(10 * 1000L);
  }

  /**
    * Gauge是获取当前值的句柄。典型的例子是，获取集合、map、或运行中的线程数等
    */
  @Test
  public void gaugesTest() throws Exception {
    List<Integer> monitorCollect = new ArrayList<>();
    monitorCollect.add(1);
    monitorCollect.add(2);
    monitorCollect.add(3);
    monitorCollect.add(4);
    Gauge.builder("monitor_collect_size", monitorCollect, List::size)
      .description("监控某线程、集合的大小")
      .register(registry);
    Thread.sleep(10 * 1000L);
    monitorCollect.add(5);
    Thread.sleep(10 * 1000L);
  }

  /**
    * Timers（计时器）
    * Timer用于测量短时间延迟和此类事件的频率。所有Timer实现至少将总时间和事件次数报告为单独的时间序列。
    */
  @Test
  public void timersTest() throws InterruptedException {
    // 记录执行开始时间
    Instant startInstant = Instant.now();
    // 切面，执行程序
    Thread.sleep(5 * 1000L);
    // 执行结束时间
    Instant endInstant = Instant.now();
    Timer.builder("c2c_publish_goods_task")
      .description("记录执行时间")
      .register(registry)
      .record(Duration.between(startInstant, endInstant));
    Thread.sleep(100 * 1000L);
  }

  /**
    * 任务计时器用于跟踪所有正在运行的长时间运行任务的总持续时间和此类任务的数量。
    */
  @Test
  public void longTaskTimers() throws InterruptedException {
    // 记录执行开始时间
    LongTaskTimer.builder("b2c_publish_goods_task")
      .description("记录执行时间")
      .register(registry)
      .record(() -> {
        try {
          Thread.sleep(5000L);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      });
    Thread.sleep(100 * 1000L);
  }



  /**
    *  Distribution summaries（分布汇总）
    *  用于跟踪分布式的事件。它在结构上类似于计时器，但是记录的值不代表时间单位。例如，记录http服务器上的请求的响应大小。
    *  可以用百分比直方图、客户端百分比
    */
  @Test
  public void distributionSummariesTest() throws InterruptedException {
    DistributionSummary uploadFileSize = DistributionSummary.builder("upload_file_size")
      .distributionStatisticExpiry(Duration.of(10, ChronoUnit.SECONDS))
      .register(registry);
    
    uploadFileSize.record(30);
    uploadFileSize.record(30);
    uploadFileSize.record(40);
    Thread.sleep(100 * 1000L);
  }
}
```

application.properties配置

```properties
spring.application.name=kun_application
management.endpoints.web.exposure.include=*
management.metrics.tags.application=${spring.application.name}
management.metrics.export.prometheus.enabled=true
```

额外补充：Histograms and percentiles（直方图和百分比） 

Timers 和 distribution summaries 支持收集数据来观察它们的百分比。查看百分比有两种主要方法：

**Percentile histograms（百分比直方图）**： Micrometer将值累积到底层直方图，并将一组预先确定的buckets发送到监控系统。监控系统的查询语言负责从这个直方图中计算百分比。目前，只有Prometheus , Atlas , Wavefront支持基于直方图的百分位数近似值，并且通过histogram_quantile , :percentile , hs()依次表示。

**Client-side percentiles（客户端百分比）**：Micrometer为每个meter ID（一组name和tag）计算百分位数近似值，并将百分位数值发送到监控系统。

配置Prometheus与Spring Boot程序对接，需要配置Prometheus Server，修改prometheus.yml

```yaml
# 添加如下信息
scrape_configs:
  - job_name: 'jobName-application'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: 
        - "targetHostaName:prot"
```

也可以使用服务发现能力集成

### Pushgateway方式

只需要改变application.properties配置和引入依赖

```properties
spring.application.name=kun_application
management.endpoints.web.exposure.include=*
management.metrics.tags.application=${spring.application.name}
management.metrics.export.prometheus.enabled=true
management.metrics.export.prometheus.pushgateway.enabled=true
management.metrics.export.prometheus.pushgateway.base-url=192.168.22.161:9091
management.metrics.export.prometheus.pushgateway.job=${spring.application.name}-job
management.metrics.export.prometheus.pushgateway.push-rate=15s
management.metrics.export.prometheus.pushgateway.base-url=http://PushgatewayHost:Prot
```

```xml
<dependency>
  <groupId>io.prometheus</groupId>
  <artifactId>simpleclient_pushgateway</artifactId>
  <version>[version]</version>
</dependency>
```

## 六、Grafana安装与集成Prometheus

Grafana是一款采用Go语言编写的开源应用，主要用于大规模指标数据的可视化展现，是网络架构和应用分析中最流行的**时序数据展示**工具，目前已经支持绝大部分常用的时序数据库。

### 下载、安装、启动

- 下载、安装

```shell
wget https://dl.grafana.com/oss/release/grafana-[version]-1.x86_64.rpm
sudo yum install grafana-[version]-1.x86_64.rpm
```

- 查看安装目录

```shell
rpm -qa | grep -i grafana
rpm -ql grafana-[version]-1.x86_64
```

- 启动

```shell
systemctl start grafana-server 
systemctl status grafana-server
```

- 查看，访问http://[hostname]:3000默认用户名admin、密码admin

### 集成

点击Setting >> Configuration >> Data Sources >> Add data Source【选择Prometheus，并填写相关信息】

### Grafana的使用

**自定义Panel**

添加PromQL即可

> Row是Panel的集合，即由多个Pannel组成一个界面

---

**拉取公共Dashboard**

访问[grafana-dashboards](https://grafana.com/grafana/dashboards/)，下载json或Copy ID文件，使用Create >> Import【上传Json文件或粘贴ID】

## 七、Grafana自定义报警

使用webhook模式可以自定义任何报警，当然也可以使用最基础的邮件报警机制

点击Alert >> contact points >> New contact point >> 添加一个webhook

编写webhook代码，也可以使用更简便的第三方云平台

```java
@RestController
public class PrometheusWebHook {

    @PostMapping("/prometheus/webhook")
    public void doAlert(@RequestBody String body) {
        System.out.println(body);
    }
}
```

到此webhook设置完成，然后对需要设置的监控指标，设置报警阈值即可





