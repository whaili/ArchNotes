# 过程视图 (Process View)

> 描述系统运行时的并发结构、进程/线程模型、异步流程与交互序列。

---

## 1. Gateway 完整请求流程

```mermaid
sequenceDiagram
    actor Client
    participant PS as proxy_server.py
    participant Auth as user_api_key_auth()
    participant Cache as InternalUsageCache (Redis+Memory)
    participant Hooks as MaxBudgetLimiter / ParallelRequestLimiter
    participant Router
    participant SDK as litellm.acompletion()
    participant Transform as ProviderConfig.transform_*()
    participant LLM as LLM Provider API
    participant Log as Logging (litellm_logging.py)
    participant DBWriter as DBSpendUpdateWriter
    participant DB as PostgreSQL

    rect rgb(230, 245, 255)
        Note over Client,DB: 请求认证阶段
        Client->>PS: POST /v1/chat/completions<br/>Authorization: Bearer sk-xxx
        PS->>Auth: user_api_key_auth(token)
        Auth->>Cache: GET api_key:{hash(token)}
        alt Cache Hit
            Cache-->>Auth: KeyInfo (spend, limits, models)
        else Cache Miss
            Auth->>DB: SELECT VerificationToken WHERE token=hash
            DB-->>Auth: KeyInfo
            Auth->>Cache: SET api_key:{hash} = KeyInfo TTL=60s
        end
        Auth-->>PS: UserAPIKeyAuthInformation
    end

    rect rgb(255, 245, 230)
        Note over PS,Hooks: 前置 Hooks 阶段
        PS->>Hooks: max_budget_limiter.async_pre_call_hook()
        Hooks->>Cache: GET user_spend:{key_id}
        Cache-->>Hooks: current_spend
        Hooks-->>PS: pass / raise BudgetExceededError

        PS->>Hooks: parallel_request_limiter.async_pre_call_hook()
        Hooks->>Cache: INCR rpm:{key_id}:{window}
        Cache-->>Hooks: rpm_count
        Hooks-->>PS: pass / raise RateLimitError
    end

    rect rgb(230, 255, 230)
        Note over PS,LLM: SDK 调用阶段
        PS->>Router: route_llm_request()
        Router->>SDK: litellm.acompletion()
        SDK->>Transform: ProviderConfig.transform_request()
        Transform-->>SDK: provider-format payload
        SDK->>LLM: HTTP POST (async)
        LLM-->>SDK: raw response
        SDK->>Transform: ProviderConfig.transform_response()
        Transform-->>SDK: ModelResponse (OpenAI 格式)
    end

    rect rgb(255, 230, 255)
        Note over SDK,DB: 成本计算 + 异步日志阶段
        SDK->>Log: update_response_metadata()
        Log->>Log: _response_cost_calculator()<br/>= tokens × price_per_token
        SDK-->>Router: ModelResponse + hidden_params[response_cost]
        Router-->>PS: ModelResponse
        PS-->>Client: HTTP 200 + x-litellm-response-cost header

        PS-->>Log: async_success_handler() [异步，不阻塞]
        Log-->>DBWriter: update_database(token, cost)
        DBWriter-->>Cache: LPUSH spend_queue:{token}
        DBWriter-->>DB: batch INSERT SpendLogs (每60s)
    end
```

---

## 2. SDK 直接调用流程

```mermaid
sequenceDiagram
    participant App as App Code
    participant Entry as litellm.completion()
    participant Resolve as get_llm_provider()
    participant CachePre as LLMCachingHandler (pre)
    participant Handler as BaseLLMHTTPHandler
    participant Config as ProviderConfig
    participant HTTP as AsyncHTTPHandler (httpx)
    participant LLM as LLM Provider
    participant CachePost as LLMCachingHandler (post)
    participant Log as Logging callbacks (async)

    App->>Entry: completion(model="claude-3-5-sonnet", messages=[...])
    Entry->>Resolve: get_llm_provider(model)
    Resolve-->>Entry: provider="anthropic"
    Entry->>CachePre: async_get_cache(request_kwargs)

    alt 缓存命中
        CachePre-->>Entry: cached ModelResponse
        Entry-->>App: ModelResponse（from cache）
    else 缓存未命中
        CachePre-->>Entry: null
        Entry->>Handler: BaseLLMHTTPHandler.acompletion()
        Handler->>Config: AnthropicConfig.transform_request()
        Config-->>Handler: Anthropic payload {messages, max_tokens, ...}
        Handler->>HTTP: POST https://api.anthropic.com/v1/messages
        HTTP->>LLM: HTTP request
        LLM-->>HTTP: response JSON
        HTTP-->>Handler: httpx.Response
        Handler->>Config: AnthropicConfig.transform_response()
        Config-->>Handler: ModelResponse (OpenAI 格式)
        Handler-->>Entry: ModelResponse
        Entry->>CachePost: async_set_cache(response)
        Entry-->>Log: async_success_handler() [fire-and-forget]
        Entry-->>App: ModelResponse
    end
```

