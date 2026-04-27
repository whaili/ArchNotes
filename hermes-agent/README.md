# Hermes Agent 架构分析 — UML 4+1 视图

> 基于 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) 源码分析（2026-04-27）

## 目录

| 视图 | 文件 | 说明 |
|------|------|------|
| 场景视图 (Scenarios +1) | [01-scenarios.md](01-scenarios.md) | 核心用例，驱动其他四个视图 |
| 逻辑视图 (Logical) | [02-logical.md](02-logical.md) | 核心类/组件结构与关系 |
| 过程视图 (Process) | [03-process.md](03-process.md) | 运行时交互、Agent 主循环、异步流程 |
| 开发视图 (Development) | [04-development.md](04-development.md) | 模块/包组织、代码层次 |
| 物理视图 (Physical) | [05-physical.md](05-physical.md) | 部署拓扑与终端后端 |

## 系统简介

Hermes Agent 是 Nous Research 开源的**自学习型 AI 代理框架**，核心特征：

- **闭环学习**：Agent 在使用过程中自动生成、改进 Markdown 技能（兼容 [agentskills.io](https://agentskills.io) 标准）
- **多层记忆**：MEMORY.md/USER.md（短期）+ FTS5 全文检索（会话级）+ Honcho dialectic 用户建模（跨会话）+ Context Engine DAG 压缩（上下文级）
- **Provider 无关**：通过 `anthropic_adapter` / `bedrock_adapter` / `gemini_native_adapter` 等 200+ 模型一键切换，无需改代码
- **多终端后端**：Local / Docker / SSH / Daytona / Singularity / Modal 六种执行环境
- **多平台网关**：CLI / Telegram / Discord / Slack / WhatsApp / Signal 统一接入
- **零上下文成本子 Agent**：通过 Python RPC 委派子任务，不消耗主对话上下文
- **研究就绪**：内建批量轨迹生成（`batch_runner.py`），可作为 RL/SFT 训练数据采集器

## 架构核心原则

```
┌────────────────────────────────────────────────────────────────────┐
│ Gateway 层（gateway/）：CLI / IM / Web 多渠道接入                   │
├────────────────────────────────────────────────────────────────────┤
│ Agent 主循环（run_agent.py / agent/）：tool-calling loop + 中断    │
├────────────────────────────────────────────────────────────────────┤
│ 工具层（tools/）：40+ 内建工具 + MCP 外部工具 + 技能调用            │
├────────────────────────────────────────────────────────────────────┤
│ 执行层（environments/ + tools/terminal_tool.py）：六种 backend     │
├────────────────────────────────────────────────────────────────────┤
│ 模型层（agent/*_adapter.py）：Provider 解耦、统一适配                │
└────────────────────────────────────────────────────────────────────┘
        ▲                          ▲                       ▲
        │                          │                       │
   memory_manager           context_engine          credential_pool
   (MEMORY/USER.md)         (DAG 压缩)              (Key 池 + OAuth)
```

- **解耦**：Gateway 只负责消息透传，Agent 主循环与执行后端正交
- **可扩展**：技能、Memory Provider、终端 backend、模型 Adapter 均为插件化
- **可观测**：Trajectory 模块记录每一步推理与工具调用，可用于训练复盘

## 阅读建议

1. 先看 [01-scenarios.md](01-scenarios.md) 了解 Hermes 解决的核心问题
2. 再看 [02-logical.md](02-logical.md) 把握组件抽象
3. 想理解运行机制看 [03-process.md](03-process.md)
4. 想做二次开发看 [04-development.md](04-development.md)
5. 想部署生产看 [05-physical.md](05-physical.md)
