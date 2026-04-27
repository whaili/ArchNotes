# 开发视图 (Development View)

> 描述代码的模块组织、包依赖关系和开发层次，帮助开发者理解代码结构、找到修改点。

---

## 1. 顶层目录结构

```mermaid
flowchart TB
    subgraph Repo["BerriAI/litellm (repo root)"]
        subgraph PyPkg["litellm/  Python Package"]
            PROXY["proxy/\nAI Gateway"]
            LLMS["llms/\nProvider Implementations"]
            CACHING["caching/\nCache Backends"]
            INTEGRATIONS["integrations/\nObservability Callbacks"]
            CORE["litellm_core_utils/\nCore Utilities"]
            STRATEGY["router_strategy/\nRouting Algorithms"]
            MAIN["main.py\nEntry Points"]
            ROUTER["router.py\nLoad Balancer"]
            UTILS["utils.py\nProvider Resolver"]
            COST["cost_calculator.py"]
        end

        UI["ui/\nReact Admin Dashboard"]
        TESTS["tests/\nTest Suite"]
        DEPLOY["deploy/\nK8s / Helm Charts"]
        DOCKER["docker/\nDockerfiles"]
        SCHEMA["schema.prisma\nDB Schema"]
        DC["docker-compose.yml"]
        PYPROJECT["pyproject.toml"]
    end
```

---

## 2. 模块层次与依赖关系

```mermaid
flowchart LR
    subgraph API["API Layer"]
        PS["proxy_server.py\nFastAPI app"]
        AUTH["auth/\nuser_api_key_auth.py"]
        MBL["hooks/\nmax_budget_limiter.py"]
        PRL["hooks/\nparallel_request_limiter_v3.py"]
        PTCC["hooks/\nproxy_track_cost_callback.py"]
        DBW["db/\ndb_spend_update_writer.py"]
        GRD["guardrails/"]
    end

    subgraph SDKCore["SDK Core"]
        MAIN["main.py"]
        ROUTERR["router.py"]
        UTILS["utils.py"]
        COST["cost_calculator.py"]
        LOG["litellm_core_utils/\nlitellm_logging.py"]
        STREAM["litellm_core_utils/\nstreaming_handler.py"]
        HTTPH["llms/custom_httpx/\nllm_http_handler.py"]
        HTTP["llms/custom_httpx/\nhttp_handler.py"]
    end

    subgraph ProviderAdapters["Provider Adapters"]
        BASE["base_llm/\nchat/transformation.py"]
        OAI["llms/openai/"]
        ANT["llms/anthropic/"]
        GEM["llms/gemini/"]
        BED["llms/bedrock/"]
        AZ["llms/azure/"]
        MORE["llms/cohere/\ndeepseek/ vertex_ai/\n90+ more"]
    end

    subgraph CrossCutting["Cross-cutting"]
        DC["caching/\ndual_cache.py"]
        REDIS["caching/\nredis_cache.py"]
        MEM["caching/\nin_memory_cache.py"]
        CL["integrations/\ncustom_logger.py"]
        LF["integrations/langfuse/"]
        DD["integrations/datadog/"]
        SA["integrations/SlackAlerting/"]
    end

    PS --> AUTH & MBL & PRL & ROUTERR & DBW
    ROUTERR --> MAIN & DC
    MAIN --> UTILS & HTTPH & LOG
    HTTPH --> BASE & HTTP
    OAI & ANT & GEM & BED & AZ & MORE -->|extends| BASE
    LOG --> CL
    LF & DD & SA & PTCC -->|extends| CL
    PTCC --> DBW
    DC --> REDIS & MEM
    MBL & PRL & AUTH --> DC
    DBW --> REDIS
```

---

## 3. `litellm/proxy/` 内部模块图

```mermaid
flowchart TD
    PS["proxy_server.py\nFastAPI 主应用，定义所有 API 路由"]

    subgraph AuthDir["auth/"]
        UAKA["user_api_key_auth.py"]
        JWT["handle_jwt.py"]
        OAUTH["oauth2_check.py"]
        RC["route_checks.py"]
        AC["auth_checks.py"]
    end

    subgraph HooksDir["hooks/"]
        MBL2["max_budget_limiter.py"]
        PRL2["parallel_request_limiter_v3.py"]
        PTCC2["proxy_track_cost_callback.py"]
        CCC["cache_control_check.py"]
    end

    subgraph DBDir["db/"]
        DBW2["db_spend_update_writer.py"]
        CV["create_views.py"]
    end

    subgraph GuardDir["guardrails/"]
        IG["init_guardrails.py"]
        GI["guardrail_initializers.py"]
        GH["guardrail_hooks/"]
    end

    subgraph MgmtDir["management_helpers/"]
        BRJ["budget_reset_job.py"]
        KRM["key_rotation_manager.py"]
        SLC["spend_log_cleanup.py"]
    end

    subgraph Endpoints["Endpoint Packages"]
        AEP["anthropic_endpoints/"]
        VAP["vertex_ai_endpoints/"]
        IEP["image_endpoints/"]
        BEP["batches_endpoints/"]
        PEP["pass_through_endpoints/"]
        RAP["response_api_endpoints/"]
        AGEP["agent_endpoints/"]
    end

    CRP["common_request_processing.py"]
    HC["health_check.py"]
    CM["config_management_endpoints/"]

    PS --> UAKA & MBL2 & PRL2 & CRP & HC & CM
    UAKA --> JWT & OAUTH & RC & AC
    MBL2 & PRL2 & PTCC2 --> DBW2
    IG --> GI --> GH
```

