# 过程视图 (Process View)

> 描述系统运行时的并发结构、进程/线程模型、异步流程与交互序列。

---

## 1. Gateway 完整请求流程（时序图）

```plantuml
@startuml Process-Gateway-Full
skinparam sequenceMessageAlign center

participant Client
participant "proxy_server.py\nchat_completion()" as PS
participant "user_api_key_auth()" as Auth
participant "InternalUsageCache\n(Redis + Memory)" as Cache
participant "MaxBudgetLimiter\nParallelRequestLimiter" as Hooks
participant "Router\nrouter.py" as Router
participant "LiteLLM SDK\nlitellm.acompletion()" as SDK
participant "ProviderConfig\n.transform_*()" as Transform
participant "LLM Provider API" as LLM
participant "Logging\nlitellm_logging.py" as Log
participant "DBSpendUpdateWriter" as DBWriter
participant "PostgreSQL" as DB

== 请求认证阶段 ==
Client -> PS : POST /v1/chat/completions\nAuthorization: Bearer sk-xxx
PS -> Auth : user_api_key_auth(token)
Auth -> Cache : GET api_key:{hash(token)}
alt Cache Hit
  Cache --> Auth : KeyInfo (spend, limits, models)
else Cache Miss
  Auth -> DB : SELECT LiteLLM_VerificationToken WHERE token=hash
  DB --> Auth : KeyInfo
  Auth -> Cache : SET api_key:{hash} = KeyInfo (TTL=60s)
end
Auth --> PS : UserAPIKeyAuthInformation

== 前置 Hooks 阶段 ==
PS -> Hooks : await max_budget_limiter.async_pre_call_hook()
Hooks -> Cache : GET user_spend:{key_id}
Cache --> Hooks : current_spend
Hooks -> Hooks : spend < max_budget?
Hooks --> PS : pass / raise BudgetExceededError

PS -> Hooks : await parallel_request_limiter.async_pre_call_hook()
Hooks -> Cache : INCR rpm:{key_id}:{window}
Cache --> Hooks : rpm_count
Hooks -> Hooks : rpm_count < rpm_limit?
Hooks --> PS : pass / raise RateLimitError

== 路由阶段 ==
PS -> Router : route_llm_request(data, user_api_key_dict)
Router -> Router : get_available_deployment(model, routing_strategy)
note right: 路由策略: lowest_latency\nlowest_tpm_rpm / simple_shuffle

== SDK 调用阶段 ==
Router -> SDK : litellm.acompletion(model, messages, **kwargs)
SDK -> Transform : ProviderConfig.transform_request()
Transform --> SDK : provider-format payload
SDK -> LLM : HTTP POST (async)
LLM --> SDK : raw response (streaming or complete)
SDK -> Transform : ProviderConfig.transform_response()
Transform --> SDK : ModelResponse (OpenAI格式)

== 成本计算阶段 ==
SDK -> Log : update_response_metadata()
Log -> Log : _response_cost_calculator()\n= tokens × price_per_token
Log --> SDK : response._hidden_params["response_cost"] = $0.002

SDK --> Router : ModelResponse
Router --> PS : ModelResponse

== 响应 + 异步日志阶段 ==
PS -> PS : 提取 hidden_params["response_cost"]
PS --> Client : HTTP 200 ModelResponse\nx-litellm-response-cost: 0.002

PS ->> Log : async_success_handler() [异步, 不阻塞响应]
Log ->> DBWriter : update_database(token, response_cost)
DBWriter ->> Cache : LPUSH spend_queue:{token}
note right: 后台 job 每60秒\nbatch 写入 PostgreSQL
DBWriter ->> DB : INSERT SpendLogs (batch, async)
@enduml
```

---

## 2. SDK 直接调用流程

```plantuml
@startuml Process-SDK-Flow
participant "App Code" as App
participant "litellm.completion()" as Entry
participant "get_llm_provider()" as Resolve
participant "LLMCachingHandler\n(pre_call)" as Cache
participant "BaseLLMHTTPHandler" as Handler
participant "ProviderConfig" as Config
participant "AsyncHTTPHandler\n(httpx)" as HTTP
participant "LLM Provider" as LLM
participant "LLMCachingHandler\n(post_call)" as CachePost
participant "Logging callbacks\n(async)" as Log

App -> Entry : completion(model="claude-3-5-sonnet", messages=[...])
Entry -> Resolve : get_llm_provider(model) → "anthropic"
Entry -> Cache : async_get_cache(request_kwargs)
alt Cache Hit
  Cache --> Entry : cached ModelResponse
  Entry --> App : ModelResponse (from cache)
else Cache Miss
  Cache --> Entry : null
  Entry -> Handler : BaseLLMHTTPHandler.acompletion()
  Handler -> Config : AnthropicConfig.transform_request()
  Config --> Handler : Anthropic payload\n{messages, max_tokens, ...}
  Handler -> HTTP : POST https://api.anthropic.com/v1/messages
  HTTP -> LLM : HTTP request
  LLM --> HTTP : response JSON
  HTTP --> Handler : httpx.Response
  Handler -> Config : AnthropicConfig.transform_response()
  Config --> Handler : ModelResponse (OpenAI格式)
  Handler --> Entry : ModelResponse
  Entry -> CachePost : async_set_cache(response)
  Entry ->> Log : async_success_handler() [fire-and-forget]
  Entry --> App : ModelResponse
end
@enduml
```

