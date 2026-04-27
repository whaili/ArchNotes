# LiteLLM 架构分析 — UML 4+1 视图

> 基于 [BerriAI/litellm](https://github.com/BerriAI/litellm) 源码分析（2026-04-27）

## 目录

| 视图 | 文件 | 说明 |
|------|------|------|
| 场景视图 (Scenarios +1) | [01-scenarios.md](01-scenarios.md) | 核心用例，驱动其他四个视图 |
| 逻辑视图 (Logical) | [02-logical.md](02-logical.md) | 核心类/组件结构与关系 |
| 过程视图 (Process) | [03-process.md](03-process.md) | 运行时交互、异步流程 |
| 开发视图 (Development) | [04-development.md](04-development.md) | 模块/包组织、代码层次 |
| 物理视图 (Physical) | [05-physical.md](05-physical.md) | 部署拓扑与基础设施 |
| 应急缓解 (扩展) | [06-k8s-mitigation.md](06-k8s-mitigation.md) | AWS/K8s 部署下的容量评估、检测与自愈 |

## 系统简介

LiteLLM 是一个开源 AI Gateway，提供：

- **统一 API**：以 OpenAI 格式调用 100+ LLM 提供商（OpenAI、Anthropic、Gemini、Bedrock、Azure 等）
- **Python SDK**：直接嵌入应用的轻量库
- **AI Gateway（Proxy）**：团队级中心化服务，带认证、限流、预算管控、负载均衡和管理面板
- **8ms P95 延迟**，支持 1k RPS

## 架构核心原则

```
Client SDK ──▶ LiteLLM AI Gateway (proxy/) ──▶ LiteLLM SDK (litellm/) ──▶ LLM Provider API
```

- **解耦**：Gateway 负责认证/路由/计费，SDK 负责协议转换/HTTP 调用
- **扩展性**：每个 Provider 独立封装在 `llms/{provider}/` 目录，新增只需实现 `BaseConfig`
- **可观测**：所有日志、指标通过 `CustomLogger` 回调异步处理
