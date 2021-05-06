---
title: Prometheus监控告警简单使用
date: 2021-05-05 13:31:49
tags:
---

## 介绍

- Prometheus

    时序数据库，用来存储和查询监控指标。

- Grafana

    数据可视化。可以将Prometheus等数据源的数据进行可视化。

- Alertmanager

    Prometheus的警报分为两个部分。Prometheus服务器中的警报规则将警报发送到Alertmanager。该Alertmanager 然后管理这些警报，包括沉默，抑制，聚集和通过的方法，如电子邮件发出通知，Webhook，以及即时通讯平台。

- Prometheus-webhook-dingtalk

    集成钉钉，将告警信息推送到钉钉。

- Alertsnitch

    集成Mysql或者Postgres，将告警信息存储到数据库。

## 环境说明

**注意：**prom-alert-webhook，sub-system-a，sub-system-b这三个应用是另外一个项目[metrics-demo](http://10.7.200.214:8929/luxiang/metrics-demo)的子模块，作为测试用。每个子模块通过路径`/actuator/prometheus`获取metrics。

|IP|说明|
|:---|:---|
|192.168.33.1|-|
|192.168.33.4|-|

|服务/应用|IP|端口|类型|Metrics Path|说明|
|:---|:---|:---|:---|:---|:---|
|Prometheus|192.168.33.4|9090|服务|`/metrics`|-|
|Grafana|192.168.33.4|3000|服务|`/metrics`|-|
|Alertmanager|192.168.33.4|9093|服务|`/metrics`|-|
|prometheus-webhook-dingtalk|192.168.33.4|8060|服务|无|-|
|alertsnitch|192.168.33.4|9567|服务|`/metrics`|-|
|prom-alert-webhook|192.168.33.1|8084|应用|`/actuator/prometheus`|-|
|sub-system-a|192.168.33.1|8081|应用|`/actuator/prometheus`|-|
|sub-system-b|192.168.33.1|8082|应用|`/actuator/prometheus`|-|


## 安装和配置

文件目录(prometheus_alert)如下：

```
├── alertsnitch
│   └── mysql
│       ├── 0.0.1-bootstrap.sql
│       └── 0.1.0-fingerprint.sql
├── altermanager_conf
│   └── altermanager.yml
├── docker-compose.yml
├── prometheus_conf
│   ├── instance.alerts.yml
│   ├── prometheus.yml
│   └── test.yml
└── prometheus-webhook-dingtalk
    ├── config.yml
    └── templates
        ├── default.tmpl
        ├── issue43
        │   └── template.tmpl
        └── legacy
            └── template.tmpl

```

首先拉取镜像：

```bash
docker pull prom/prometheus:v2.26.0 && \
  docker pull prom/alertmanager:v0.21.0 && \
  docker pull grafana/grafana:7.5.5 && \
  docker pull timonwong/prometheus-webhook-dingtalk:latest && \
  docker pull luxiang0412/alertsnitch:0.2.1
```

进入文件目录(prometheus_alert)的根下：

`cd prometheus_alert`

**下面操作都是在prometheus_alert目录下进行**

### Prometheus配置

配置说明

检查prometheus.yml格式是否正确

```bash
docker run --rm -it --entrypoint='' -v "$PWD/prometheus_conf:/etc/prometheus_conf" prom/prometheus:v2.26.0 promtool check config /etc/prometheus_conf/prometheus.yml
```

prometheus.yml

```yml
# 全局配置
global:
  scrape_interval: 15s # 抓取指标的间隔
  evaluation_interval: 15s # 计算规则的间隔
  scrape_timeout: 10s #抓取指标超时时间

# Alertmanager配置
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 192.168.33.4:9093

# 规则文件
rule_files:
  - "/etc/prometheus_conf/instance.alerts.yml"

# 抓取配置
scrape_configs:
  # 名称
  - job_name: "prometheus"
    metrics_path: "/metrics"
    scheme: "http"
    static_configs:
      - targets: ["192.168.33.4:9090"]
        labels:
          ip_address: "192.168.33.4"
  # 名称
  - job_name: "application"
    metrics_path: "/actuator/prometheus"
    scheme: "http"
    static_configs:
      - targets: ["192.168.33.1:8081"]
        labels:
          ip_address: "192.168.33.1"
          type: "application"
      - targets: ["192.168.33.1:8082"]
        labels:
          ip_address: "192.168.33.1"
          type: "application"
      - targets: ["192.168.33.1:8084"]
        labels:
          ip_address: "192.168.33.1"
          type: "application"

```

检查rules.yml格式是否正确

```bash
docker run --rm -it --entrypoint='' -v "$PWD/prometheus_conf:/etc/prometheus_conf" prom/prometheus:v2.26.0 promtool check rules /etc/prometheus_conf/instance.alerts.yml
```

instance.alerts.yml

```yml
# This is the rules file.

groups:
  - name: Instance
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5s
        labels:
          severity: page
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."
    # 组名
  - name: SdService
    # 规则
    rules:
      # 警报名称
      - alert: SdServiceDown
        # 表达式
        expr: sd_service_status == 0
        # 持续时间(如果expr为真，且时间大于for的值则触发警报)
        for: 5s
        labels:
          severity: page
        annotations:
          summary: "sd service {{ $labels.application}} down "
          description: "sd service {{ $labels.application}} down"

```

测试告警规则

```bash
docker run --rm -it --entrypoint='' -v "$PWD/prometheus_conf:/etc/prometheus_conf" prom/prometheus:v2.26.0 promtool test rules /etc/prometheus_conf/test.yml
```

test.yml rules测试

```yml
# This is the main input for unit testing.
# Only this file is passed as command line argument.

rule_files:
  - /etc/prometheus_conf/instance.alerts.yml

evaluation_interval: 1m

tests:
  # Test 1.
  - name: Test-1
    interval: 1m
    # Series data.
    input_series:
      - series: 'up{instance="192.168.33.4:9090", ip_address="192.168.33.4", job="prometheus"}'
        values: "1+0x6 0 0 0 0 0 0 0 0" # 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0
      - series: 'go_goroutines{instance="192.168.33.4:9090", ip_address="192.168.33.4", job="prometheus"}'
        values: "10+10x2 30+20x5" # 10 20 30 30 50 70 90 110 130
      - series: 'sd_service_status{application="prom-alert-webhook", instance="192.168.33.1:8084", ip_address="192.168.33.1", job="application", type="application"}'
        values: "1+0x5 0+0x5" # 1 1 1 1 1 1 0 0 0 0 0 0
      - series: 'sd_service_status{application="sub-system-a", instance="192.168.33.1:8081", ip_address="192.168.33.1", job="application", type="application"}'
        values: "1+0x5 0+0x5" # 1 1 1 1 1 1 0 0 0 0 0 0
      - series: 'sd_service_status{application="sub-system-b", instance="192.168.33.1:8082", ip_address="192.168.33.1", job="application", type="application"}'
        values: "1+0x5 0+0x5" # 1 1 1 1 1 1 0 0 0 0 0 0
    # Unit test for alerting rules.
    alert_rule_test:
      # Unit test 1.
      - eval_time: 10m
        alertname: InstanceDown
        exp_alerts:
          # Alert 1.
          - exp_labels:
              severity: page
              instance: 192.168.33.4:9090
              job: prometheus
              ip_address: 192.168.33.4
            exp_annotations:
              summary: "Instance 192.168.33.4:9090 down"
              description: "192.168.33.4:9090 of job prometheus has been down for more than 5 minutes."
      # sd service alert 单元测试
      - eval_time: 10m
        alertname: SdServiceDown
        exp_alerts:
          # service prom-alert-webhook 当alert的label和annotations 和 下面预期的一样则通过test
          - exp_labels:
              severity: page
              instance: 192.168.33.1:8084
              job: application
              ip_address: 192.168.33.1
              application: prom-alert-webhook
              type: application
            exp_annotations:
              summary: "sd service prom-alert-webhook down "
              description: "sd service prom-alert-webhook down"
          # service sub-system-a
          - exp_labels:
              severity: page
              instance: 192.168.33.1:8081
              job: application
              ip_address: 192.168.33.1
              application: sub-system-a
              type: application
            exp_annotations:
              summary: "sd service sub-system-a down "
              description: "sd service sub-system-a down"
          # service sub-system-b
          - exp_labels:
              severity: page
              instance: 192.168.33.1:8082
              job: application
              ip_address: 192.168.33.1
              application: sub-system-b
              type: application
            exp_annotations:
              summary: "sd service sub-system-b down "
              description: "sd service sub-system-b down"
    # Unit tests for promql expressions.
    promql_expr_test:
      # Unit test 1.
      - expr: go_goroutines > 5
        eval_time: 5m
        exp_samples:
          # Sample 1.
          - labels: 'go_goroutines{instance="192.168.33.4:9090", ip_address="192.168.33.4", job="prometheus"}'
            value:
              70
              # Unit test 1.
      - expr: up > -1
        eval_time: 7m
        exp_samples:
          # Sample 2.
          - labels: 'up{instance="192.168.33.4:9090", ip_address="192.168.33.4", job="prometheus"}'
            value: 0
```

### Grafana配置



### Alertmanager配置

检查alertmanager配置文件

```bash
docker run --rm -it --entrypoint='' -v "$PWD/alertmanager_conf:/etc/alertmanager_conf" prom/alertmanager:v0.21.0 amtool check-config /etc/alertmanager_conf/alertmanager.yml
```

alertmanager.yml

```yml
global:
  smtp_from: lux@haoshudao.com
  smtp_smarthost: smtp.qiye.aliyun.com:465
  smtp_auth_username: ****
  smtp_auth_password: ****
  smtp_require_tls: false
route:
  receiver: "web.hook.db"
  group_by: ["alertname"]
  continue: false
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 1h
  routes:
    - receiver: "web.hook.db"
      continue: true
      match:
        type: "application"

    - receiver: "web.hook.dingding"
      continue: true
      match:
        type: "application"

    - receiver: "email.luxiang"
      continue: true
      match:
        type: "application"
receivers:
  - name: "web.hook.dingding"
    webhook_configs:
      - url: "http://192.168.33.4:8060/dingtalk/webhook_legacy/send"
  - name: "web.hook.db"
    webhook_configs:
      - url: "http://192.168.33.4:9567/webhook"
  - name: "email.luxiang"
    email_configs:
      - to: 1374920889@qq.com
        send_resolved: true
inhibit_rules:
  - source_match:
      severity: "critical"
    target_match:
      severity: "warning"
    equal: ["alertname", "dev", "instance"]

```

### Prometheus-webhook-dingtalk配置

config.yml
```yml
## Request timeout
# timeout: 5s

## Customizable templates path
# templates:
#   - contrib/templates/legacy/template.tmpl

## You can also override default template using `default_message`
## The following example to use the 'legacy' template from v0.3.0
# default_message:
#   title: '{{ template "legacy.title" . }}'
#   text: '{{ template "legacy.content" . }}'

## Targets, previously was known as "profiles"
targets:
  webhook_legacy:
    url: https://oapi.dingtalk.com/robot/send?access_token=****
    secret: ****
    # Customize template content
    #message:
      # Use legacy template
      #title: '{{ template "legacy.title" . }}'
      #text: '{{ template "legacy.content" . }}'
    mention:
      all: true

```

### Alertsnitch配置

mysql-init.sql
```sql
DROP PROCEDURE IF EXISTS bootstrap;

DELIMITER //
CREATE PROCEDURE bootstrap()
BEGIN
  SET @exists := (SELECT 1 FROM information_schema.tables I WHERE I.table_name = "Model" AND I.table_schema = database());
  IF @exists IS NULL THEN

    CREATE TABLE `Model` (
      `ID` enum('1') NOT NULL,
      `version` VARCHAR(20) NOT NULL,
      PRIMARY KEY (`ID`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

    INSERT INTO `Model` (`version`) VALUES ("0.0.1");

  ELSE
    SIGNAL SQLSTATE '42000' SET MESSAGE_TEXT='Model Table Exists, quitting...';
  END IF;
END;
//
DELIMITER ;

-- Execute the procedure
CALL bootstrap();

-- Drop the procedure
DROP PROCEDURE bootstrap;

-- Create the rest of the tables
CREATE TABLE `AlertGroup` (
	`ID` INT NOT NULL AUTO_INCREMENT,
	`time` TIMESTAMP NOT NULL,
	`receiver` VARCHAR(100) NOT NULL,
	`status` VARCHAR(50) NOT NULL,
	`externalURL` TEXT NOT NULL,
	`groupKey` VARCHAR(255) NOT NULL,
	KEY `idx_time` (`time`) USING BTREE,
    KEY `idx_status_ts` (`status`, `time`) USING BTREE,
	PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `GroupLabel` (
	`ID` INT NOT NULL AUTO_INCREMENT,
    `AlertGroupID` INT NOT NULL,
    `GroupLabel` VARCHAR(100) NOT NULL,
    `Value` VARCHAR(1000) NOT NULL,
    FOREIGN KEY (AlertGroupID) REFERENCES AlertGroup (ID) ON DELETE CASCADE,
	PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `CommonLabel` (
	`ID` INT NOT NULL AUTO_INCREMENT,
    `AlertGroupID` INT NOT NULL,
    `Label` VARCHAR(100) NOT NULL,
    `Value` VARCHAR(1000) NOT NULL,
    FOREIGN KEY (AlertGroupID) REFERENCES AlertGroup (ID) ON DELETE CASCADE,
	PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `CommonAnnotation` (
	`ID` INT NOT NULL AUTO_INCREMENT,
    `AlertGroupID` INT NOT NULL,
    `Annotation` VARCHAR(100) NOT NULL,
    `Value` VARCHAR(1000) NOT NULL,
    FOREIGN KEY (AlertGroupID) REFERENCES AlertGroup (ID) ON DELETE CASCADE,
	PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `Alert` (
	`ID` INT NOT NULL AUTO_INCREMENT,
    `alertGroupID` INT NOT NULL,
	`status` VARCHAR(50) NOT NULL,
    `startsAt` DATETIME NOT NULL,
    `endsAt` DATETIME DEFAULT NULL,
	`generatorURL` TEXT NOT NULL,
    FOREIGN KEY (alertGroupID) REFERENCES AlertGroup (ID) ON DELETE CASCADE,
	PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `AlertLabel` (
	`ID` INT NOT NULL AUTO_INCREMENT,
    `AlertID` INT NOT NULL,
    `Label` VARCHAR(100) NOT NULL,
    `Value` VARCHAR(1000) NOT NULL,
    FOREIGN KEY (AlertID) REFERENCES Alert (ID) ON DELETE CASCADE,
	PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `AlertAnnotation` (
	`ID` INT NOT NULL AUTO_INCREMENT,
    `AlertID` INT NOT NULL,
    `Annotation` VARCHAR(100) NOT NULL,
    `Value` VARCHAR(1000) NOT NULL,
    FOREIGN KEY (AlertID) REFERENCES Alert (ID) ON DELETE CASCADE,
	PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
ALTER TABLE Alert 
    ADD `fingerprint` TEXT NOT NULL
;

UPDATE `Model`  SET `version`="0.1.0";

```

### docker-compose启动

docker-compose.yml
```yml
version: "3"
services:
    prometheus:
        image: prom/prometheus:v2.26.0
        container_name: prometheus
        restart: always
        ports:
            - 9090:9090
        environment: 
            - TZ=Asia/Shanghai
        command:
            - --config.file=/etc/prometheus_conf/prometheus.yml
        volumes:
            - ./prometheus_conf:/etc/prometheus_conf:ro
    alertmanager:
        image: prom/alertmanager:v0.21.0
        container_name: alertmanager
        restart: always
        ports:
            - 9093:9093
        environment: 
            - TZ=Asia/Shanghai
        command:
            - --config.file=/etc/altermanager_conf/altermanager.yml
            - --web.external-url=http://192.168.33.4:9093/
            # test
            - --log.level=debug
        volumes:
            - ./altermanager_conf:/etc/altermanager_conf:ro
    # alertmanager webhook集成钉钉（可选）
    prometheus-webhook-dingtalk:
        image: timonwong/prometheus-webhook-dingtalk:latest
        container_name: prometheus-webhook-dingtalk
        restart: always
        ports:
            - 8060:8060
        environment: 
            - TZ=Asia/Shanghai
        command:
            - --config.file=/etc/prometheus-webhook-dingtalk/config.yml
            - --web.enable-ui
            - --web.enable-lifecycle
        volumes:
            - ./prometheus-webhook-dingtalk:/etc/prometheus-webhook-dingtalk:ro
    # alertmanager database集成（可选）
    alertsnitch:
        image: luxiang0412/alertsnitch:0.2.1
        container_name: alertsnitch
        restart: always
        ports:
            - 9567:9567
        environment: 
            - TZ=Asia/Shanghai
        environment: 
            ALERTSNITCH_DSN: 'root:password@tcp(192.168.33.4:3306)/dataplatform_test'
            ALERSTNITCH_BACKEND: 'mysql'
    grafana:
        image: grafana/grafana:7.5.5
        container_name: grafana
        restart: always
        ports:
            - "3000:3000"
        environment: 
            - TZ=Asia/Shanghai

```

## Spring-boot 自定义指标

可以在自己项目中暴露业务指标，然后Prometheus抓取指标存储到TSDB（Time series database）；  
然后通过grafana定制自己的监控面板，还可以定制自己的业务告警规则，也可以在项目中通过HTTP API查询Prometheus数据作展示；

依赖

pom.xml （版本自己选择）

```xml
<!-- Spring boot actuator to expose metrics endpoint -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <version>${spring-boot.version}</version>
</dependency>
<!-- Micormeter core dependecy  -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-core</artifactId>
    <version>${io.micrometer.version}</version>
</dependency>
<!-- Micrometer Prometheus registry  -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>${io.micrometer.version}</version>
</dependency>
```

---

> [common-application-properties-actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-application-properties.html#common-application-properties-actuator)

application.yml

```yml
management:
  endpoint:
    # 启用metrics端点
    metrics.enabled: true
    # 启用prometheus端点
    prometheus.enabled: true
  # 指标暴露到prometheus端点
  metrics.export.prometheus.enabled: true
  endpoints:
    web:
      exposure:
        include:
          #- 'health'
          #- 'info'
          #- 'metrics'
          - 'prometheus'
          #- 'heapdump'
```
---

自定义业务指标

[有四种类型的指标](https://prometheus.io/docs/concepts/metric_types/)

|类型|说明|
|:---|:---|
|Counter|-|
|Gauge|-|
|Histogram|-|
|Summary|-|

test metrics

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

/**
 * @author luxiang
 * description  //自定义指标注册
 * create       2021-04-29 11:06
 */
@Component
public class CustomizeMetrics {

    /**
     * 应用名称
     */
    private String applicationName;

    private MeterRegistry meterRegistry;

    /**
     * 计数器
     */
    private final Counter counter;

    /**
     * 状态
     */
    private double[] status = {1D};

    /**
     * 构造方法
     * @param meterRegistry meter registry
     * @param applicationName 应用名称
     */
    public CustomizeMetrics(MeterRegistry meterRegistry,
                            @Value("${spring.application.name}") String applicationName) {
        this.meterRegistry = meterRegistry;
        this.applicationName = applicationName;
        counter = Counter.builder("sd.count")
                .tag("application", applicationName)
                .description("计数器")
                .register(meterRegistry);

        Gauge.builder("sd.service.status", status, e -> e[0])
                .tag("application", applicationName)
                .description("服务状态 0停止 1启动")
                .register(meterRegistry);
    }

    /**
     * 处理计数器 +1
     */
    public void processCounter() {
        counter.increment();
    }

    /**
     * 改变状态 0->1 1->0
     */
    public void processStatus() {
        if (status[0] == 0) {
            status[0] = 1;
        } else {
            status[0] = 0;
        }
    }
}
```

## 参考

- [Prometheus](https://prometheus.io/docs/prometheus/latest/getting_started/)
- [Alertmanager](https://prometheus.io/docs/alerting/latest/overview/)
- [Grafana Prometheus](https://grafana.com/docs/grafana/latest/datasources/prometheus/)
- [Grafana dashboard](https://grafana.com/grafana/dashboards)
- [Prometheus webhook dingtalk](https://github.com/timonwong/prometheus-webhook-dingtalk)
- [Alertsnitch](https://github.com/yakshaving-art/alertsnitch)
- [Alertmanager webhook receiver](https://prometheus.io/docs/operating/integrations/#alertmanager-webhook-receiver)
- [Alertmanager 配置](https://www.yxingxing.net/archives/alertmanager-20191218-config)