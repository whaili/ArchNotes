# 场景视图 (Scenarios / Use Cases — "+1")

> 场景视图是 4+1 架构中的驱动力。以下用例覆盖系统核心功能，并与其余四个视图形成追踪关系。

---

## 核心用例总览

```mermaid
flowchart LR
    Dev(["应用开发者\nSDK User"])
    Admin(["AI 工程师\nProxy Admin"])
    EU(["终端用户\nEnd User"])
    LLM(["☁️ LLM Provider"])

    subgraph System["LiteLLM System"]
        UC01["UC-01\n直接 SDK 调用"]
        UC02["UC-02\n通过 Gateway 调用 LLM"]
        UC03["UC-03\n虚拟 Key 管理"]
        UC04["UC-04\n负载均衡与故障转移"]
        UC05["UC-05\n预算与限流管控"]
        UC06["UC-06\n响应缓存"]
        UC07["UC-07\n可观测性与日志"]
        UC08["UC-08\n多模型路由"]
    end

    Dev --> UC01
    Dev --> UC02
    Admin --> UC03
    Admin --> UC05
    Admin --> UC08
    EU --> UC02
    UC02 --> UC04
    UC02 --> UC05
    UC02 -.->|extend| UC06
    UC02 --> UC07
    UC01 --> LLM
    UC02 --> LLM
```

---

## UC-01：直接 SDK 调用

**参与者**：应用开发者 | **目标**：用统一接口调用任意 LLM，无需关心 Provider 差异

```mermaid
sequenceDiagram
    actor Developer
    participant SDK as litellm.completion()
    participant Resolve as utils.get_llm_provider()
    participant Handler as BaseLLMHTTPHandler
    participant Transform as ProviderConfig.transform_*()
    participant API as LLM Provider API

    Developer->>SDK: completion(model="gpt-4o", messages=[...])
    SDK->>Resolve: 解析 model → provider
    Resolve-->>SDK: provider="openai"
    SDK->>Handler: BaseLLMHTTPHandler.completion()
    Handler->>Transform: transform_request(OpenAI 格式)
    Transform-->>Handler: provider 格式 payload
    Handler->>API: HTTP POST /v1/chat/completions
    API-->>Handler: raw response
    Handler->>Transform: transform_response()
    Transform-->>Handler: ModelResponse (OpenAI 格式)
    Handler-->>SDK: ModelResponse
    SDK-->>Developer: ModelResponse
```

---

## UC-02：通过 Gateway 调用 LLM

**参与者**：终端用户 / 应用开发者 | **目标**：通过 HTTP 调用 Gateway，享受认证、路由、计费等企业特性

```mermaid
sequenceDiagram
    actor Client
    participant Proxy as proxy_server.py
    participant Auth as user_api_key_auth()
    participant Hook as Rate Limiter Hook
    participant Router
    participant SDK as litellm SDK
    participant LLM as LLM Provider

    Client->>Proxy: POST /v1/chat/completions<br/>Authorization: Bearer sk-xxx
    Proxy->>Auth: 验证 API Key
    Auth-->>Proxy: UserAPIKeyAuth (key info, limits)
    Proxy->>Hook: max_budget_limiter<br/>parallel_request_limiter
    Hook-->>Proxy: passed / rejected
    Proxy->>Router: route_llm_request()
    Router->>SDK: litellm.acompletion()
    SDK->>LLM: HTTP 请求
    LLM-->>SDK: 响应
    SDK-->>Router: ModelResponse
    Router-->>Proxy: ModelResponse + cost
    Proxy-->>Client: HTTP 200 + x-litellm-response-cost header
```

---

## UC-03：虚拟 Key 管理

**参与者**：AI 工程师 | **目标**：创建/吊销虚拟 Key，绑定预算、模型权限和团队

