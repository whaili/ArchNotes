# 场景视图 (Scenarios / Use Cases — "+1")

> 场景视图是 4+1 架构中的驱动力。以下用例覆盖系统核心功能，并与其余四个视图形成追踪关系。

---

## 核心用例总览

```plantuml
@startuml UC-Overview
left to right direction
skinparam packageStyle rectangle

actor "应用开发者\n(SDK User)" as Dev
actor "AI 工程师\n(Proxy Admin)" as Admin
actor "终端用户\n(End User)" as EU
actor "LLM Provider\n(OpenAI/Anthropic/...)" as LLM

rectangle "LiteLLM System" {
  usecase "UC-01\n直接 SDK 调用" as UC01
  usecase "UC-02\n通过 Gateway 调用 LLM" as UC02
  usecase "UC-03\n虚拟 Key 管理" as UC03
  usecase "UC-04\n负载均衡与故障转移" as UC04
  usecase "UC-05\n预算与限流管控" as UC05
  usecase "UC-06\n响应缓存" as UC06
  usecase "UC-07\n可观测性与日志" as UC07
  usecase "UC-08\n多模型路由" as UC08
}

Dev --> UC01
Dev --> UC02
Admin --> UC03
Admin --> UC05
Admin --> UC08
EU --> UC02
UC02 ..> UC04 : <<include>>
UC02 ..> UC05 : <<include>>
UC02 ..> UC06 : <<extend>>
UC02 ..> UC07 : <<include>>
UC01 --> LLM
UC02 --> LLM
@enduml
```

---

## UC-01：直接 SDK 调用

**参与者**：应用开发者  
**目标**：用统一接口调用任意 LLM，无需关心 Provider 差异

```plantuml
@startuml UC01
actor Developer
participant "litellm.completion()" as SDK
participant "utils.get_llm_provider()" as Resolve
participant "BaseLLMHTTPHandler" as Handler
participant "ProviderConfig\n.transform_request()" as Transform
participant "LLM Provider API" as API

Developer -> SDK : completion(model="gpt-4o", messages=[...])
SDK -> Resolve : 解析 model → provider="openai"
Resolve --> SDK : provider info
SDK -> Handler : BaseLLMHTTPHandler.completion()
Handler -> Transform : transform_request(OpenAI格式)
Transform --> Handler : provider 格式 payload
Handler -> API : HTTP POST /v1/chat/completions
API --> Handler : raw response
Handler -> Transform : transform_response()
Transform --> Handler : ModelResponse (OpenAI格式)
Handler --> SDK : ModelResponse
SDK --> Developer : ModelResponse
@enduml
```

**关联视图**：逻辑视图（SDK 核心类）、过程视图（SDK 请求序列）

---

## UC-02：通过 Gateway 调用 LLM

**参与者**：终端用户 / 应用开发者  
**目标**：通过 HTTP 调用 Gateway，享受认证、路由、计费等企业特性

```plantuml
@startuml UC02
actor Client
participant "POST /v1/chat/completions\n(proxy_server.py)" as Proxy
participant "user_api_key_auth()" as Auth
participant "Rate Limiter Hook" as Hook
participant "Router" as Router
participant "litellm SDK" as SDK
participant "LLM Provider" as LLM

Client -> Proxy : POST /v1/chat/completions\n{Authorization: Bearer sk-xxx}
Proxy -> Auth : 验证 API Key
Auth --> Proxy : UserAPIKeyAuth (key info, limits)
Proxy -> Hook : max_budget_limiter\nparallel_request_limiter
Hook --> Proxy : passed / rejected
Proxy -> Router : route_llm_request()
Router -> SDK : litellm.acompletion()
SDK -> LLM : HTTP 请求
LLM --> SDK : 响应
SDK --> Router : ModelResponse
Router --> Proxy : ModelResponse + cost
Proxy --> Client : HTTP 200 + x-litellm-response-cost header
@enduml
```

**关联视图**：逻辑视图（Gateway 组件）、过程视图（Gateway 请求流）、物理视图（部署结构）

---

## UC-03：虚拟 Key 管理

**参与者**：AI 工程师（Admin）  
**目标**：创建/吊销虚拟 Key，绑定预算、模型权限和团队

