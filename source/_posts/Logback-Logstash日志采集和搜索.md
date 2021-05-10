---
title: Logback + Logstash日志采集和搜索
date: 2021-05-05 13:32:28
tags:
---

## 环境说明

|名称|版本|
|:---|:---|
|Elasticsearch|5.5.0|
|Kibana|5.5.0|
|Logstash|5.5.0|
|logstash-logback-encoder|6.6|

## 环境搭建

相关下载：

- [elasticsearch-5.5.0.tar.gz](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.0.tar.gz)
- [elasticsearch-analysis-ik-5.5.3.zip](https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.3/elasticsearch-analysis-ik-5.5.3.zip)
- [kibana](https://artifacts.elastic.co/downloads/kibana/kibana-5.5.0-linux-x86_64.tar.gz)
- [logstash](https://artifacts.elastic.co/downloads/logstash/logstash-5.5.0.tar.gz)
- [elasticsearch-analysis-pinyin-5.5.3.zip](https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v5.5.3/elasticsearch-analysis-pinyin-5.5.3.zip)
- [lu-xiang-3bedf7c7-4c72-40d4-855b-0caec7fb4c6d-v5.json](https://gitee.com/luxiang0412/x-pack/attach_files/691767/download/lu-xiang-3bedf7c7-4c72-40d4-855b-0caec7fb4c6d-v5.json)
- [x-pack-5.5.0.tar.gz](https://gitee.com/luxiang0412/x-pack/attach_files/691768/download/x-pack-5.5.0.tar.gz)
- [Kibana_Hanization.tar.gz](https://gitee.com/luxiang0412/x-pack/attach_files/691786/download/Kibana_Hanization.tar.gz)


elasticsearch安装

```bash
# 解压elasticsearch-5.5.0.tar.gz
tar -zxvf elasticsearch-5.5.0.tar.gz -C /usr/local
# 创建软连接
ln -s /usr/local/elasticsearch-5.5.0 elasticsearch
# 创建用户elastic
useradd elastic
# 下载json
curl -o /usr/local/elasticsearch/lu-xiang-3bedf7c7-4c72-40d4-855b-0caec7fb4c6d-v5.json -L https://gitee.com/luxiang0412/x-pack/attach_files/691767/download/lu-xiang-3bedf7c7-4c72-40d4-855b-0caec7fb4c6d-v5.json
# 下载x-pack破解
curl -o /usr/local/elasticsearch/plugins/x-pack-5.5.0.tar.gz -L https://gitee.com/luxiang0412/x-pack/attach_files/691768/download/x-pack-5.5.0.tar.gz
# 解压x-pack
tar -zxvf /usr/local/elasticsearch/plugins/x-pack-5.5.0.tar.gz -C /usr/local/elasticsearch/plugins/
# 删除x-pack压缩包
rm -rf /usr/local/elasticsearch/plugins/x-pack-5.5.0.tar.gz
# 解压ik
unzip -d /usr/local/elasticsearch/plugins/ik elasticsearch-analysis-ik-5.5.3.zip && \
    mv /usr/local/elasticsearch/plugins/ik/elasticsearch/* /usr/local/elasticsearch/plugins/ik/ && \
    rm -rf /usr/local/elasticsearch/plugins/ik/elasticsearch

# 修改版本号
sed -i 's#elasticsearch.version=5.5.3#elasticsearch.version=5.5.0#g' /usr/local/elasticsearch/plugins/ik/plugin-descriptor.properties

# 解压pinyin
unzip -d /usr/local/elasticsearch/plugins/pinyin elasticsearch-analysis-pinyin-5.5.3.zip && \
    mv /usr/local/elasticsearch/plugins/pinyin/elasticsearch/* /usr/local/elasticsearch/plugins/pinyin/ && \
    rm -rf /usr/local/elasticsearch/plugins/pinyin/elasticsearch

# 修改版本号
sed -i 's#elasticsearch.version=5.5.3#elasticsearch.version=5.5.0#g' /usr/local/elasticsearch/plugins/pinyin/plugin-descriptor.properties

# 修改所有者和组
chown -R elastic:elastic /usr/local/elasticsearch /usr/local/elasticsearch-5.5.0/

# 启动elasticsearch
runuser -u elastic -- /usr/local/elasticsearch/bin/elasticsearch -d -p /usr/local/elasticsearch/pid

# 停止elasticsearch
runuser -u elastic -- pkill -F /usr/local/elasticsearch/pid

# 测试elasticsearch
curl -u elastic:changeme http://localhost:9200

# 修改license
curl -u elastic:changeme http://localhost:9200/_xpack/license -d @/usr/local/elasticsearch/lu-xiang-3bedf7c7-4c72-40d4-855b-0caec7fb4c6d-v5.json
# 修改elasticsearch密码
curl -XPUT -u elastic:changeme 'http://localhost:9200/_xpack/security/user/elastic/_password' -d '{ "password" : "111111" }'

# ik test https://github.com/medcl/elasticsearch-analysis-ik/tree/5.x
curl -u 'elastic:111111' -XPUT http://localhost:9200/index

curl -u 'elastic:111111' -XPOST http://localhost:9200/index/fulltext/_mapping -d'
{
        "properties": {
            "content": {
                "type": "text",
                "analyzer": "ik_max_word",
                "search_analyzer": "ik_max_word"
            }
        }
    
}'

curl -u 'elastic:111111' -XPOST http://localhost:9200/index/fulltext/1 -d'
{"content":"美国留给伊拉克的是个烂摊子吗"}
'

curl -u 'elastic:111111' -XPOST http://localhost:9200/index/fulltext/2 -d'
{"content":"公安部：各地校车将享最高路权"}
'

curl -u 'elastic:111111' -XPOST http://localhost:9200/index/fulltext/3 -d'
{"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}
'

curl -u 'elastic:111111' -XPOST http://localhost:9200/index/fulltext/4 -d'
{"content":"中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"}
'

curl -u 'elastic:111111' -XPOST http://localhost:9200/index/fulltext/_search  -d'
{
    "query" : { "match" : { "content" : "中国" }},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "content" : {}
        }
    }
}
'

# pinyin test https://github.com/medcl/elasticsearch-analysis-pinyin/tree/5.x
curl -u 'elastic:111111' -XPUT http://localhost:9200/medcl/ -d'
{
    "index" : {
        "analysis" : {
            "analyzer" : {
                "pinyin_analyzer" : {
                    "tokenizer" : "my_pinyin"
                    }
            },
            "tokenizer" : {
                "my_pinyin" : {
                    "type" : "pinyin",
                    "keep_separate_first_letter" : false,
                    "keep_full_pinyin" : true,
                    "keep_original" : true,
                    "limit_first_letter_length" : 16,
                    "lowercase" : true,
                    "remove_duplicated_term" : true
                }
            }
        }
    }
}'

curl -u 'elastic:111111' -XGET 'http://localhost:9200/medcl/_analyze?text=%e5%88%98%e5%be%b7%e5%8d%8e&analyzer=pinyin_analyzer'

curl -u 'elastic:111111' -XPOST http://localhost:9200/medcl/folks/_mapping -d'
{
    "folks": {
        "properties": {
            "name": {
                "type": "keyword",
                "fields": {
                    "pinyin": {
                        "type": "text",
                        "store": "no",
                        "term_vector": "with_offsets",
                        "analyzer": "pinyin_analyzer",
                        "boost": 10
                    }
                }
            }
        }
    }
}'

curl -u 'elastic:111111' -XPOST http://localhost:9200/medcl/folks/andy -d'{"name":"刘德华"}'

curl -u 'elastic:111111' -XGET 'http://localhost:9200/medcl/folks/_search?q=name:%E5%88%98%E5%BE%B7%E5%8D%8E'

curl -u 'elastic:111111' 'http://localhost:9200/medcl/folks/_search?q=name.pinyin:%e5%88%98%e5%be%b7'

curl -u 'elastic:111111' http://localhost:9200/medcl/folks/_search?q=name.pinyin:liu

curl -u 'elastic:111111' http://localhost:9200/medcl/folks/_search?q=name.pinyin:ldh

curl -u 'elastic:111111' http://localhost:9200/medcl/folks/_search?q=name.pinyin:de+hua

curl -u 'elastic:111111' -XPUT http://localhost:9200/medcl1/ -d'
{
    "index" : {
        "analysis" : {
            "analyzer" : {
                "user_name_analyzer" : {
                    "tokenizer" : "whitespace",
                    "filter" : "pinyin_first_letter_and_full_pinyin_filter"
                }
            },
            "filter" : {
                "pinyin_first_letter_and_full_pinyin_filter" : {
                    "type" : "pinyin",
                    "keep_first_letter" : true,
                    "keep_full_pinyin" : false,
                    "keep_none_chinese" : true,
                    "keep_original" : false,
                    "limit_first_letter_length" : 16,
                    "lowercase" : true,
                    "trim_whitespace" : true,
                    "keep_none_chinese_in_first_letter" : true
                }
            }
        }
    }
}'

curl -u 'elastic:111111' -XGET 'http://localhost:9200/medcl1/_analyze?text=%e5%88%98%e5%be%b7%e5%8d%8e+%e5%bc%a0%e5%ad%a6%e5%8f%8b+%e9%83%ad%e5%af%8c%e5%9f%8e+%e9%bb%8e%e6%98%8e+%e5%9b%9b%e5%a4%a7%e5%a4%a9%e7%8e%8b&analyzer=user_name_analyzer'
```

kibana安装

```bash
# 解压kibana
tar -zxvf kibana-5.5.0-linux-x86_64.tar.gz -C /usr/local/

# 创建软连接
ln -s /usr/local/kibana-5.5.0-linux-x86_64 /usr/local/kibana

# 修改配置
sed -i 's/#server.host: "localhost"/server.host: "0.0.0.0"/g' /usr/local/kibana/config/kibana.yml
sed -i 's@#elasticsearch.url: "http://localhost:9200"@elasticsearch.url: "http://localhost:9200"@g' /usr/local/kibana/config/kibana.yml
sed -i 's/#elasticsearch.username: "user"/elasticsearch.username: "elastic"/g' /usr/local/kibana/config/kibana.yml
sed -i 's/#elasticsearch.password: "pass"/elasticsearch.password: "111111"/g' /usr/local/kibana/config/kibana.yml

# 汉化kibana
tar -zxvf Kibana_Hanization.tar.gz -C /usr/local/ && \
    cd /usr/local/Kibana_Hanization/old && \
    python main.py /usr/local/kibana

# 启动kibana
nohup /usr/local/kibana/bin/kibana >> /usr/local/kibana/kibana.log 2>&1 & echo $! > /usr/local/kibana/pid

# 停止kibana
pkill -F /usr/local/kibana/pid
```

logstash安装

```bash
# 解压logstash
tar -zxvf logstash-5.5.0.tar.gz -C /usr/local/ && \
    ln -s /usr/local/logstash-5.5.0 logstash

# 启动logstash  配置文件在下面
nohup /usr/local/logstash/bin/logstash -f pipeline/logstash.conf >> /usr/local/logstash/logstash.log 2>&1 & echo $! > /usr/local/logstash/pid

# 停止logstash
pkill -F /usr/local/logstash/pid
```

elasticsearch-5x.json 索引模板

```json
{
    "template": "logstash-*",
    "version": 50001,
    "settings": {
        "index.refresh_interval": "5s"
    },
    "mappings": {
        "_default_": {
            "_all": {
                "enabled": true,
                "norms": false
            },
            "dynamic_templates": [
                {
                    "message_field": {
                        "path_match": "message",
                        "match_mapping_type": "string",
                        "mapping": {
                            "type": "text",
                            "norms": false,
                            "analyzer": "ik_max_word",
                            "fields": {
                                "suggest": {
                                    "type": "completion",
                                    "analyzer": "ik_max_word"
                                },
                                "keyword": {
                                    "type": "keyword"
                                }
                            }
                        }
                    }
                },
                {
                    "string_fields": {
                        "match": "*",
                        "match_mapping_type": "string",
                        "mapping": {
                            "type": "text",
                            "norms": false,
                            "fields": {
                                "keyword": {
                                    "type": "keyword",
                                    "ignore_above": 256
                                }
                            }
                        }
                    }
                }
            ],
            "properties": {
                "@timestamp": {
                    "type": "date",
                    "include_in_all": false
                },
                "@version": {
                    "type": "keyword",
                    "include_in_all": false
                },
                "geoip": {
                    "dynamic": true,
                    "properties": {
                        "ip": {
                            "type": "ip"
                        },
                        "location": {
                            "type": "geo_point"
                        },
                        "latitude": {
                            "type": "half_float"
                        },
                        "longitude": {
                            "type": "half_float"
                        }
                    }
                }
            }
        }
    }
}
```

pipeline/logstash.conf
```
input {
    tcp {
        port => 4560
        codec => json_lines
    }
    
    # 正式环境只需要一个TCP端口，多个logstash分布不同机器
    tcp {
        port => 4561
        codec => json_lines
    }
}
output {

    elasticsearch {
        # elasticsearch地址，这个是数组
        hosts => ["http://192.168.33.4:9200"]
        # 索引的名称
        index => "logstash-%{+YYYY.MM.dd}"
        # 索引模板名称
        template_name => "logstash"
        # 索引模板json路径
        template => "/usr/local/logstash/config/elasticsearch-5x.json"
        # 覆盖默认的模板
        template_overwrite => true
        manage_template => true
        # elasticsearch 用户名
        user => "elastic"
        # elasticsearch 密码
        password => "111111"
    }
    # 正式环境注释掉，这个是输出到控制台
    stdout {
        codec => rubydebug
    }
}
```

## spring-boot和logstash-logback-encoder配置

可以参考项目[metrics-demo](http://10.7.200.214:8929/luxiang/metrics-demo/-/blob/master/prom-alert-webhook/src/main/resources/logback-spring.xml)的配置

引入依赖
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>6.6</version>
</dependency>
<!-- Your project must also directly depend on either logback-classic or logback-access.  For example: -->
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.3</version>
</dependency>
```

application.yml（多个logstash地址用逗号隔开）

```yml
spring:
  application:
    # 应用名称
    name: sub-system-a
# logstash TCP 端口（logstash是无状态的，可以横向扩展）
logstash.address: |
  192.168.33.4:4560,
  192.168.33.4:4561
```

logback-spring.xml
下面的配置文件需要修改的地方：

- `<listener class="com.sd.promalertwebhook.listen.logback.MyTcpAppenderListener"/>`
- log level根据实际情况调整
- 日志大小，保存天数，最大大小等；需要更具实际情况调整。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <springProperty scope="local" name="spring.application.name" source="spring.application.name"
                    defaultValue=""/>

    <springProperty scope="local" name="logstash.address" source="logstash.address"
                    defaultValue=""/>

    <!--日志存放路径-->
    <property name="log.path" value="${user.home}/logs/${spring.application.name}"/>

    <!--日志文件最大大小（字节）-->
    <property name="file.max-filesize" value="100MB"/>

    <!-- 日志输出格式 -->
    <property name="log.pattern" value="[%d{HH:mm:ss.SSS}] [%thread] %-5level %logger{20} - [%method,%line] - %msg%n"/>

    <appender name="STASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>${logstash.address}</destination>
        <!-- encoder is required -->
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <version>5.5.0</version>
            <!-- 自定义字段 添加项目名称 -->
            <customFields>{"application_name": "${spring.application.name}"}</customFields>
            <includeContext>false</includeContext>
            <includeCallerData>true</includeCallerData>
        </encoder>

        <!-- 空闲超时 -->
        <keepAliveDuration>5 minutes</keepAliveDuration>

        <!-- 连接策略 1.preferPrimary（先选第一个） 2.roundRobin（循环） 3.random（随机） -->
        <connectionStrategy>
            <random>
                <connectionTTL>5 minutes</connectionTTL>
            </random>
        </connectionStrategy>

        <!-- 重新连接延迟 -->
        <reconnectionDelay>7 second</reconnectionDelay>

        <!-- 写入缓冲大小 -->
        <writeBufferSize>8192</writeBufferSize>

        <!-- 写入超时 当超时，重连之后 缓冲区的数据会丢失 -->
        <writeTimeout>1 minute</writeTimeout>

        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <!-- 过滤的级别 -->
            <level>ERROR</level>
            <!-- 匹配时的操作：接收（记录） -->
            <onMatch>ACCEPT</onMatch>
            <!-- 不匹配时的操作：拒绝（不记录） -->
            <onMismatch>DENY</onMismatch>
        </filter>
        <!-- 如果需要配置监听则实现 net.logstash.logback.appender.listener.TcpAppenderListener 这个接口 -->
        <!-- <listener class="com.sd.promalertwebhook.listen.logback.MyTcpAppenderListener"/> -->
    </appender>

    <!--控制台输出-->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder charset="UTF-8">
            <pattern>${log.pattern}</pattern>
        </encoder>
    </appender>

    <!--记录INFO日志-->
    <appender name="FILE_INFO"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/info.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/info-%d{yyyy-MM-dd}.%i.log
            </fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <!--文件最大大小-->
                <maxFileSize>${file.max-filesize}</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--最大保存时间（天）-->
            <MaxHistory>30</MaxHistory>
            <!--日志总大小-->
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${log.pattern}</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <!-- 过滤的级别 -->
            <level>INFO</level>
            <!-- 匹配时的操作：接收（记录） -->
            <onMatch>ACCEPT</onMatch>
            <!-- 不匹配时的操作：拒绝（不记录） -->
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!--记录ERROR日志-->
    <appender name="FILE_ERROR"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/error.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/error-%d{yyyy-MM-dd}.%i.log
            </fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <!--文件最大大小-->
                <maxFileSize>${file.max-filesize}</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--最大保存时间（天）-->
            <MaxHistory>30</MaxHistory>
            <!--日志总大小-->
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${log.pattern}</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <!-- 过滤的级别 -->
            <level>ERROR</level>
            <!-- 匹配时的操作：接收（记录） -->
            <onMatch>ACCEPT</onMatch>
            <!-- 不匹配时的操作：拒绝（不记录） -->
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE_INFO"/>
        <appender-ref ref="FILE_ERROR"/>
        <appender-ref ref="STASH"/>
    </root>

    <!--日志级别-->
    <logger name="com.sd.logcenter" level="INFO"/>

</configuration> 
```

## 常见问题

- Logstash output Elasticsearch 索引时区问题
- 不能自动创建索引
    ```bash
    PUT /_cluster/settings
    {
        "persistent" : {
            "action": {
            "auto_create_index": "true"
            }
        }
    }
    ```

## 参考

- [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/index.html)
- [Kibana](https://www.elastic.co/guide/en/kibana/5.5/index.html)
- [Logstash](https://www.elastic.co/guide/en/logstash/5.5/index.html)
- [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder)
- [Logback configuration](http://logback.qos.ch/manual/configuration.html)
- [vscode-logstash-editor](https://marketplace.visualstudio.com/items?itemName=fbaligand.vscode-logstash-editor#review-details)