# 逻辑视图 (Logical View)

> 描述系统的核心抽象和功能分解，聚焦于类/接口的结构与关系，与部署/进程无关。

---

## 1. 顶层组件分解

```plantuml
@startuml Logical-Components
skinparam componentStyle rectangle

package "LiteLLM System" {

  package "AI Gateway (proxy/)" {
    component [ProxyServer\nproxy_server.py] as PS
    component [AuthLayer\nauth/user_api_key_auth.py] as Auth
    component [ProxyHooks\nhooks/] as Hooks
    component [ManagementAPI\n/key /team /user] as MgmtAPI
    component [GuardrailsEngine\nguardrails/] as Guard
    component [DBSpendWriter\ndb/db_spend_update_writer.py] as DBWriter
  }

  package "SDK Core (litellm/)" {
    component [EntryPoints\nmain.py] as Entry
    component [Router\nrouter.py] as Router
    component [ProviderResolver\nutils.get_llm_provider()] as Resolver
    component [HTTPHandler\nllms/custom_httpx/llm_http_handler.py] as HTTP
    component [LLMCachingHandler\ncaching/llm_caching_handler.py] as Cache
    component [CostCalculator\ncost_calculator.py] as Cost
    component [LoggingCore\nlitellm_core_utils/litellm_logging.py] as LogCore
  }

  package "Provider Layer (llms/)" {
    component [BaseConfig\nbase_llm/chat/transformation.py] as Base
    component [OpenAIConfig\nllms/openai/chat/] as OAI
    component [AnthropicConfig\nllms/anthropic/chat/] as ANT
    component [GeminiConfig\nllms/gemini/chat/] as GEM
    component [BedrockConfig\nllms/bedrock/chat/] as BED
    component [AzureConfig\nllms/azure/] as AZ
    component ["100+ Providers\n..." ] as Others
  }

  package "Integrations (integrations/)" {
    component [CustomLogger\ncustom_logger.py] as CL
    component [LangfuseLogger] as Langfuse
    component [DatadogLogger] as DD
    component [SlackAlerting] as Slack
    component [ProxyCostTracker\nproxy_track_cost_callback.py] as PCT
  }

  package "Caching (caching/)" {
    component [DualCache\ndual_cache.py] as DC
    component [RedisCache\nredis_cache.py] as Redis
    component [InMemoryCache\nin_memory_cache.py] as Mem
    component [S3Cache / GCSCache] as CloudCache
  }
}

PS --> Auth
PS --> Hooks
PS --> Router
PS --> MgmtAPI
PS --> Guard

Router --> Entry
Router --> DC

Entry --> Resolver
Entry --> HTTP
Entry --> LogCore
Entry --> Cache

HTTP --> Base
Base <|-- OAI
Base <|-- ANT
Base <|-- GEM
Base <|-- BED
Base <|-- AZ
Base <|-- Others

LogCore --> CL
CL <|-- Langfuse
CL <|-- DD
CL <|-- Slack
CL <|-- PCT

PCT --> DBWriter

Cache --> DC
DC --> Redis
DC --> Mem
Redis <.. CloudCache : fallback
@enduml
```

---

## 2. SDK 核心类图

