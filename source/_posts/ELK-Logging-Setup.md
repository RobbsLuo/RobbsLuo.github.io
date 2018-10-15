---
title: 监控接入：先把日志这摊事用 ELK 理顺
date: 2018-10-15 14:00:00
tags:
  - Career
  - ELK
  - DevOps
  - Logging
categories:
  - Career
description: 服务跑起来之后，下一个头疼的就是日志。这篇记我们怎么用 ELK 把分散在几十台 EC2 上的日志理顺，让排障不再靠 SSH 上去 grep。
---

## 日志这事，比发布更脏

上一篇写完发布，服务能跑起来了。但跑起来只是开始，服务跑起来之后会出事，出事之后你得能查。

最早 CoreTeam 的日志是什么状态呢？每个服务打到自己的 EC2 本地文件里，`/var/log/app.log` 之类的。一旦出问题，排查流程是这样的：

1. 问业务方"挂了哪个接口"，业务方说"不清楚，前端报 500"。
2. SSH 上某台 EC2，`tail -f /var/log/app.log`。
3. 翻几万行日志找错误。找不到？换一台 EC2 再来。
4. 几个小时过去了，用户已经骂街。

SSH 上服务器 grep 日志，是运维层面最昂贵的排障方式。不在于机器慢，在于人慢。

这种状态在十个服务、二十台机器的时候还能忍，到了几十个服务、上百台 EC2 的规模，就是事故级的工作流。所以我们决定：先把日志这一摊理顺。

注意，是先日志。metrics（CPU、内存、QPS 那一类）后面再上。为什么这个顺序？

因为业务问题 80% 是日志能告诉你的，某个请求报了什么错、走了哪个分支、传了什么参数。metrics 告诉你"系统慢了"，但日志告诉你"为什么慢"。先把"为什么"搞定。

## ELK 选型

ELK 是 Elasticsearch + Logstash + Kibana 的组合。2018 年那会儿基本是日志系统的标配。

- Elasticsearch：分布式全文搜索引擎，存日志、搜日志的核心。
- Logstash：日志解析、过滤、转发的管道。
- Kibana：可视化界面，查日志、画图表。

后来又加了 Filebeat，替代了 Logstash 直接在机器上抓日志的角色，Filebeat 更轻量，专门做"采集"这件事。

ELK 不是唯一选择，但它是 2018 年最稳的选择。自建 ELK 的人多、文档全、坑都被踩过了。内部团队没必要为了"工具先进性"自己趟雷。

整套架构长这样：

```
EC2 上的应用日志
   ↓ (Filebeat 采集)
Logstash (解析、过滤、标准化)
   ↓
Elasticsearch (存储 + 索引)
   ↓
Kibana (查询、可视化)
```

## 第一步：让日志先到 stdout

ELK 接入的第一步，不是搭 ES，而是让应用日志打到 stdout/stderr。

为什么这件事这么重要？因为容器化时代，应用日志打到 stdout 之后，Docker / ECS 会自动把它收集到 json-file driver，Filebeat 也能从 Docker logs 里抓。这样：

- 应用代码不需要关心日志往哪儿写。
- 容器挂了，日志不会丢在 ephemeral filesystem 里。
- 任何语言、任何框架，统一的接入方式。

```javascript
// Node.js 应用的标准做法
const logger = require('pino')();

// 错误日志打到 stdout
app.use((err, req, res, next) => {
  logger.error({
    msg: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method,
  });
  res.status(500).json({ error: 'internal_error' });
});
```

PHP、Python、Java、Go 同理。这一条约定的价值，比后面任何工具配置都大。

## 第二步：Filebeat 部署到每台机器

Filebeat 是个轻量的 agent，跑在每台 EC2（或每个 ECS 容器实例）上。配置文件大致是这样：

```yaml
# /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
      - /var/lib/docker/containers/*/*.log
    fields:
      env: prod
      project: coreteam
    fields_under_root: true

  - type: container
    paths:
      - /var/lib/docker/containers/*.log
    processors:
      - add_kubernetes_metadata: ~  # ECS 场景改成 add_docker_metadata

output.logstash:
  hosts: ["logstash.internal:5044"]
  loadbalance: true
  bulk_max_size: 1024
```

注意几个关键点：

- `fields_under_root`：把自定义字段放到顶层，方便 ES 里按 `env`、`project` 过滤。
- bulk_max_size：批量发送，别一条一条传，性能差十倍。
- Logstash 前面最好挂个负载均衡，方便横向扩展。

## 第三步：Logstash 做日志标准化

Filebeat 送过来的是原始日志，格式五花八门。Nginx 是它自己的格式，JSON 是另一回事，Java 的 stacktrace 是多行的。Logstash 的活儿就是把这一坨东西标准化。

