---
title: DDL/DML 评审：索引、锁、迁移风险一张表说清
date: 2022-02-15 14:00:00
tags:
  - Career
categories:
  - Career
description: DAL 小组评审 DDL/DML 时到底在看什么？一张清单把索引、锁、迁移风险全过一遍。
---

做 DAL 小组评审做久了，我发现大多数 SQL 问题其实就那么几类。索引没用对、锁没估准、迁移没想清楚后果。新来的人一开始评审总抓不住重点，东看一眼西看一眼，漏掉关键点。

后来我把这些东西整理成了一张清单，评审的时候照着过。这篇就把这张清单摊开来讲讲。

## DDL 评审清单

### 1. 索引设计

这是重头戏。80% 的慢查询都是索引没设计好。

**要看的东西：**

- 新加的字段，查询条件里会不会用到？会用到就该考虑加索引。
- 加的索引基数（cardinality）够不够？一个 status 字段只有 0/1 两个值，单独建索引基本没用。
- 有没有联合索引可以替代多个单列索引？联合索引的列顺序很关键，等值条件在前，范围条件在后。

```sql
-- 反例：单独给低基数字段建索引，基本没用
CREATE INDEX idx_status ON t_xxx_orders(status);

-- 正例：联合索引，高基数在前
CREATE INDEX idx_uid_status ON t_xxx_orders(user_id, status);
```

- 有没有冗余索引？比如已经有 `(user_id, status)` 了，再单独建 `(user_id)` 就是浪费。

### 2. 字段类型

选错类型后患无穷。我见过太多用 `VARCHAR(255)` 装状态值的，明明 `TINYINT` 就够了。

- 状态值、枚举：`TINYINT` 或 `SMALLINT`，别用字符串。
- 金额：`DECIMAL`，别用 `FLOAT`，浮点精度问题够你喝一壶。
- 时间：`DATETIME` 还是 `TIMESTAMP` 要想清楚，`TIMESTAMP` 有 2038 问题，但占用空间小一半。

### 3. 锁的影响

MySQL 的 DDL 不是无锁的。

- MySQL 5.6 之前，`ALTER TABLE` 会锁全表，读写都阻塞。
- 5.6 之后，加了 Online DDL，大部分操作可以"在线"做，但不是所有操作都支持。比如给一个已有的大表加 `NOT NULL` 列，照样锁表。
- 大表（百万行以上）一律走 pt-online-schema-change 或 gh-ost，别裸跑 DDL。

```bash
# gh-ost 比 pt-osc 更现代，触发器换成 binlog 订阅
gh-ost \
  --alter "ADD COLUMN remark VARCHAR(200)" \
  --database=t_xxx --table=orders \
  --execute
```

### 4. 迁移风险

这一块新人最容易忽略。加了字段之后，老代码会不会挂？

举个例子：原来一个接口返回订单的 JSON，现在多了一个字段，下游服务反序列化的 POJO 里没有这个字段。大多数情况 Jackson 会忽略未知字段，但如果配置了 `FAIL_ON_UNKNOWN_PROPERTIES`，直接炸。

DDL 上线前必须确认：

- 新字段有没有默认值？
- 老代码读到新字段会不会报错？
- 下游服务有没有兼容性处理？
- 回滚方案是什么？删字段还是留着不管？

## DML 评审清单

### 1. UPDATE / DELETE 的 WHERE 条件

这是事故高发区。我见过有人写 `UPDATE t_xxx_orders SET status = 1;` 忘了加 `WHERE`，一秒钟全表更新，直接 gg。

评审时必看：

- 有没有 `WHERE` 条件？（听起来像废话，但真有人忘）
- `WHERE` 条件走的是索引还是全表扫描？`EXPLAIN` 一下就知道了。
- 影响行数预估多少？如果 `EXPLAIN` 的 `rows` 列显示百万级，得停下来想想。
- 大批量更新要分批做，别一把梭。

```sql
-- 大批量更新分批做，每批 1000 行
UPDATE t_xxx_orders
SET status = 2
WHERE status = 1 AND id <= 1000000
LIMIT 1000;
```

### 2. 事务大小

一个事务里更新的行数太多，会导致：

- undo log 膨胀，占空间。
- 长事务阻塞其他查询，锁等待飙升。
- 主从延迟拉大。

原则：一个事务别超过 1 万行更新。超了就拆。

### 3. INSERT 批量 vs 单条

批量 `INSERT` 性能比循环单条 `INSERT` 高几十倍，这个不用多解释。

```sql
-- 反例：循环单条插入
INSERT INTO t_xxx_log(user_id, action) VALUES (1, 'login');
INSERT INTO t_xxx_log(user_id, action) VALUES (2, 'login');

-- 正例：批量插入
INSERT INTO t_xxx_log(user_id, action) VALUES (1, 'login'), (2, 'login');
```

## 一张表总结

| 检查项 | DDL | DML | 严重程度 |
|--------|-----|-----|---------|
| WHERE 条件走索引？ | - | 必查 | 致命 |
| 锁表风险？ | 必查 | 看情况 | 高 |
| 字段类型合理？ | 必查 | - | 中 |
| 影响行数可控？ | 必查 | 必查 | 高 |
| 回滚方案？ | 必查 | 必查 | 致命 |
| 下游兼容性？ | 必查 | 看情况 | 高 |

## 最后说一句

清单这东西，关键是每次都用，不能挑着用。

评审的目的不是挑毛病，是帮你把那些"应该没事"变成"确定没事"。

我见过太多人觉得"这条 SQL 简单，应该没问题"，然后就出事了。简单的东西恰恰最容易翻车，因为你放松了警惕。养成习惯，每次评审都照着清单过一遍，不管 SQL 长什么样。