```plantuml
@startuml SDK-Classes
skinparam classAttributeIconSize 0

class "litellm.completion()\nlitellm.acompletion()" as Entry {
  +completion(model, messages, **kwargs): ModelResponse
  +acompletion(model, messages, **kwargs): Coroutine
  +embedding(model, input, **kwargs): EmbeddingResponse
}

class Router {
  -model_list: List[DeploymentConfig]
  -cache: DualCache
  -routing_strategy: RoutingStrategy
  -fallbacks: List[Dict]
  +acompletion(model, messages): ModelResponse
  +get_available_deployment(): Deployment
  +set_model_list(model_list): void
}

abstract class RoutingStrategy {
  +get_available_deployment(model, messages): Deployment
}

class LowestLatencyStrategy {
  +get_available_deployment(): Deployment
}

class LowestTPMRPMStrategy {
  +get_available_deployment(): Deployment
}

class SimpleShuffle {
  +get_available_deployment(): Deployment
}

abstract class BaseConfig {
  +transform_request(model, messages, optional_params): Dict
  +transform_response(model, raw_response, model_response): ModelResponse
  +get_supported_openai_params(): List[str]
  +map_openai_params(optional_params): Dict
}

class BaseLLMHTTPHandler {
  +completion(model, messages, optional_params, headers): ModelResponse
  +acompletion(): Coroutine
  +aembedding(): Coroutine
  -_make_common_call_or_get_cache(): ModelResponse
}

class HTTPHandler {
  -client: httpx.Client
  +post(url, headers, data): Response
}

class AsyncHTTPHandler {
  -client: httpx.AsyncClient
  +post(url, headers, data): Coroutine[Response]
}

class LLMCachingHandler {
  -cache: Cache
  +async_get_cache(request_kwargs): Optional[ModelResponse]
  +async_set_cache(result, request_kwargs): void
}

class DualCache {
  -in_memory: InMemoryCache
  -redis_cache: RedisCache
  +get_cache(key): Any
  +set_cache(key, value, ttl): void
  +async_get_cache(key): Coroutine
}

class "Logging\n(LiteLLMLoggingObj)" as Log {
  -model: str
  -messages: List
  -custom_logger_compatible_callbacks: List
  +pre_call(input, api_key): void
  +success_handler(result, start_time, end_time): void
  +async_success_handler(result): Coroutine
  +_response_cost_calculator(result): float
}

abstract class CustomLogger {
  +log_pre_api_call(model, messages, kwargs): void
  +log_success_event(kwargs, response, start_time, end_time): void
  +async_log_success_event(kwargs, response): Coroutine
  +async_log_failure_event(kwargs, exception): Coroutine
}

Entry --> Router : uses (when router configured)
Entry --> BaseLLMHTTPHandler : delegates HTTP
Entry --> Log : creates per request

Router --> RoutingStrategy
RoutingStrategy <|-- LowestLatencyStrategy
RoutingStrategy <|-- LowestTPMRPMStrategy
RoutingStrategy <|-- SimpleShuffle
Router --> DualCache

BaseLLMHTTPHandler --> BaseConfig : calls transform_*
BaseLLMHTTPHandler --> HTTPHandler
BaseLLMHTTPHandler --> AsyncHTTPHandler
BaseLLMHTTPHandler --> LLMCachingHandler

LLMCachingHandler --> DualCache

Log --> CustomLogger : notifies (async)
@enduml
```

---

## 3. Gateway (Proxy) 核心类图

```plantuml
@startuml Proxy-Classes
skinparam classAttributeIconSize 0

class ProxyServer {
  +chat_completion(request): Response
  +embedding(request): Response
  +health_check(): Response
  -initialize_litellm_settings(): void
  -startup_event(): void
}

class UserAPIKeyAuth {
  +user_api_key_auth(api_key, request): UserAPIKeyAuthInformation
  -get_api_key_from_cache(token): Optional[KeyInfo]
  -db_key_lookup(token): KeyInfo
}

class InternalUsageCache {
  -dual_cache: DualCache
  +get_cache(key): Any
  +set_cache(key, value, ttl): void
}

abstract class CustomHook {
  +async_pre_call_hook(user_api_key_dict, cache, call_type, data): void
  +async_post_call_success_hook(data, user_api_key_dict, response): void
}

class MaxBudgetLimiter {
  +async_pre_call_hook(user_api_key_dict, cache, data): void
  -check_budget(spend, max_budget): bool
}

class ParallelRequestLimiterV3 {
  +async_pre_call_hook(user_api_key_dict, cache, data): void
  -increment_rpm_counter(key, window): int
}

class DBSpendUpdateWriter {
  -redis_queue: RedisCache
  -prisma: PrismaClient
  +update_database(token, response_cost): void
  +flush_spend_to_db(): Coroutine
}

class PrismaClient {
  +get_data(token, table_name): Any
  +insert_data(data, table_name): void
  +update_data(token, data): void
}

class ProxyConfig {
  +load_config(config_path): void
  +add_deployment(db_models): void
  +get_model_list(): List[DeploymentConfig]
}

ProxyServer --> UserAPIKeyAuth : 每请求验证
ProxyServer --> CustomHook : pre/post call hooks
ProxyServer --> Router : 路由请求
ProxyServer --> ProxyConfig : 加载配置

CustomHook <|-- MaxBudgetLimiter
CustomHook <|-- ParallelRequestLimiterV3

UserAPIKeyAuth --> InternalUsageCache
InternalUsageCache --> DualCache

DBSpendUpdateWriter --> PrismaClient
DBSpendUpdateWriter --> RedisCache
@enduml
```

