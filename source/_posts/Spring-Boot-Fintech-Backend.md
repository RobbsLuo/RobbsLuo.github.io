---
title: 工具线后端：用 Spring Boot 把 API 收敛起来
date: 2024-08-19 14:00:00
tags:
 - Career
 - Spring Boot
 - Java
 - 后端架构
categories:
 - Career
description: 13 个金融工具前端各自拉数据是地狱模式。这篇讲我们怎么用 Spring Boot multi-module 把 API 收敛到一套，统一响应格式、鉴权和审计。
---

接上篇 monorepo。前端 13 个工具塞一个仓库，那后端呢？总不能 13 套独立服务各自为政吧。前端收敛了，后端不收敛，等于把混乱从仓库搬到了网络层。

我们最后的选择是：一套 Spring Boot 服务，multi-module 结构，把 13 个工具的 API 全部收进来。这篇讲清楚怎么做的。

## 为什么是一个服务，不是 13 个微服务

我知道"微服务"听着高级。但在金融工具这个场景，13 个微服务是过度设计：

- 工具之间数据强耦合：理财计算工具和退休规划工具共用同一份客户档案，拆成两个服务就要做数据同步，反而更脆弱。
- 团队规模不够：维护 13 个服务的运维成本（监控、发布、链路追踪）我们这种团队扛不动。
- 金融监管要求统一审计：一个服务一套审计逻辑，监管来看你直接懵。

我的判断是：先做模块化的单体（Modular Monolith），等真正出现瓶颈再拆。这不是偷懒，是工程上的克制。

## multi-module 怎么分

Maven multi-module 是这次的主角。我们把整个服务分成几个模块，每个工具是一个独立的 module：

```
fintech-backend/
├── pom.xml                    # 根 POM，统一依赖版本
├── common/                    # 公共模块
│   ├── common-core/           # 统一响应、错误码、工具类
│   ├── common-web/            # 拦截器、过滤器、全局异常处理
│   ├── common-security/       # 鉴权、JWT、权限
│   └── common-audit/          # 审计日志
├── domain/                    # 领域模块
│   ├── domain-wealth/         # 理财领域
│   ├── domain-retirement/     # 退休领域
│   ├── domain-tax/            # 税务领域
│   └── ...
├── api-wealth-calc/           # 某理财计算工具的 API
├── api-retirement-plan/       # 某退休规划工具的 API
├── api-tax-optimize/          # 某税务优化工具的 API
├── app/                       # 启动模块，聚合所有 API
└── docker/
```

这个结构有个关键设计：common 和 domain 是独立 module，api 模块依赖它们但不互相依赖。也就是说 `api-wealth-calc` 不能直接调用 `api-retirement-plan` 的代码，要协作就走 domain 层或者事件。

根 POM 用 `dependencyManagement` 把版本统死：

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
    <!-- 内部模块版本也在这里统一管理 -->
    <dependency>
      <groupId>com.suite</groupId>
      <artifactId>common-core</artifactId>
      <version>${project.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

## 统一响应格式

13 个工具如果返回的 JSON 结构各不相同，前端 `api-client` 就得写 13 套适配代码。我们在 `common-core` 里定了一个统一信封：

```java
public class ApiResponse<T> {
    private int code;        // 业务码，0 表示成功
    private String message;
    private T data;
    private String traceId;  // 链路追踪 ID

    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(0, "OK", data, TraceContext.currentId());
    }

    public static <T> ApiResponse<T> fail(ErrorCode errorCode) {
        return new ApiResponse<>(errorCode.getCode(), errorCode.getMessage(), null, TraceContext.currentId());
    }
}
```

统一信封看着是小决策，但它让前端的错误处理从 13 套变成了 1 套。

前端那边的 `api-client` 就可以统一拦截：

```typescript
apiClient.interceptors.response.use((response) => {
  const body = response.data;
  if (body.code !== 0) {
    // 统一走错误处理
    return Promise.reject(new BizError(body.code, body.message));
  }
  return body.data;
});
```

## 全局异常处理

Spring 的 `@RestControllerAdvice` 在这里很关键。我们把所有业务异常收口到一个地方，避免每个 controller 重复写 try-catch：

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

注意最后那个 catch-all，金融系统绝不能把堆栈泄露给前端，只返回一个模糊的"系统错误"，详细信息走 traceId 在日志里查。

## 鉴权：每个工具有独立的 scope

13 个工具，不是每个用户都有权用所有工具。我们在 JWT 里塞了 `tools` 字段，标记当前用户可以用哪些工具：

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

注册的时候给每个 api module 的路径打上标记：

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

## 审计日志：金融场景的硬要求

这块我觉得是金融项目和普通项目最大的差别：每一次计算请求都必须留痕。监管来了要能说清楚"某年某月某日，某顾问用某工具做了什么计算，输入是什么"。

我们的做法是在 `common-audit` 里定义一个切面：

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

注意 `requestHash` 这里，我们不存原始入参（含客户敏感数据），只存 hash，既能追溯"是否同一份输入"，又不泄露客户信息。这个设计是和合规团队磨了好几轮才定的。

## 前后端契约：OpenAPI 生成 TS 类型

最后一个关键点：前后端的类型对齐。手写 TS 类型必然和后端 drift。我们用 springdoc 生成 OpenAPI spec，然后前端自动生成类型：

```xml
<!-- 后端 pom.xml -->
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.3.0</version>
</dependency>
```

前端 monorepo 的 `api-client` 包里：

```bash
# 从后端拉 OpenAPI spec，生成 TS 类型
openapi-typescript http://localhost:8080/v3/api-docs -o src/types/api.d.ts
```

类型不靠人维护，靠工具同步，这是 13 个工具规模下唯一可持续的做法。

## 小结

13 个工具收敛到一套 Spring Boot 后端，关键就几件事：multi-module 把模块化分清楚、common-core 统一 API 信封、common-audit 让审计可追溯。不是花哨的架构，但稳。

下一篇讲计算引擎，金融报表怎么做到"每一张都能复算"。
