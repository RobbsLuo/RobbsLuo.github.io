---
title: A Calculation Engine That Makes Reports Reproducible
date: 2024-10-15 11:00:00
tags:
 - Career
 - Calculation Engine
 - Reproducibility
 - Engineering Architecture
categories:
 - Career
lang: en
description: In finance, reports must be reproducible — given the same input and engine version, the output must match forever. This post covers only the engineering architecture, never the formulas.
---

This is the part of the whole tool line that feels most "financial" to me: the calculation engine.

Let me be clear up front: **this post does not touch any formulas, coefficients or rate definitions**. Those are business secrets and not mine to share. What I cover is engineering: how to make a financial report computed three months ago produce the exact same result when you re-run it today.

## What "reproducible" means

The biggest gap between financial tools and ordinary tools boils down to one phrase: **reproducible**.

> **Same input + same engine version = same output. Always.**

Why does this matter so much? Because financial businesses face regulatory audit. An auditor asks: "How was that wealth planning report from half a year ago computed?" You cannot shrug and say "the algorithm was upgraded, it comes out differently now." That is a serious incident.

So our engine design revolves entirely around this single goal.

## Three key design decisions

### 1. Input snapshot

Every time a user kicks off a calculation, we do not just shovel parameters into the calculation function. We first capture a full input snapshot:

```java
public class CalculationRequest {
    private String requestId;
    private String engineVersion;
    private String toolId;
    private String inputSnapshot;    // full JSON of all inputs
    private Instant requestedAt;
    private String userId;
}
```

`inputSnapshot` is a complete JSON containing every input that calculation required. **It is bound to the requestId and stored permanently**.

Why store the full snapshot instead of a reference? Because client data changes. If you only store a `clientId`, three months later the client profile has been updated, and re-running the calculation produces a different result. A snapshot means "freeze the world at that moment".

### 2. Engine versioning

Every calculation result must be bound to an engine version. Our versioning is semantic:

```java
public class EngineVersion {
    private int major;    // formula logic change (requires re-audit)
    private int minor;    // new calculation dimension (backward compatible)
    private int patch;    // bug fix
}
```

> **A major bump means the report's calculation basis has changed, and that requires sign-off from compliance before it can ship.**

The engine itself is a stateless, pure-functional module:

```java
public interface CalculationEngine {
    CalculationResult compute(CalculationInput input);
    EngineVersion version();
}
```

The crucial part: **multiple versions coexist; old versions are never overwritten**. A registry maintains a version table:

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

The report stores `engineVersion`, and re-calculation uses the corresponding version, never the latest.

### 3. Calculation log: traceable intermediate steps

Having the result is not enough. Auditors want to know "where did this number come from". So the engine emits a step-by-step log alongside every run:

```java
public class CalculationResult {
    private String requestId;
    private BigDecimal finalValue;
    private List<CalcStep> steps;
    private EngineVersion engineVersion;
    private Instant computedAt;
}

public class CalcStep {
    private String stepName;           // desensitized, generic name
    private String inputSummary;
    private BigDecimal stepOutput;
    private String note;
}
```

Note that `stepName` uses desensitized generic labels, things like "phase one aggregation" or "entitlement conversion". These are engineering tags, **never exposing the actual formula logic**. Auditors see the skeleton of the calculation flow but not the formula details.

Where is this log stored? Directly in a PostgreSQL JSONB column for easy querying. At higher volume you could move to a dedicated time-series store, but at our scale Postgres is plenty.

## The full re-calculation flow

Wiring the three pieces together, the flow looks like this:

```
[Original report]
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
    [Fresh CalculationResult]
            │
            ▼
    Compare against original → exact match ✓
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

The code looks simple, but the implication is heavy. **Any report, at any time, can be independently verified.**

## Traps we hit

**Trap one: the BigDecimal precision trap.** Financial computation in Java must use `BigDecimal`, but `equals` and `compareTo` behave differently. `new BigDecimal("1.0").equals(new BigDecimal("1.00"))` is false, while `compareTo` returns 0. We hit this during result comparison early on and spent an afternoon tracking it down.

```java
// wrong: misled by scale
if (a.equals(b)) { ... }

// correct: compares numeric value only
if (a.compareTo(b) == 0) { ... }
```

**Trap two: floating point must never touch the calculation pipeline.** A teammate took a shortcut using `double` for an intermediate conversion, and on a boundary value the result drifted by 0.01. Sounds small, but at millions of clients that is a serious incident. We added a static check forbidding `double` and `float` in the calculation module.

**Trap three: timezones making "the same day" inconsistent.** If dates in the snapshot are not pinned to a timezone, re-deploying to a different region makes re-calculation drift. We standardized on UTC storage and convert at presentation time.

## Wrapping up

The core of a calculation engine is **engineering discipline**:

- Freeze the input (snapshot)
- Lock the version (never overwrite old versions)
- Leave a trail (calculation log)

**A reproducible report is the real asset of a financial system.** Next up is the component library: how 10+ components support both UMD and an npm package simultaneously.