```plantuml
@startuml UC03
actor Admin
participant "POST /key/generate\n(proxy_server.py)" as API
participant "PrismaClient\n(PostgreSQL)" as DB
participant "DualCache\n(Redis + Memory)" as Cache

Admin -> API : POST /key/generate\n{team_id, budget, models}
API -> DB : INSERT LiteLLM_VerificationToken
DB --> API : token record
API -> Cache : 写入 key cache
Cache --> API : ok
API --> Admin : {key: "sk-xxx", budget: $10}
@enduml
```

---

## UC-04：负载均衡与故障转移

**参与者**：Router（自动）  
**目标**：跨多个 LLM 部署实例实现负载均衡，失败时自动 fallback

```plantuml
@startuml UC04
participant "Router" as R
participant "RoutingStrategy\n(lowest_latency/tpm_rpm)" as Strategy
participant "Deployment A\n(gpt-4o @ OpenAI)" as D_A
participant "Deployment B\n(gpt-4o @ Azure)" as D_B
participant "DualCache\n(TPM/RPM tracking)" as Cache

R -> Strategy : get_available_deployment()
Strategy -> Cache : 查询各 deployment 的 TPM/RPM/latency
Cache --> Strategy : metrics
Strategy --> R : 选择 Deployment A
R -> D_A : 调用
D_A --> R : 超时 / 失败
R -> R : 触发 fallback 逻辑
R -> D_B : 调用 fallback deployment
D_B --> R : 成功响应
R --> R : 更新 cooldown 状态
@enduml
```

---

## UC-05：预算与限流管控

**参与者**：MaxBudgetLimiter Hook  
**目标**：实时阻断超过预算或速率限制的请求

```plantuml
@startuml UC05
participant "Request" as Req
participant "MaxBudgetLimiter" as Budget
participant "ParallelRequestLimiter" as Rate
participant "InternalUsageCache\n(Redis)" as Cache
participant "PostgreSQL" as DB

Req -> Budget : pre_call_hook()
Budget -> Cache : GET key_spend
Cache --> Budget : current_spend = $8.5
Budget -> Budget : check: $8.5 < $10 (limit)
Budget --> Req : 允许继续

Req -> Rate : pre_call_hook()
Rate -> Cache : INCR + EXPIRE (RPM counter)
Cache --> Rate : rpm_count = 45
Rate -> Rate : check: 45 < 60 (limit)
Rate --> Req : 允许继续

note right of Req : 若超限，抛出 BudgetExceededError\n或 RateLimitError
@enduml
```

---

## UC-06：响应缓存

**参与者**：LLMCachingHandler  
**目标**：对相同请求返回缓存响应，减少 LLM 调用成本

```plantuml
@startuml UC06
participant "litellm.completion()" as SDK
participant "LLMCachingHandler" as Cache
participant "Cache Backend\n(Redis/Memory/S3)" as Backend
participant "LLM Provider" as LLM

SDK -> Cache : pre_call_hook(request)
Cache -> Backend : GET cache_key(model+messages+params)
alt 缓存命中
  Backend --> Cache : cached ModelResponse
  Cache --> SDK : 直接返回
else 缓存未命中
  Backend --> Cache : null
  Cache --> SDK : 继续调用
  SDK -> LLM : 实际 HTTP 请求
  LLM --> SDK : ModelResponse
  SDK -> Cache : post_call_hook(response)
  Cache -> Backend : SET cache_key = response (TTL)
end
@enduml
```

---

## UC-07：可观测性与日志

**参与者**：CustomLogger 集成（Langfuse、Datadog 等）  
**目标**：异步将调用详情、成本、tokens 推送到外部观测平台

```plantuml
@startuml UC07
participant "litellm SDK" as SDK
participant "Logging\n(litellm_logging.py)" as Log
participant "async_success_handler()" as Handler
participant "Langfuse\n/ Datadog\n/ SlackAlerting" as Ext
participant "DBSpendUpdateWriter" as Writer
participant "PostgreSQL" as DB

SDK -> Log : success_handler(response, metadata)
Log -> Log : _response_cost_calculator()
Log -> Handler : async_success_handler()
Handler -> Ext : log_success_event(payload)\n[异步, 不阻塞主链路]
Handler -> Writer : update_database(spend)
Writer -> DB : batch INSERT SpendLogs\n(每60秒批量写入)
@enduml
```
