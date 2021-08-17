---
title: 算法规则治理：让报表可验证
date: 2021-08-17 14:00:00
tags:
  - Career
  - Java
  - SpringBoot
categories:
  - Career
description: 抵押品报表的数字是怎么算出来的？以前只有写存储过程的人知道。做了规则文档化 + Spring Boot 查询接口后，业务方终于能自己核验每张报表了。
---

## 业务方的灵魂拷问

做 Collateral 项目的人大概都经历过这种对话：

业务方盯着一张某张报表，突然问："这个数字怎么来的？"

你翻开代码，发现计算逻辑散落在三个存储过程、两个视图、一个 trigger 里，中间还有张临时表做了数据转换。你跟业务方说"稍等我查一下"，然后自己在几千行 SQL 里翻了一下午。

这事儿不止发生过一次。每次出报表，总有人来问数字的口径。数字其实没算错，问题在于没人说得清数字是怎么算出来的。

报表最大的麻烦往往不是算错，而是不可解释。不可解释的数字，业务方不敢用。

## 问题出在哪

复盘下来，我觉得根源有三个：

规则只有代码知道。计算逻辑全在存储过程和 SQL 里，业务方看不懂代码，也没有地方查阅"人类可读"的规则说明。

口径会偷偷变。有人改了存储过程里的一行 SQL，口径就变了，但没有变更记录，也没有通知业务方。这个月和上个月的数字对不上，谁也说不清为什么。

没有可追溯的输入输出。报表给出一个最终数字，但中间每一步用了什么输入、做了什么转换，没有留痕。想验算都不知道从哪步开始。

## 我做了什么

这个事的核心不是重写算法，而是让规则可见、让过程可查。具体做了三件事。

### 第一件：规则文档化

把每张报表涉及的计算规则从代码里提取出来，写成业务方能看懂的规则文档。每条规则包含：

- 规则编号：唯一标识，方便定位
- 业务含义：这段计算在业务上代表什么（不含具体公式系数）
- 输入数据源：用了哪些表、哪些字段
- 输出字段：结果写到报表的哪个位置
- 版本与变更记录：每次口径调整都留痕

```markdown
# 规则文档示例（脱敏）

## RULE-COLL-001：抵押品估值汇总

**业务含义**：按客户维度汇总其名下所有抵押品的当前估值。

**输入数据源**：
- `collateral_master` 表：抵押品基础信息
- `collateral_valuation` 表：最新估值快照

**输出字段**：`report_summary.client_total_value`

**版本记录**：
- v1.2 (2021-07-20) 修正了注销状态抵押品的过滤条件
- v1.1 (2021-06-05) 新增二级抵押品的纳入逻辑
- v1.0 (2021-05-01) 初始版本
```

注意这里只写业务含义和数据流向，不写具体计算公式和系数。公式是业务核心资产，不该出现在通用文档里。但数据从哪来、到哪去、什么时候改过，这些是业务方需要知道的。

### 第二件：Spring Boot 查询接口

光有文档不够，文档会和代码脱节。我另外做了一套基于 Spring Boot 的查询接口，让业务方能通过 API 直接查到某张报表的规则元数据。

```java
// 规则查询接口（脱敏示意）
@RestController
@RequestMapping("/api/rules")
public class RuleController {

    @Autowired
    private RuleRegistry ruleRegistry;

    /**
     * 查询某张报表关联的所有计算规则
     */
    @GetMapping("/report/{reportId}")
    public ApiResponse<List<RuleDTO>> getRulesByReport(
            @PathVariable String reportId) {
        List<RuleDTO> rules = ruleRegistry.findByReport(reportId);
        return ApiResponse.ok(rules);
    }

    /**
     * 查询某条规则的版本变更历史
     */
    @GetMapping("/{ruleId}/versions")
    public ApiResponse<List<RuleVersionDTO>> getRuleVersions(
            @PathVariable String ruleId) {
        List<RuleVersionDTO> versions = ruleRegistry
            .findVersions(ruleId);
        return ApiResponse.ok(versions);
    }
}
```

这套接口背后是一个规则注册表：每条规则在代码里有对应的注册项，包含编号、版本、数据源描述等信息。规则注册表和实际计算逻辑绑定在一起，改代码时同步改注册信息，减少文档和代码脱节的概率。

### 第三件：报表可追溯

每张报表生成时，记录它用了哪些规则的哪个版本。业务方拿到一张报表，可以反查：

- 这张报表用了 RULE-COLL-001 v1.2
- 输入数据来自 `collateral_master` 表的 2021-08-17 快照
- 如果口径变了，会标注变更原因

```java
// 报表生成时记录规则版本（脱敏示意）
@Service
public class ReportGenerator {

    public Report generate(String reportType, LocalDate runDate) {
        List<Rule> activeRules = ruleRegistry.getActiveRules(reportType);
        Report report = calculationEngine.compute(activeRules, runDate);

        // 记录这张报表用了哪些规则的哪个版本
        report.setRuleSnapshot(activeRules.stream()
            .map(r -> new RuleRef(r.getId(), r.getVersion()))
            .collect(Collectors.toList()));

        return reportRepository.save(report);
    }
}
```

业务方现在可以拿着报表编号去查接口，看到这张报表当时用了哪些规则、哪个版本。数字变得可解释了。

## 效果

上线之后最直观的变化：来问"这个数字怎么来的"的人少了一大半。

业务方能自己查规则文档、自己看版本记录、自己验算输入数据。剩下的问题大多是规则本身需要调整，而不是"数字对不对"。

最好的报表系统不是算得最快的那个，而是让业务方能自证的那个。

另外有个意外的收获：规则文档化之后，开发团队自己也受益。新人接手时不用再翻几千行 SQL 逆向口径，看规则文档就能快速上手。这个我事先没想到，但确实是实打实的红利。

说到底，算法规则治理表面是技术问题，底子是信任问题。业务方信不信你的数字，取决于能不能解释清楚。你解释得越透明，信任就越牢靠。