```ruby
# logstash.conf (简化版)
input {
  beats {
    port => 5044
  }
}

filter {
  # 如果是 JSON 格式的业务日志
  if [type] == "json" or [log][file][path] =~ "app" {
    json {
      source => "message"
      remove_field => ["message"]
    }
  }

  # 多行 stacktrace 合并
  if [log][file][path] =~ "java" or [container][image] =~ "java" {
    multiline {
      pattern => "^\s"
      what => "previous"
    }
  }

  # 统一时间字段
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }

  # 加上地理信息（如果日志里有 IP）
  geoip {
    source => "client_ip"
  }
}

output {
  elasticsearch {
    hosts => ["es.internal:9200"]
    index => "coreteam-logs-%{+YYYY.MM.dd}"
    template_name => "coreteam"
  }
}
```

Logstash 最大的价值不是"转发"，是"标准化"。让 Nginx 日志、业务 JSON 日志、MySQL 慢查询日志在 ES 里都长一个样，能用同一套查询语言搜。

## 第四步：Elasticsearch 索引策略

ES 这边最关键的一个决定是：索引按天切分，加上 lifecycle management。

```json
// 索引模板
PUT _index_template/coreteam-logs
{
  "index_patterns": ["coreteam-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "project": { "type": "keyword" },
        "env": { "type": "keyword" },
        "message": { "type": "text" }
      }
    }
  }
}

// Lifecycle：7 天后迁到 warm，30 天后删除
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_age": "1d", "max_size": "50gb" }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink": { "number_of_shards": 1 }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

为什么按天切分？因为日志是时间序列数据：

- 最近的日志查询最频繁，放热节点。
- 7 天前的日志基本只用于排查，迁到便宜的 warm 节点。
- 30 天前的日志（除非有合规要求），该删就删。留着不删就是慢性自杀，磁盘和查询性能都会被拖垮。

不要想着"日志永久保留"。真正有用的日志是最近 7 天的，再往前的查询频率断崖式下降。把存档这件事交给 S3 + Athena，而不是让 ES 扛所有历史。

## Kibana：让业务方自己查

ELK 的最后一步是让业务方能用 Kibana。否则你搭得再漂亮，业务方不会查，照样 SSH 上去 grep。

我们做的事：

1. 建索引模式：`coreteam-logs-*`，按 `@timestamp` 排序。
2. 保存常用查询：按 `project`、`level:error` 这种业务方一键能用的查询。
3. 做几个 Dashboard：每个服务的错误率、P99 延迟、TOP 异常接口。
4. 写一份 Kibana 速查手册：教业务方怎么搜日志、怎么过滤、怎么导出。

最后这一步看着 low，但工具落地的最后一公里，永远是培训。文档没人看？那就在内部做一次分享会，把 Kibana 查询的基本套路讲一遍。

## 踩过的坑

第一个坑是日志量爆炸。接入 ELK 第一周，磁盘水位飙到 90%。某个服务 DEBUG 日志全开了，一天写了 50G。后来我们强制：生产环境只允许 INFO 级别以上，DEBUG 临时开要 PR review。

第二个坑是 Java 多行日志被截断。stacktrace 在 ES 里变成一行一行的，根本看不懂。后来用 Filebeat 的 multiline processor 解决：

```yaml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/java/*.log
    multiline:
      type: pattern
      pattern: '^\d{4}-\d{2}-\d{2}'   # 行首是日期才算新日志
      negate: true
      match: after
```

第三个坑是 Logstash 性能瓶颈。早期 Logstash 用 JVM，吃内存吃得厉害，日志高峰期会堆积。后来我们把一部分解析逻辑前置到 Filebeat 的 processors，让 Logstash 只做重活儿，吞吐量提了三倍。

## metrics 我们当时怎么处理

注意我一直没提 Prometheus、CloudWatch 这些。这是刻意的，那一阶段我们就是聚焦日志，metrics 是分开的一条线，跑在 AWS CloudWatch 上（因为是 ECS 原生支持的）。

CloudWatch 给我们的是 CPU、内存、网络 IO 这些"机器指标"。ELK 给我们的是"应用层发生了什么"。两者结合才能完整还原事故现场，但起步阶段先把 ELK 做扎实，这是性价比最高的。

## 跑起来之后

到 2018 年底，CoreTeam 的日志体系是这样：

- 所有业务日志统一进 ELK，SSH grep 这件事基本消失。
- 日志保留 30 天，超过的归档到 S3。
- Kibana 给业务方自己查，平台组不再当"日志快递员"。
- 出事故先看 Kibana，定位时间从小时级降到分钟级。

这一步最大的价值不是"搭了个 ELK"，而是改变了整个团队排障的工作方式。从"靠人去翻日志"到"靠工具搜日志"，这种转变的影响是深远的。

监控系统的成功标志，不是看 dashboard 多漂亮，而是看大家排障时第一反应去哪儿。第一反应打开 Kibana，这事就成了；第一反应 SSH 上服务器，那就还差得远。

下一篇是权限审计，谁能动生产环境这事，比监控更严肃。
