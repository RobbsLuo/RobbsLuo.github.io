---
title: Logging First: Wiring Up ELK
date: 2018-10-15 14:00:00
tags: [Career, ELK, DevOps, Logging]
categories:
  - Career
description: Once services were running, the next headache was logs. This is how we used ELK to make sense of logs scattered across dozens of EC2 instances, so debugging no longer meant SSHing in and grepping.
lang: en
---

## Logging is dirtier than release

The previous post covered release: services can run now. But running is just the start. **Once services run, things break, and when things break you need to investigate.**

What did CoreTeam's logging look like at first? Each service wrote to a local file on its EC2, something like `/var/log/app.log`. When something went wrong, the debugging flow was:

1. Ask the business team "which endpoint failed". They say "no idea, frontend got a 500".
2. SSH onto some EC2, `tail -f /var/log/app.log`.
3. Page through tens of thousands of lines looking for the error. Not there? Try another EC2.
4. Hours pass. Users are already yelling.

> **SSHing into a box to grep logs is the most expensive debugging workflow ops has.** Not because the machine is slow; because the human is slow.

This was tolerable at ten services and twenty machines. At dozens of services and a hundred-plus EC2 instances, it became an incident-grade workflow. So we decided: **logging first**.

Note, logging first. Metrics (CPU, memory, QPS) come later. Why this order?

**Because 80% of business issues are answerable from logs: what error a request threw, which branch it took, what parameters it received. Metrics tell you the system is slow; logs tell you why.** Nail the "why" first.

## Choosing ELK

ELK is Elasticsearch + Logstash + Kibana. Back in 2018 it was basically the default logging stack.

- **Elasticsearch**: distributed full-text search engine, the core that stores and searches logs.
- **Logstash**: the parsing, filtering and forwarding pipeline.
- **Kibana**: the UI for querying and charting.

We also added **Filebeat**, replacing Logstash's role of pulling logs directly off hosts. Filebeat is lighter and dedicated to collection.

> **ELK is not the only choice, but in 2018 it was the most stable one.** Lots of people self-hosting, deep docs, all the traps already stepped in. There was no reason for an internal team to blaze new trails for "tool novelty."

The architecture:

```
app logs on EC2
   ↓ (collected by Filebeat)
Logstash (parse, filter, standardize)
   ↓
Elasticsearch (storage + indexing)
   ↓
Kibana (query, visualization)
```

## Step one: stdout first

**The first step of ELK onboarding is making apps log to stdout/stderr. Standing up ES comes later.**

Why does this matter so much? Because in the container era, once logs hit stdout, Docker / ECS automatically captures them via the json-file driver, and Filebeat can pick them up from Docker logs. That way:

- Application code does not care where logs end up.
- If the container dies, logs do not die with the ephemeral filesystem.
- Every language, every framework, one consistent onboarding path.

```javascript
// Standard practice for a Node.js app
const logger = require('pino')();

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

Same story for PHP, Python, Java, Go. **The value of this single convention outweighs any tool configuration that comes after.**

## Step two: Filebeat on every machine

Filebeat is a lightweight agent running on each EC2 (or each ECS container instance). Its config looks roughly like:

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
      - add_kubernetes_metadata: ~  # swap for add_docker_metadata on ECS

output.logstash:
  hosts: ["logstash.internal:5044"]
  loadbalance: true
  bulk_max_size: 1024
```

Key points:

- **`fields_under_root`**: put custom fields at top level, so you can filter by `env` or `project` easily in ES.
- **bulk_max_size**: send in batches, not one row at a time. Order of magnitude better throughput.
- Put a load balancer in front of Logstash so you can scale it horizontally.

## Step three: standardize in Logstash

What Filebeat ships is raw logs in all sorts of formats. Nginx has its own format, JSON is something else entirely, Java stacktraces span multiple lines. **Logstash's job is to standardize this mess.**