```mermaid
sequenceDiagram
    actor Admin
    participant API as POST /key/generate
    participant DB as PrismaClient (PostgreSQL)
    participant Cache as DualCache (Redis + Memory)

    Admin->>API: POST /key/generate<br/>{team_id, budget, models}
    API->>DB: INSERT LiteLLM_VerificationToken
    DB-->>API: token record
    API->>Cache: 写入 key cache
    Cache-->>API: ok
    API-->>Admin: {key: "sk-xxx", budget: $10}
```

---

## UC-04：负载均衡与故障转移

**参与者**：Router（自动） | **目标**：跨多个 LLM 部署实例实现负载均衡，失败时自动 fallback

```mermaid
sequenceDiagram
    participant R as Router
    participant Strategy as RoutingStrategy
    participant Cache as DualCache
    participant DA as Deployment A (OpenAI)
    participant DB as Deployment B (Azure)

    R->>Strategy: get_available_deployment()
    Strategy->>Cache: 查询各 deployment 的 TPM/RPM/latency
    Cache-->>Strategy: metrics
    Strategy-->>R: 选择 Deployment A
    R->>DA: 调用
    DA-->>R: 超时 / 失败
    R->>R: 触发 fallback 逻辑，设置 cooldown
    R->>DB: 调用 fallback deployment
    DB-->>R: 成功响应
```

---

## UC-05：预算与限流管控

**参与者**：MaxBudgetLimiter Hook | **目标**：实时阻断超过预算或速率限制的请求

```mermaid
sequenceDiagram
    participant Req as Request
    participant Budget as MaxBudgetLimiter
    participant Rate as ParallelRequestLimiter
    participant Cache as InternalUsageCache (Redis)

    Req->>Budget: pre_call_hook()
    Budget->>Cache: GET key_spend
    Cache-->>Budget: current_spend = $8.5
    Note right of Budget: check: $8.5 < $10 (limit) ✓
    Budget-->>Req: 允许继续

    Req->>Rate: pre_call_hook()
    Rate->>Cache: INCR + EXPIRE (RPM counter)
    Cache-->>Rate: rpm_count = 45
    Note right of Rate: check: 45 < 60 (limit) ✓
    Rate-->>Req: 允许继续

    Note over Req,Cache: 若超限，抛出 BudgetExceededError 或 RateLimitError
```

---

## UC-06：响应缓存

**参与者**：LLMCachingHandler | **目标**：对相同请求返回缓存响应，减少 LLM 调用成本

```mermaid
sequenceDiagram
    participant SDK as litellm.completion()
    participant Cache as LLMCachingHandler
    participant Backend as Cache Backend (Redis/Memory/S3)
    participant LLM as LLM Provider

    SDK->>Cache: pre_call_hook(request)
    Cache->>Backend: GET cache_key(model+messages+params)

    alt 缓存命中
        Backend-->>Cache: cached ModelResponse
        Cache-->>SDK: 直接返回（跳过 LLM 调用）
    else 缓存未命中
        Backend-->>Cache: null
        Cache-->>SDK: 继续调用
        SDK->>LLM: 实际 HTTP 请求
        LLM-->>SDK: ModelResponse
        SDK->>Cache: post_call_hook(response)
        Cache->>Backend: SET cache_key = response (TTL)
    end
```

---

## UC-07：可观测性与日志

**参与者**：CustomLogger 集成（Langfuse、Datadog 等） | **目标**：异步将调用详情、成本推送到外部观测平台

```mermaid
sequenceDiagram
    participant SDK as litellm SDK
    participant Log as Logging (litellm_logging.py)
    participant Handler as async_success_handler()
    participant Ext as Langfuse / Datadog / Slack
    participant Writer as DBSpendUpdateWriter
    participant DB as PostgreSQL

    SDK->>Log: success_handler(response, metadata)
    Log->>Log: _response_cost_calculator()
    Log->>Handler: async_success_handler()
    Handler-->>Ext: log_success_event(payload)
    Note right of Handler: 异步，不阻塞主链路
    Handler->>Writer: update_database(spend)
    Writer->>DB: batch INSERT SpendLogs
    Note right of Writer: 每 60s 批量写入
```