---

## 4. 数据模型（核心实体）

```plantuml
@startuml Data-Models

entity "LiteLLM_VerificationToken\n(Virtual API Key)" as Key {
  * token : String [PK]
  * key_name : String
  * budget_id : String [FK]
  * team_id : String [FK]
  * user_id : String [FK]
  * models : String[]
  spend : Float
  max_budget : Float
  tpm_limit : BigInt
  rpm_limit : BigInt
  expires : DateTime
}

entity "LiteLLM_TeamTable" as Team {
  * team_id : String [PK]
  * team_alias : String
  * budget_id : String [FK]
  * models : String[]
  spend : Float
  max_budget : Float
  tpm_limit : BigInt
  rpm_limit : BigInt
}

entity "LiteLLM_UserTable" as User {
  * user_id : String [PK]
  * user_email : String
  * budget_id : String [FK]
  * models : String[]
  spend : Float
  max_budget : Float
  user_role : String
}

entity "LiteLLM_BudgetTable" as Budget {
  * budget_id : String [PK]
  max_budget : Float
  soft_budget : Float
  tpm_limit : BigInt
  rpm_limit : BigInt
  budget_duration : String
  budget_reset_at : DateTime
}

entity "LiteLLM_SpendLogs" as SpendLog {
  * request_id : String [PK]
  * api_key : String
  * model : String
  * spend : Float
  prompt_tokens : Int
  completion_tokens : Int
  startTime : DateTime
  endTime : DateTime
  user : String
  team_id : String
}

entity "LiteLLM_OrganizationTable" as Org {
  * organization_id : String [PK]
  * organization_alias : String
  * budget_id : String [FK]
  models : String[]
  spend : Float
}

entity "LiteLLM_ProxyModelTable" as Model {
  * model_id : String [PK]
  * model_name : String
  litellm_params : Json
  model_info : Json
}

Key }o--|| Budget : budget_id
Key }o--|| Team : team_id
Key }o--|| User : user_id
Team }o--|| Budget : budget_id
Team }o--|| Org : organization_id
User }o--|| Budget : budget_id
User }o--|| Org : organization_id
SpendLog }o--|| Key : api_key
@enduml
```

---

## 5. Provider 扩展架构

```plantuml
@startuml Provider-Extension

abstract class BaseConfig {
  +transform_request(model, messages, optional_params, litellm_params, headers): Dict
  +transform_response(model, raw_response, model_response, logging_obj, ...): ModelResponse
  +get_supported_openai_params(model): List[str]
  +map_openai_params(non_default_params, optional_params, model, drop_params): Dict
  +validate_environment(headers, model, messages, optional_params, api_key): Dict
}

class OpenAIGPTConfig {
  +transform_request(): Dict
  +transform_response(): ModelResponse
  +get_supported_openai_params(): List[str]
}

class AnthropicConfig {
  +transform_request(): Dict
  +transform_response(): ModelResponse
  +_transform_messages_with_cache_control(): List
}

class GeminiConfig {
  +transform_request(): Dict
  +transform_response(): ModelResponse
  +_format_parts_for_gemini(): List
}

class BedrockConverseConfig {
  +transform_request(): Dict
  +transform_response(): ModelResponse
}

class CustomProviderConfig {
  +transform_request(): Dict
  +transform_response(): ModelResponse
}

BaseConfig <|-- OpenAIGPTConfig
BaseConfig <|-- AnthropicConfig
BaseConfig <|-- GeminiConfig
BaseConfig <|-- BedrockConverseConfig
BaseConfig <|-- CustomProviderConfig

note right of CustomProviderConfig
  用户可实现 BaseConfig
  注册为自定义 Provider
end note

class BaseLLMHTTPHandler {
  +completion(model, config: BaseConfig, ...): ModelResponse
}

BaseLLMHTTPHandler --> BaseConfig : config.transform_request()\nconfig.transform_response()
@enduml
```
