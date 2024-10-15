---
title: 金融计算引擎：让每张报表都能复算
date: 2024-10-15 11:00:00
tags:
 - Career
 - 计算引擎
 - 可复算
 - 工程架构
categories:
 - Career
description: 金融场景下报表必须可复算——给定输入和引擎版本，结果完全一致。这篇只讲工程架构怎么落地，不碰计算公式。
---

这一篇是整个工具线里我觉得最"金融"的一块：计算引擎。

先说清楚：这篇不涉及任何计算公式、系数、利率口径。那些是业务机密，也不是我该讲的。我讲的是工程：怎么让一张金融报表，三个月后回头看，还能算出一模一样的结果。

## 什么是"可复算"

金融工具和普通工具最大的区别就在这四个字：可复算（Reproducible）。同一份输入配上同一个引擎版本，输出永远一致。

为什么这么讲究？因为金融业务要面对监管审计。审计人员会问："半年前那张理财规划报表，是怎么算出来的？"你不能耸耸肩说"算法升级了，现在算出来不一样了"，那就出大事了。

所以我们的引擎设计，从头到尾都围绕这一个目标。

## 关键设计

### 1. 输入快照

用户每次发起计算请求，我们不是直接把参数丢进计算函数就完事。而是先做一份完整的输入快照：

```java
public class CalculationRequest {
    private String requestId;        // 唯一标识
    private String engineVersion;    // 引擎版本号
    private String toolId;           // 哪个工具发起的
    private String inputSnapshot;    // 完整入参的 JSON 快照
    private Instant requestedAt;
    private String userId;
}
```

`inputSnapshot` 是一份完整的 JSON，包含了那次计算需要的所有入参。这份快照和 requestId 绑定，永久存储。

为什么要存完整快照而不是引用？因为客户数据会变。三个月后客户档案更新了，如果你只存一个 `clientId`，回头复算的结果就和原来不一致了。快照意味着"冻结那一刻的世界"。

### 2. 引擎版本化

每次计算结果都必须绑定一个引擎版本号。我们的版本规则是语义化的：

```java
public class EngineVersion {
    private int major;    // 公式逻辑变更（需要重新审计）
    private int minor;    // 新增计算维度（向前兼容）
    private int patch;    // Bug 修复
}
```

major 版本变更意味着报表口径变了，这是需要合规团队签字才能上线的事。

引擎本身是个无状态的纯函数式模块：

```java
public interface CalculationEngine {
    CalculationResult compute(CalculationInput input);
    EngineVersion version();
}
```

关键在于我们不覆盖旧版本，而是让多个版本共存。注册中心维护一个版本表：

```java
@Component
public class EngineRegistry {
    private final Map<String, CalculationEngine> engines = new ConcurrentHashMap<>();

    @PostConstruct
    public void init() {
        register("1.0.0", new WealthEngineV1());
        register("1.1.0", new WealthEngineV1_1());
        register("2.0.0", new WealthEngineV2());
    }

    public CalculationEngine get(String version) {
        CalculationEngine engine = engines.get(version);
        if (engine == null) {
            throw new BizException(ErrorCode.ENGINE_VERSION_NOT_FOUND);
        }
        return engine;
    }
}
```

报表里存了 `engineVersion`，复算时用对应版本的引擎跑一遍，不是用最新版。

### 3. 计算日志：中间步骤可追溯

光有结果不够。审计要看的是"这个数字怎么来的"。所以引擎每次跑完会顺手吐出一份 step-by-step 的计算日志：

```java
public class CalculationResult {
    private String requestId;
    private BigDecimal finalValue;        // 最终结果
    private List<CalcStep> steps;         // 计算步骤
    private EngineVersion engineVersion;
    private Instant computedAt;
}

public class CalcStep {
    private String stepName;              // 步骤名（脱敏的泛化名称）
    private String inputSummary;          // 这一步的入参摘要
    private BigDecimal stepOutput;        // 这一步的输出
    private String note;                  // 备注
}
```

注意 `stepName` 用的是脱敏后的泛化名称，比如 "阶段一汇总""权益折算"这种工程标签，绝不暴露具体算法口径。审计能看到计算流程的骨架，但看不到公式细节。

这套日志存在哪？直接落 PostgreSQL 的 JSONB 字段，查询方便。量大的时候可以考虑单独的时序存储，但我们目前的规模 PG 够用。

## 复算的完整流程

把上面三块串起来，复算流程是这样的：

```
[原始报表]
    │
    ├── requestId: "REQ-2024-001234"
    ├── engineVersion: "1.1.0"
    └── inputSnapshot: {...}
            │
            ▼
    [EngineRegistry.get("1.1.0")]
            │
            ▼
    [engine.compute(snapshot)]
            │
            ▼
    [新的 CalculationResult]
            │
            ▼
    与原始报表对比 → 完全一致 ✓
```

```java
public class RecalculationService {

    public RecalcResult verify(String originalRequestId) {
        CalculationRequest original = requestRepo.findById(originalRequestId);
        CalculationEngine engine = engineRegistry.get(original.getEngineVersion());

        CalculationInput input = CalculationInput.fromJson(original.getInputSnapshot());
        CalculationResult fresh = engine.compute(input);

        boolean matches = fresh.getFinalValue().compareTo(
            original.getResult().getFinalValue()
        ) == 0;

        return new RecalcResult(original, fresh, matches);
    }
}
```

这段代码看着简单，但背后的含义很重：任何一张报表，任何时候都能被独立验证。

## 踩过的坑

坑一：BigDecimal 的精度陷阱。Java 里做金融计算必须用 `BigDecimal`，但 `equals` 和 `compareTo` 行为不一致：`new BigDecimal("1.0")` 和 `new BigDecimal("1.00")` 用 equals 是 false，用 compareTo 才是 true。早期复算对比时踩过这个坑，排查了一下午。

```java
// 错误：会被 scale 误导
if (a.equals(b)) { ... }

// 正确：只比较数值大小
if (a.compareTo(b) == 0) { ... }
```

坑二：浮点数绝对不能出现在计算链路里。有同事图省事用 `double` 做了中间转换，结果在某个边界值上偏差了 0.01。听起来不多，但金融场景下 0.01 的偏差放大到百万级客户就是大事故。后来加了静态检查，禁止计算模块使用 `double`/`float`。

坑三：时区导致"同一天"不一致。快照里的日期如果没固定时区，服务器换了部署区域后复算结果就飘了。最后我们统一用 UTC 存储，展示时再转。

## 小结

计算引擎这事的核心不是算法多精妙，而是工程纪律：

- 输入冻结（快照）
- 版本锁定（不覆盖旧版本）
- 过程留痕（计算日志）

能复算的报表，才是金融系统真正的资产。下一篇讲组件库，10+ 个组件怎么同时支持 UMD 和 npm 包两种集成方式。