---

## 3. 流式响应处理流程

```plantuml
@startuml Process-Streaming
participant "Client" as C
participant "proxy_server\nchat_completion()" as PS
participant "litellm SDK" as SDK
participant "LLM Provider\n(SSE stream)" as LLM
participant "CustomStreamWrapper" as Stream
participant "LoggingObj" as Log

C -> PS : POST /v1/chat/completions\n{stream: true}
PS -> SDK : litellm.acompletion(stream=True)
SDK -> LLM : HTTP POST with stream=True
LLM --> SDK : SSE stream (data: {...}\n)
SDK -> Stream : CustomStreamWrapper(response)

loop 每个 SSE chunk
  Stream -> LLM : read_next_chunk()
  LLM --> Stream : chunk bytes
  Stream -> Stream : parse SSE → ChatCompletionChunk
  Stream --> PS : yield ChatCompletionChunk
  PS --> C : data: {"choices":[{"delta":{"content":"..."}}]}
end

Stream -> Stream : 最后 chunk: finish_reason="stop"
Stream -> Log : async_success_handler(full_response)
Log ->> Log : 异步发送 callbacks (Langfuse/DD...)
@enduml
```

---

## 4. 路由器故障转移活动图

```plantuml
@startuml Process-Fallback
start
:收到 LLM 请求;
:Router.acompletion(model, messages);

:RoutingStrategy.get_available_deployment();
note right: 查询 DualCache\n的 TPM/RPM/latency 数据

if (找到可用 Deployment?) then (yes)
  :调用 litellm.acompletion(deployment);
  if (调用成功?) then (yes)
    :更新 deployment 延迟统计;
    :返回 ModelResponse;
    stop
  else (no, exception)
    :记录失败, 设置 cooldown;
    :检查 fallbacks 配置;
    if (有 fallback?) then (yes)
      :选择下一个 fallback model;
      :重新尝试;
    else (no)
      :抛出 litellm.APIError;
      stop
    endif
  endif
else (no)
  :抛出 openai.RateLimitError;
  stop
endif
@enduml
```

---

## 5. 预算重置后台 Job

```plantuml
@startuml Process-BudgetReset
participant "APScheduler\n(后台进程)" as Sched
participant "BudgetResetJob\n(budget_reset_job.py)" as Job
participant "PrismaClient" as Prisma
participant "PostgreSQL" as DB
participant "SlackAlerting" as Slack

loop 每10-12分钟
  Sched -> Job : reset_budget()
  Job -> Prisma : 查询 budget_reset_at <= now()
  Prisma -> DB : SELECT keys/teams/users\nWHERE budget_reset_at <= NOW()
  DB --> Prisma : 待重置记录列表
  
  loop 每条记录
    Job -> Prisma : UPDATE spend=0\nbudget_reset_at=now()+budget_duration
    Prisma -> DB : UPDATE
    Job -> Slack : send_alert_if_budget_reset()
  end
  
  Job --> Sched : 完成
end
@enduml
```

---

## 6. 并发模型概览

```plantuml
@startuml Process-Concurrency
skinparam componentStyle rectangle

node "LiteLLM Proxy Process" {

  component "FastAPI / Uvicorn\nasyncio event loop" as Loop {
    component "HTTP Worker\n(async coroutine)" as W1
    component "HTTP Worker\n(async coroutine)" as W2
    component "HTTP Worker\n(async coroutine)" as W3
  }

  component "APScheduler\n后台线程池" as Sched {
    component "update_spend job\n(60s)" as J1
    component "reset_budget job\n(10min)" as J2
    component "health_check job\n(continuous)" as J3
  }

  component "ThreadPoolExecutor\n(sync callback wrapper)" as TP

  Loop --> Sched : 共享 PostgreSQL / Redis 连接池
  Loop --> TP : run_in_executor (sync SDK calls)
}

database "Redis\n(共享状态)" as Redis
database "PostgreSQL\n(持久化)" as PG

Loop --> Redis : asyncio-redis (non-blocking)
Loop --> PG : prisma async client
Sched --> PG : 批量写入
@enduml
```