```ruby
# logstash.conf (simplified)
input {
  beats {
    port => 5044
  }
}

filter {
  # Business logs in JSON
  if [type] == "json" or [log][file][path] =~ "app" {
    json {
      source => "message"
      remove_field => ["message"]
    }
  }

  # Merge multi-line Java stacktraces
  if [log][file][path] =~ "java" or [container][image] =~ "java" {
    multiline {
      pattern => "^\s"
      what => "previous"
    }
  }

  # Normalize the timestamp
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }

  # Geo enrichment if there's an IP
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

**The biggest value of Logstash is standardization, not forwarding.** Making Nginx logs, business JSON logs and MySQL slow logs all look the same in ES, searchable with one query language.

## Step four: ES index strategy

The most important decision on the ES side: **split indices by day and add lifecycle management.**

```json
// Index template
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

// Lifecycle: hot for 7 days, delete after 30
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

Why split by day? Because logs are **time-series data**:

- Recent logs are queried most often, keep them on hot nodes.
- Logs older than 7 days are mostly for forensics, move them to cheaper warm nodes.
- Logs older than 30 days (unless compliance requires otherwise) should be deleted. **Hoarding logs is slow suicide**: both disk and query performance suffer.

> **Do not try to keep logs forever.** The logs that actually matter are from the last 7 days; query frequency drops off a cliff after that. Let S3 + Athena handle archives, not ES.

## Kibana: hand it to the business teams

The last mile of ELK is making Kibana usable by business teams. **Otherwise you build something beautiful, they still cannot use it, and they go back to SSH + grep.**

What we did:

1. **Index pattern**: `coreteam-logs-*`, sorted by `@timestamp`.
2. **Saved searches**: queries like `project:X AND level:error` that business teams could click once.
3. **A few dashboards**: error rate per service, P99 latency, top failing endpoints.
4. **A Kibana cheat sheet**: how to search, how to filter, how to export.

The last item looks lowly, but **the last mile of tool adoption is always training**. Docs go unread? Run an internal session, walk through the basic Kibana patterns once.

## Pitfalls we hit

**Pitfall one: log volume explosion.** First week after onboarding, disk usage hit 90%. Some service had DEBUG logging fully on, writing 50GB a day. We then enforced: production is INFO and above only; opening DEBUG temporarily requires a PR.

**Pitfall two: Java multiline logs got truncated.** Stacktraces showed up as disconnected lines in ES and made no sense. Fix: Filebeat's multiline processor.

```yaml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/java/*.log
    multiline:
      type: pattern
      pattern: '^\d{4}-\d{2}-\d{2}'   # only lines starting with a date begin a new log
      negate: true
      match: after
```

**Pitfall three: Logstash as a bottleneck.** Logstash is JVM-based and memory hungry, with backlogs at log peaks. We moved some parsing up into Filebeat processors and let Logstash handle only the heavy lifting; throughput tripled.

## On metrics

I have deliberately not mentioned specific metrics tools. That is intentional: **that phase was about logging only**. Metrics ran as a separate track on AWS CloudWatch (native to ECS).

CloudWatch gave us "machine metrics": CPU, memory, network IO. ELK gave us "what happened at the application layer". Together they reconstruct an incident, but **starting with ELK firmly was the highest ROI move**.

## Where it landed us

By the end of 2018, CoreTeam's logging looked like:

- All business logs flowing into ELK. SSH + grep basically disappeared.
- 30-day retention in ES, older logs archived to S3.
- Kibana usable by business teams; platform stopped being the "log courier".
- Incidents start in Kibana; mean time to identify dropped from hours to minutes.

The biggest win here is **that the entire team's debugging workflow changed**, not "we stood up ELK". From "humans paging through logs" to "tools searching logs", the ripple effect is huge.

> **The success metric for a monitoring system is not how pretty the dashboard is, it is where people instinctively go when debugging.** If the first instinct is to open Kibana, you've made it. If it is SSH, you are not there yet.

Next post: permission auditing, who can touch production. A more serious topic than monitoring.