---

## 4. `litellm/llms/` Provider 组织结构

```mermaid
flowchart TB
    subgraph Abstract["base_llm/  抽象基类"]
        BASE["base_llm/chat/transformation.py\nBaseConfig ABC"]
        EMBASE["base_llm/embedding/transformation.py\nBaseEmbeddingConfig"]
    end

    subgraph Orchestrator["custom_httpx/  HTTP 编排层"]
        HTTPH2["llm_http_handler.py\nBaseLLMHTTPHandler"]
        HTTP2["http_handler.py\nHTTPHandler + AsyncHTTPHandler"]
    end

    subgraph OpenAI["openai/"]
        OAI2["chat/gpt_transformation.py\nOpenAIGPTConfig"]
    end

    subgraph Anthropic["anthropic/"]
        ANT2["chat/transformation.py\nAnthropicConfig"]
        ANTPT["experimental_pass_through/"]
    end

    subgraph Gemini["gemini/"]
        GEM2["chat/transformation.py\nGeminiConfig"]
    end

    subgraph Bedrock["bedrock/"]
        BEDCONV["chat/converse_transformation.py\nBedrockConverseConfig"]
        BEDINV["chat/invoke_transformations/\nanthropic_claude3_transformation.py"]
    end

    subgraph VertexAI["vertex_ai/"]
        VERTGEM["gemini/transformation.py"]
        VERTANT["vertex_ai_partner_models/anthropic/"]
    end

    subgraph Azure["azure/"]
        AZ2["chat/transformation.py\nAzureOpenAIConfig"]
    end

    MORE2["cohere/ deepseek/ mistral/\nand 90+ more providers"]

    BASE --> HTTPH2
    HTTPH2 --> HTTP2

    OAI2 & ANT2 & GEM2 & BEDCONV & BEDINV & VERTGEM & AZ2 & MORE2 -->|extends| BASE
```

---

## 5. 测试模块结构

```mermaid
flowchart LR
    subgraph Tests["tests/"]
        subgraph Translation["llm_translation/  单元测试"]
            T1["test_openai_translation.py"]
            T2["test_anthropic_translation.py"]
            T3["test_bedrock_translation.py"]
            T4["test_gemini_translation.py"]
            T5["test_vertex_translation.py"]
        end

        subgraph ProxyUnit["proxy_unit_tests/  Gateway 单元测试"]
            TA["test_auth.py"]
            TBL["test_budget_limiter.py"]
            TPRL["test_parallel_request_limiter.py"]
            TR["test_router.py"]
        end

        subgraph LoadTests["load_tests/"]
            TL["test_load.py"]
        end

        subgraph Integration["test_amazing_proxy_custom_logger/"]
            TCL["test_custom_logger.py"]
        end
    end

    Note1["Translation 测试\n无需真实 API Key\n直接实例化 ProviderConfig\n测试 transform_*()"]
    Note2["Gateway 测试\n使用 FastAPI TestClient\nMock Redis + Prisma"]

    Translation --- Note1
    ProxyUnit --- Note2
```

---

## 6. 关键文件快速索引

| 场景 | 文件路径 |
|------|----------|
| 新增 LLM Provider | `litellm/llms/{provider}/chat/transformation.py` |
| 新增 Proxy Hook | `litellm/proxy/hooks/` + 注册到 `PROXY_HOOKS` |
| 修改请求路由逻辑 | `litellm/router.py` |
| 新增路由策略 | `litellm/router_strategy/{strategy}.py` |
| 修改 API Key 认证 | `litellm/proxy/auth/user_api_key_auth.py` |
| 修改成本计算 | `litellm/cost_calculator.py` |
| 修改 DB Schema | `schema.prisma` |
| 新增 API 端点 | `litellm/proxy/proxy_server.py` |
| 新增可观测集成 | `litellm/integrations/{service}/`（继承 `CustomLogger`） |
| 修改缓存逻辑 | `litellm/caching/caching_handler.py` |
| 流式响应处理 | `litellm/litellm_core_utils/streaming_handler.py` |