---

## 3. 流式响应处理流程

```mermaid
sequenceDiagram
    actor Client
    participant PS as proxy_server.py
    participant SDK as litellm SDK
    participant LLM as LLM Provider (SSE)
    participant Stream as CustomStreamWrapper
    participant Log as LoggingObj

    Client->>PS: POST /v1/chat/completions {stream: true}
    PS->>SDK: litellm.acompletion(stream=True)
    SDK->>LLM: HTTP POST with stream=True
    LLM-->>SDK: SSE stream open

    loop 每个 SSE chunk
        SDK->>Stream: read_next_chunk()
        Stream->>LLM: 读取下一行
        LLM-->>Stream: chunk bytes
        Stream->>Stream: parse SSE → ChatCompletionChunk
        Stream-->>PS: yield ChatCompletionChunk
        PS-->>Client: data: {"choices":[{"delta":{"content":"..."}}]}
    end

    Stream->>Stream: finish_reason="stop"
    Stream-->>Log: async_success_handler(full_response)
    Log-->>Log: 异步发送 callbacks (Langfuse/DD...)
```

---

## 4. 路由器故障转移流程

```mermaid
flowchart TD
    A([收到 LLM 请求]) --> B[Router.acompletion]
    B --> C[RoutingStrategy.get_available_deployment\n查询 DualCache TPM/RPM/latency]
    C --> D{找到可用\nDeployment?}

    D -- 否 --> E[抛出 openai.RateLimitError]
    E --> Z([结束])

    D -- 是 --> F[调用 litellm.acompletion\ndeployment]
    F --> G{调用成功?}

    G -- 是 --> H[更新 deployment 延迟统计]
    H --> I([返回 ModelResponse])

    G -- 否 --> J[记录失败\n设置 cooldown]
    J --> K{有 fallback\n配置?}

    K -- 否 --> L[抛出 litellm.APIError]
    L --> Z

    K -- 是 --> M[选择下一个\nfallback model]
    M --> F
```

---

## 5. 预算重置后台 Job

```mermaid
sequenceDiagram
    participant Sched as APScheduler (后台线程)
    participant Job as BudgetResetJob
    participant Prisma as PrismaClient
    participant DB as PostgreSQL
    participant Slack as SlackAlerting

    loop 每 10-12 分钟
        Sched->>Job: reset_budget()
        Job->>Prisma: 查询 budget_reset_at <= now()
        Prisma->>DB: SELECT keys/teams/users WHERE budget_reset_at <= NOW()
        DB-->>Prisma: 待重置记录列表
        loop 每条记录
            Job->>Prisma: UPDATE spend=0, budget_reset_at=now()+duration
            Prisma->>DB: UPDATE
            Job-->>Slack: send_alert_if_budget_reset()
        end
        Job-->>Sched: 完成
    end
```

---

## 6. 并发模型概览

```mermaid
flowchart TB
    subgraph Process["LiteLLM Proxy Process"]
        subgraph EventLoop["FastAPI + Uvicorn — asyncio event loop"]
            W1["HTTP Worker\nasync coroutine"]
            W2["HTTP Worker\nasync coroutine"]
            W3["HTTP Worker\nasync coroutine"]
        end

        subgraph BgJobs["APScheduler — 后台线程池"]
            J1["update_spend\n每 60s"]
            J2["reset_budget\n每 10min"]
            J3["health_check\ncontinuous"]
        end

        TP["ThreadPoolExecutor\n同步回调包装器"]
    end

    Redis[("Redis\n共享速率/缓存状态")]
    PG[("PostgreSQL\n持久化数据")]

    W1 & W2 & W3 -->|asyncio-redis non-blocking| Redis
    W1 & W2 & W3 -->|prisma async client| PG
    J1 & J2 -->|批量写入| PG
    W1 & W2 & W3 -->|run_in_executor| TP

    note1["Proxy 无本地状态\n可无限水平扩展"]
```
