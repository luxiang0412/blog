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

## 安装和配置

### Prometheus配置

配置说明

prometheus.yml

`promtool check config prometheus.yml`检查配置文件格式是否正确

```yml
global:
  # 抓取指标的频率 默认是1m
  scrape_interval: 15s

  # 抓取指标超时设置 默认10s
  scrape_timeout: 15s

  # 规则配置计算间隔 默认1m
  evaluation_interval: 10s

  # 额外的标签
  external_labels:
    # 标签名称和标签对应的值
    company: shudao

# 规则文件
rule_files:
    # 文件路径
  - "/etc/prometheus_conf/instance.alerts.yml"

# 抓取配置
scrape_configs:
  # Job名称
  - job_name: "prometheus"
    # 访问路径
    metrics_path: "/metrics"
    # schema
    scheme: "http"
    # Job抓取配置
    static_configs:
        # IP:端口
      - targets: ["192.168.33.4:9090"]
        # 标签列表
        labels:
          ip_address: "192.168.33.4"

# Alertmanager配置
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 192.168.33.4:9093
```

instance.alerts.yml
`promtool check rules instance.alerts.yml`检查规则格式是否正确

```yml
groups:
- name: Instance
  rules:

    # 告警名称
  - alert: InstanceDown
    # 触发告警的表达式
    expr: up == 0
    # Down持续5s则出发告警
    for: 5s
    # 标签
    labels:
        severity: page
    # 注解
    annotations:
        # 告警概要 golang template
        summary: "Instance {{ $labels.instance }} down"
        # 告警详情 golang template
        description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."

    # 同上
  - alert: AnotherInstanceDown
    expr: up == 0
    for: 10s
    labels:
        severity: page
    annotations:
        summary: "Instance {{ $labels.instance }} down"
        description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."
```

test.yml rules测试
`promtool test rules test.yml`测试告警规则

```yml
# 测试的规则文件
rule_files:
    - /etc/prometheus_conf/instance.alerts.yml

# 计算间隔
evaluation_interval: 1m

# 测试列表
tests:
    # Test 1.
    - interval: 1m
      # Series data.
      # 输入的数据
      input_series:
          - series: 'up{job="prometheus", instance="192.168.33.4:9090"}'
            values: '0 0 0 0 0 0 0 0 0 0 0 0 0 0 0'
          - series: 'up{job="node_exporter", instance="localhost:9100"}'
            values: '1+0x6 0 0 0 0 0 0 0 0' # 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0
          - series: 'go_goroutines{job="prometheus", instance="192.168.33.4:9090"}'
            values: '10+10x2 30+20x5' # 10 20 30 30 50 70 90 110 130
          - series: 'go_goroutines{job="node_exporter", instance="localhost:9100"}'
            values: '10+10x7 10+30x4' # 10 20 30 40 50 60 70 80 10 40 70 100 130

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
                  exp_annotations:
                      summary: "Instance 192.168.33.4:9090 down"
                      description: "192.168.33.4:9090 of job prometheus has been down for more than 5 minutes."
      # Unit tests for promql expressions.
      promql_expr_test:
          # Unit test 1.
          - expr: go_goroutines > 5
            eval_time: 4m
            exp_samples:
                # Sample 1.
                - labels: 'go_goroutines{job="prometheus",instance="192.168.33.4:9090"}'
                  value: 50
                # Sample 2.
                - labels: 'go_goroutines{job="node_exporter",instance="192.168.33.4:9100"}'
                  value: 50
```

### Grafana配置



### Alertmanager配置

alertmanager.yml

`amtool check-config alertmanager.yml`检查alertmanager配置文件

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

## 使用



## 参考

- [Prometheus](https://prometheus.io/docs/prometheus/latest/getting_started/)
- [Alertmanager](https://prometheus.io/docs/alerting/latest/overview/)
- [Grafana Prometheus](https://grafana.com/docs/grafana/latest/datasources/prometheus/)
- [Grafana dashboard](https://grafana.com/grafana/dashboards)
- [Prometheus webhook dingtalk](https://github.com/timonwong/prometheus-webhook-dingtalk)
- [Alertsnitch](https://github.com/yakshaving-art/alertsnitch)
- [Alertmanager webhook receiver](https://prometheus.io/docs/operating/integrations/#alertmanager-webhook-receiver)
- [Alertmanager 配置](https://www.yxingxing.net/archives/alertmanager-20191218-config)