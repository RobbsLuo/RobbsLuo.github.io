---
title: Making Reports Verifiable: Governing Calculation Rules
date: 2021-08-17 14:00:00
tags:
  - Career
  - Java
  - SpringBoot
categories:
  - Career
lang: en
description: How was a given number on a Collateral report computed? For a long time, only the person who wrote the stored procedure knew. After rule documentation and a Spring Boot query API, business stakeholders could finally verify every report themselves.
---

## The Stakeholder's Question

Anyone who has worked on a Collateral system has probably been through this conversation:

A business stakeholder is staring at a report and suddenly asks, "Where does this number come from?"

You open the code and realize the calculation logic is scattered across three stored procedures, two views, and a trigger, with a temp table doing transformations somewhere in between. You tell them "give me a minute," and then spend the afternoon digging through thousands of lines of SQL.

This happened more than once. Every time a report went out, someone would come asking about the methodology behind a number. **The numbers weren't wrong. Nobody could just explain how they were derived.**

> **The biggest problem with a report is rarely that it's miscalculated. It's that it's unexplainable. And an unexplainable number is a number the business won't trust.**

## Where the Problem Came From

Looking back, I traced it to three root causes:

**Only the code knew the rules.** The calculation logic lived entirely in stored procedures and SQL. Stakeholders can't read code, and there was nowhere to find a human-readable description of the rules.

**Methodology drifted silently.** Someone tweaks a line of SQL in a stored procedure, and the calculation subtly changes. No change log, no notification. When this month's number doesn't match last month's, nobody can explain why.

**No traceable inputs and outputs.** The report shows a final number, but what inputs fed each step and what transformations happened along the way — none of it was recorded. You couldn't even begin to verify because there was no starting point.

## What I Did About It

The core of this work was **making rules visible and the process queryable**. Three concrete things.

### First: Rule Documentation

I extracted the calculation rules for each report from the codebase and wrote them up as human-readable rule documents. Each rule entry includes:

- **Rule ID**: a unique identifier for quick lookup
- **Business meaning**: what this calculation represents in business terms (no specific formulas or coefficients)
- **Input data sources**: which tables and fields are used
- **Output field**: where the result lands in the report
- **Version and change log**: every methodology adjustment is recorded

```markdown
# Example rule document (sanitized)

## RULE-COLL-001: Collateral Valuation Summary

**Business meaning**: Aggregate the current valuation of all
collateral under each client.

**Input data sources**:
- `collateral_master` table: collateral base info
- `collateral_valuation` table: latest valuation snapshot

**Output field**: `report_summary.client_total_value`

**Version history**:
- v1.2 (2021-07-20) Fixed filter for cancelled collateral
- v1.1 (2021-06-05) Added secondary collateral inclusion
- v1.0 (2021-05-01) Initial version
```

Note what this deliberately leaves out: **specific formulas and coefficients**. Those are core business IP and don't belong in a general-purpose document. But where data comes from, where it goes, and when the methodology changed is exactly what stakeholders need to know.

### Second: Spring Boot Query API

Documentation alone isn't enough. It drifts from the code. So I built a Spring Boot API on top of a **rule registry**, letting stakeholders query the rule metadata behind any report directly.

```java
// Rule query API (sanitized)
@RestController
@RequestMapping("/api/rules")
public class RuleController {

    @Autowired
    private RuleRegistry ruleRegistry;

    /**
     * Fetch all calculation rules tied to a given report
     */
    @GetMapping("/report/{reportId}")
    public ApiResponse<List<RuleDTO>> getRulesByReport(
            @PathVariable String reportId) {
        List<RuleDTO> rules = ruleRegistry.findByReport(reportId);
        return ApiResponse.ok(rules);
    }

    /**
     * Fetch the version change history for a rule
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

Behind this API sits a **rule registry**: each rule has a corresponding registration entry in code, with its ID, version, data source description, and so on. The registry is bound to the actual calculation logic, so when someone changes the code, they update the registry entry at the same time. It doesn't eliminate drift entirely, but it dramatically reduces the gap between docs and code.

### Third: Report Traceability

When a report is generated, the system records which rule versions it used. A stakeholder holding a report can trace back:

- This report used RULE-COLL-001 v1.2
- Input data came from the 2021-08-17 snapshot of `collateral_master`
- If the methodology changed, the reason is noted

```java
// Record rule versions at report generation time (sanitized)
@Service
public class ReportGenerator {

    public Report generate(String reportType, LocalDate runDate) {
        List<Rule> activeRules = ruleRegistry.getActiveRules(reportType);
        Report report = calculationEngine.compute(activeRules, runDate);

        // Snapshot which rules and versions this report used
        report.setRuleSnapshot(activeRules.stream()
            .map(r -> new RuleRef(r.getId(), r.getVersion()))
            .collect(Collectors.toList()));

        return reportRepository.save(report);
    }
}
```

Now a stakeholder can take a report ID, hit the API, and see exactly which rules and versions produced that report. **The numbers became explainable.**

## The Payoff

The most visible change after launch: **the number of people asking "where does this number come from" dropped dramatically.**

Stakeholders can look up rule docs themselves, check version history themselves, verify input data themselves. The remaining questions are almost always about whether a rule needs adjusting, not whether a number is correct.

> **The best reporting system isn't the one that computes fastest. It's the one where the business can verify the numbers themselves.**

An unexpected bonus: the dev team benefited too. New hires no longer have to reverse-engineer methodology from thousands of lines of SQL. They read the rule docs and get up to speed fast. I didn't anticipate that, but it's a genuine dividend.

**Calculation rule governance is a trust problem.** Whether stakeholders trust your numbers depends entirely on whether you can explain them. The more transparent the explanation, the stronger the trust.
