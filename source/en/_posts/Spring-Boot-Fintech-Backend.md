---
title: Backend for a Tool Suite: APIs in Spring Boot
date: 2024-08-19 14:00:00
tags:
 - Career
 - Spring Boot
 - Java
 - Backend Architecture
categories:
 - Career
lang: en
description: Thirteen frontend tools each fetching their own data is hell mode. Here is how we used a Spring Boot multi-module service to unify the API surface, response format, auth and audit trail.
---

Following the previous monorepo post. The frontend lives in one repo with thirteen tools; what about the backend? Thirteen independent services each going their own way was never on the table.

> **Unify the frontend but leave the backend fragmented, and all you have done is move chaos from the repo to the network layer.**

We ended up with a single Spring Boot service using a multi-module structure that absorbs all thirteen tools' APIs. This post explains how.

## Why one service, not thirteen microservices

I know "microservices" sounds sophisticated. But in the financial-tools scenario, thirteen microservices is over-engineering.

- Tools share data heavily. The wealth calculation tool and the retirement planning tool read the same client profile. Split them into two services and you immediately need data sync, which is fragile.
- The team is not big enough. Operating thirteen services (monitoring, releases, distributed tracing) is more than we could carry.
- Finance regulation demands unified audit. One audit logic per service means you are lost when the regulator shows up.

My take: start with a modular monolith. Split only when a real bottleneck appears. This is not laziness, it is engineering restraint.

## How multi-module is sliced

Maven multi-module does the heavy lifting here. The service is sliced into modules, each tool owning an independent module:

```
fintech-backend/
├── pom.xml                    # root POM, unified versions
├── common/                    # shared modules
│   ├── common-core/           # unified response, error codes, utilities
│   ├── common-web/            # interceptors, filters, global exception handling
│   ├── common-security/       # auth, JWT, permissions
│   └── common-audit/          # audit log
├── domain/                    # domain modules
│   ├── domain-wealth/
│   ├── domain-retirement/
│   ├── domain-tax/
│   └── ...
├── api-wealth-calc/           # API for a wealth calculation tool
├── api-retirement-plan/       # API for a retirement planning tool
├── api-tax-optimize/          # API for a tax optimization tool
├── app/                       # bootstrap module, aggregates all APIs
└── docker/
```

The critical design here: common and domain are independent modules, and api modules depend on them but never on each other. `api-wealth-calc` cannot call into `api-retirement-plan` directly; if they need to cooperate, they go through the domain layer or events.

The root POM uses `dependencyManagement` to lock every version:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.2.5</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <!-- internal module versions also unified here -->
    <dependency>
      <groupId>com.suite</groupId>
      <artifactId>common-core</artifactId>
      <version>${project.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

## Unified response envelope

If thirteen tools each return a different JSON shape, the frontend `api-client` has to write thirteen adapters. We defined a single envelope in `common-core`:

```java
public class ApiResponse<T> {
    private int code;        // business code, 0 means success
    private String message;
    private T data;
    private String traceId;  // tracing id

    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(0, "OK", data, TraceContext.currentId());
    }

    public static <T> ApiResponse<T> fail(ErrorCode errorCode) {
        return new ApiResponse<>(errorCode.getCode(), errorCode.getMessage(), null, TraceContext.currentId());
    }
}
```

> **The unified envelope looks like a small decision, but it collapses thirteen sets of frontend error handling into one.**

The frontend interceptor then becomes trivial:

```typescript
apiClient.interceptors.response.use((response) => {
  const body = response.data;
  if (body.code !== 0) {
    return Promise.reject(new BizError(body.code, body.message));
  }
  return body.data;
});
```

## Global exception handling

Spring's `@RestControllerAdvice` is essential here. It funnels every business exception into one place so controllers stop repeating try-catch:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)
    public ApiResponse<?> handleBiz(BizException e) {
        return ApiResponse.fail(e.getErrorCode());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse<?> handleValidation(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ApiResponse.fail(ErrorCode.VALIDATION_FAILED.withMessage(msg));
    }

    @ExceptionHandler(Exception.class)
    public ApiResponse<?> handleUnknown(Exception e) {
        log.error("unhandled exception, traceId={}", TraceContext.currentId(), e);
        return ApiResponse.fail(ErrorCode.INTERNAL_ERROR);
    }
}
```

Note the catch-all at the end: a financial system must never leak a stack trace to the frontend. It returns a vague "system error" and logs the detail keyed by traceId.

## Auth: each tool has its own scope

Thirteen tools, and not every user is entitled to all of them. We put a `tools` claim inside the JWT marking which tools the current user may access:

```java
public class ToolAccessInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object handler) {
        String tool = req.getHeader("X-Tool-Id");
        UserContext user = UserContext.fromRequest(req);
        if (!user.hasToolAccess(tool)) {
            resp.setStatus(403);
            return false;
        }
        return true;
    }
}
```

Registered against each api module's path prefix:

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(toolAccessInterceptor())
                .addPathPatterns("/api/wealth/**", "/api/retirement/**", "/api/tax/**");
    }
}
```

## Audit log: a hard requirement in finance

This is where I think financial projects diverge most from ordinary ones: every calculation request must leave a trace. When the regulator arrives, you must be able to explain "on such date, such advisor used such tool to perform such calculation, with such inputs".

We define an aspect in `common-audit`:

```java
@Aspect
@Component
public class AuditAspect {

    @Autowired
    private AuditLogRepository auditRepo;

    @Around("@annotation(audited)")
    public Object audit(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
        Object result = pjp.proceed();
        AuditLog log = AuditLog.builder()
            .userId(UserContext.currentId())
            .toolId(audited.tool())
            .action(audited.action())
            .requestHash(DigestUtils.sha256Hex(toJson(pjp.getArgs())))
            .timestamp(Instant.now())
            .build();
        auditRepo.save(log);
        return result;
    }
}
```

Notice `requestHash`: we do not store the raw input (which contains sensitive client data), only its hash. This lets us prove "was this the same input as last time" without leaking client information. That design took several rounds with the compliance team to settle on.

## Frontend-backend contract: OpenAPI generates TS types

One last key point: keeping the types in sync. Hand-writing TS types will inevitably drift from the backend. We use springdoc to emit an OpenAPI spec, and the frontend generates types from it automatically.

```xml
<!-- backend pom.xml -->
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.3.0</version>
</dependency>
```

Inside the frontend `api-client` package:

```bash
# pull OpenAPI spec from backend, emit TS types
openapi-typescript http://localhost:8080/v3/api-docs -o src/types/api.d.ts
```

> **Types should not be maintained by humans. They should be synced by tooling; that is the only sustainable approach at thirteen-tool scale.**

## Wrapping up

Collapsing thirteen tools into one Spring Boot backend comes down to clean modularization (multi-module), a unified API envelope (common-core), and a traceable audit log (common-audit). Not flashy, just solid.

The next post covers the calculation engine: how financial reports become fully reproducible.
