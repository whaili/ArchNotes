# Hermes Agent vs OpenClaw：架构与设计异同

> 调研日期：2026-04-22  
> 参考版本：hermes-agent main / openclaw main  
> OpenClaw 详细笔记见 [openclaw-arch-notes.md](./openclaw-arch-notes.md)

---

## 目录

1. [概览对比](#1-概览对比)
2. [相同点](#2-相同点)
3. [核心差异](#3-核心差异)
   - [语言与技术栈](#31-语言与技术栈)
   - [模型集成方式](#32-模型集成方式)
   - [工具调用与分发机制](#33-工具调用与分发机制)
   - [执行环境与沙箱](#34-执行环境与沙箱)
   - [内存与上下文管理](#35-内存与上下文管理)
   - [技能/知识持久化](#36-技能知识持久化)
   - [多用户与多平台支持](#37-多用户与多平台支持)
   - [子 Agent 与并行化](#38-子-agent-与并行化)
   - [资源控制与计费](#39-资源控制与计费)
4. [设计哲学总结](#4-设计哲学总结)

---

## 1. 概览对比

| 维度 | OpenClaw | Hermes Agent |
|------|----------|--------------|
| 主语言 | TypeScript / Node.js | Python（87%）+ TypeScript（web 层） |
| 定位 | 工具执行代理，强调安全隔离 | 自学习代理，强调跨平台持久记忆 |
| 模型绑定 | 偏 Anthropic / MCP 生态 | 完全 Provider 无关（OpenAI / Anthropic / OpenRouter / 本地） |
| 执行后端 | Docker / SSH / 插件化自定义 | Local / Docker / SSH / Modal / Daytona / Singularity |
| 多平台 | 单渠道（CLI + 自研 gateway） | 多平台网关（Telegram / Discord / Slack / WhatsApp / Signal / CLI） |
| 多用户 | 进程级外部隔离（非原生） | 内建 DM pairing / 平台级用户认证 |
| 内存 | LanceDB 向量存储（可选） | Markdown 文件 + FTS5 全文检索 + Honcho 用户建模 |
| 技能系统 | 无 | 自主生成 Markdown 技能（兼容 agentskills.io） |
| 上下文压缩 | 无原生机制 | DAG 压缩引擎，可配置 token 预算 |
| 资源限制 | per-agent Docker cgroup 限制 | 无同等粒度的资源控制文档 |
| 计费追踪 | 内建 token 追踪 + 阶梯定价（1300+ 行） | 未见同等内建计费基础设施 |
| License | — | MIT |

---

## 2. 相同点

### 2.1 执行后端多样性

两者都支持 **Docker 容器** 和 **SSH 远程** 两种核心执行方式，且都将 backend 设计为可扩展：

- OpenClaw：`registerSandboxBackend()` 插件接口，三种内建（docker / ssh / 自定义）
- Hermes：`terminal_backend` 配置项，六种内建后端

### 2.2 MCP 协议支持

两者都集成了 **MCP（Model Context Protocol）**，但角色不同（见 [§3.3](#33-工具调用与分发机制)）。

### 2.3 子 Agent 委托

两者都支持将子任务委托给独立 Agent 执行：

- OpenClaw：通过 spawn 子 agent（配置不同资源 profile）
- Hermes：spawn isolated subagents + Python RPC 接口实现零上下文成本调用

### 2.4 Sandbox 懒启动思路

两者的容器都不是服务器启动时立即创建，而是按需（首次任务触发）创建。

### 2.5 开放性与可扩展性

两者都通过插件/provider 机制支持扩展——OpenClaw 是 backend 层的插件，Hermes 是 memory provider / toolset 层的扩展。

---

## 3. 核心差异

### 3.1 语言与技术栈

| | OpenClaw | Hermes |
|--|----------|--------|
| 核心语言 | TypeScript | Python |
| 运行时 | Node.js | CPython |
| 向量库 | LanceDB + sqlite-vec（native addon） | 无原生向量库（FTS5 全文检索） |
| Web 层 | — | TypeScript |

语言选择直接影响生态：OpenClaw 与 npm 工具链天然契合，Hermes 与 Python AI/ML 生态（Modal、Daytona、Singularity 等计算基础设施）天然契合。

---

### 3.2 模型集成方式

**OpenClaw** 以 MCP 作为工具调用的核心协议，MCP 是 Anthropic 定义的标准，项目在工具层深度依赖该协议，与 Anthropic 生态耦合较深。

**Hermes** 设计为完全 Provider 无关：

```
hermes model <provider>/<model>  # 无需改代码切换模型
```

内建适配器：`anthropic_adapter.py`、`gemini_native_adapter.py`、`bedrock_adapter.py`……  
对 Anthropic 无特殊依赖，MCP 仅作为可选的外部工具扩展机制。

---

### 3.3 工具调用与分发机制

这是两者**架构上最核心的差异**。

**OpenClaw：MCP 作为核心内部总线**

```
LLM 返回 tool_call
        ↓
  agent runner（推理主循环）
        ↓  HTTP POST 127.0.0.1:<随机端口>（JSON-RPC 2.0）
  MCP loopback server（主进程内）
        ↓  工具路由 + 权限检查 + 策略执行
  tool.execute()
        ↓
  exec → docker exec / ssh / spawn
```

MCP loopback 是所有工具调用必经的中间层，绑定 `127.0.0.1`（Docker 容器内无法反向访问，这也是工具层无法搬入容器的根本原因）。

**Hermes：Python 原生调用 + MCP 作为外部扩展**

```
LLM 返回 tool_call
        ↓
  Python agent loop（直接调用）
        ↓
  tool_handler（Python 函数）
        ↓  或通过 Python RPC（零上下文成本）
  execution backend（local/docker/ssh/...）
```

MCP server 作为**可选插件**接入，扩展内建工具集以外的能力，不是核心分发总线。

**含义对比：**

| 方面 | OpenClaw（MCP 总线） | Hermes（Python 原生） |
|------|---------------------|----------------------|
| 工具隔离 | HTTP 边界天然隔离 | 同进程，隔离靠代码约束 |
| 跨语言工具 | 只要实现 JSON-RPC 即可 | 主要是 Python |
| 性能开销 | loopback HTTP 有微小延迟 | 函数直调，延迟更低 |
| 调试可观测性 | HTTP 请求可独立抓包 | 需要 Python 级追踪 |

---

### 3.4 执行环境与沙箱

**两者都支持 Docker + SSH**，但关注点不同：

| 方面 | OpenClaw | Hermes |
|------|----------|--------|
| 资源限制粒度 | per-agent `memory`/`cpus`/`pidsLimit` Docker cgroup | 未见同等配置文档 |
| 容器生命周期管理 | 明确 `idleHours`/`maxAgeDays` prune 机制 | 未见明确 prune 策略 |
| Sandbox scope | session / agent / shared 三级 | 无对应文档 |
| 安全特性 | cap_drop / read-only root / seccomp | 无专项安全配置文档 |
| 无服务器后端 | ❌（需自定义插件） | ✅ Modal（内建） |
| HPC 容器 | ❌ | ✅ Singularity（内建） |

OpenClaw 在**安全隔离精细度**上更成熟（cgroup 限制、capability dropping、只读根文件系统等均有明确配置）；  
Hermes 在**执行基础设施多样性**上更宽广（Modal serverless、Daytona 云沙箱、Singularity HPC）。

---

### 3.5 内存与上下文管理

这是两者差距最大的模块之一。

**OpenClaw 内存**：

- 向量记忆：LanceDB（native addon，可选）
- 无明确的跨 session 上下文压缩机制
- session 级别 token 追踪（用于计费，非上下文管理）
- 内存开销：LanceDB + sqlite-vec 在主进程内，开启后有显著内存压力

**Hermes 内存（多层架构）**：

```
┌─────────────────────────────────────────────────┐
│ 用户建模层：Honcho dialectic（跨 session 用户画像）│
├─────────────────────────────────────────────────┤
│ 会话记忆层：FTS5 全文检索 + LLM 摘要（跨 session）│
├─────────────────────────────────────────────────┤
│ 当前上下文层：Context Engine（DAG 压缩）           │
├─────────────────────────────────────────────────┤
│ 短期记忆：MEMORY.md / USER.md（agent 自主维护）    │
└─────────────────────────────────────────────────┘
```

**上下文压缩**是 Hermes 的独特能力：

- `context_engine.py` 管理全局 token 预算（`threshold_tokens`）
- 超出预算时触发 DAG 构建 + 摘要压缩
- 保护头尾若干消息（`protect_first_n` / `protect_last_n`）
- 支持 pre-flight 估算（API 调用前预判是否压缩）

**MemoryManager 设计**（Hermes）：

- 一个内建 provider（不可移除）+ 最多一个外部 plugin provider
- 防止 tool schema 膨胀和冲突
- 生命周期钩子：`on_turn_start` / `on_session_end` / `on_pre_compress` / `on_delegation`

---

### 3.6 技能/知识持久化

**OpenClaw**：无原生技能系统。过往经验不会自动提炼为可复用知识，每次对话从零开始（除向量记忆外）。

**Hermes**：**自主技能生成**是核心差异化特性：

```
完成复杂任务
      ↓
agent 自动生成 SKILL.md（含 YAML frontmatter + 指令 markdown）
      ↓
存储于 ~/.hermes/skills/<category>/SKILL.md
      ↓
用户后续通过 /skill-name 调用
      ↓
agent 使用时进一步改进技能内容（closed learning loop）
```

技能文件支持模板变量（`${HERMES_SKILL_DIR}`）、内联 shell 展开（ `` !`date` ``）、config.yaml 注入，兼容 agentskills.io 开放标准——技能可跨 agent 实例复用。

---

### 3.7 多用户与多平台支持

**OpenClaw**（见 [openclaw-arch-notes.md §1](./openclaw-arch-notes.md)）：

- **单用户/单 Operator 架构**，无原生用户隔离
- 推荐方案：进程级隔离 + 外部管理层
- 有限的 `session.scope: "per-sender"` 发送者分组，非真正用户隔离
- 单渠道网关，无跨平台消息整合

**Hermes**：

- **多平台网关**是一等公民：Telegram / Discord / Slack / WhatsApp / Signal / CLI 统一接入
- DM pairing 机制做平台级用户认证
- 跨平台会话连续性（同一用户在不同平台切换，记忆连续）
- 内建调度器（cron）支持自动化任务投递到任意平台

**含义**：OpenClaw 面向单租户工具代理场景；Hermes 面向面向多用户的个人助理/Bot 部署场景。

---

### 3.8 子 Agent 与并行化

| | OpenClaw | Hermes |
|--|----------|--------|
| 子 agent spawn | ✅ 支持（不同资源 profile） | ✅ 支持（isolated subagents） |
| 并行化机制 | 多子 agent 并发 | parallel workstreams |
| 零上下文成本调用 | ❌ | ✅ Python RPC（工具调用不消耗主 agent 上下文） |
| 资源隔离 | Docker cgroup per-agent | 无同等文档 |

Hermes 的 **Python RPC** 是独特设计：将工具调用从主 agent 对话上下文中剥离，直接在独立进程/线程中执行，返回结果后才写入主上下文——对需要频繁调用工具但不希望撑爆上下文窗口的场景（如批量文件处理）很有价值。

---

### 3.9 资源控制与计费

**OpenClaw**（详见 [openclaw-arch-notes.md §3](./openclaw-arch-notes.md)）：

- Docker 层：per-agent `memory` / `cpus` / `pidsLimit` cgroup 限制
- 计费：`session-cost-usage.ts`（1300+ 行），按模型/Provider/天维度聚合，支持阶梯定价
- 缺口：无原生限额执行、无用户维度、无计费 webhook

**Hermes**：

- 未见同等精细度的 Docker 资源控制文档
- 未见内建计费/token 追踪基础设施
- 多平台场景下，API 成本追踪是明显缺口

---

## 4. 设计哲学总结

```
OpenClaw：安全工具执行引擎
──────────────────────────────────────────────────────
强调 "代码以可控方式执行"
  → 精细的沙箱资源控制（cgroup + capability dropping）
  → MCP 作为审计/权限检查的中间层总线
  → 懒启动 + 自动 prune 的容器生命周期管理
  → 完整的 token 计费追踪
  → 多用户靠外部进程隔离，保持 core 简洁

适合场景：受控执行环境、需要可审计工具调用的企业代理、
        运维自动化、代码执行沙箱


Hermes Agent：自学习个人助理
──────────────────────────────────────────────────────
强调 "agent 持续成长，全渠道陪伴用户"
  → 自主技能生成 + closed learning loop
  → 多层记忆（向量/全文/用户画像）+ 上下文压缩
  → 多平台网关（从 CLI 到即时通讯统一体验）
  → Provider 无关，轻松切换模型
  → Python RPC 降低工具调用的上下文成本

适合场景：个人 AI 助理部署、跨平台 Bot、
        需要长期记忆和自我优化的 agent、研究轨迹收集
```

### 互补性

两者并非竞争关系，而是侧重点互补：

- **OpenClaw 的沙箱安全模型** 可作为 Hermes 的 Docker 后端增强（Hermes 的 Docker backend 缺乏同等安全配置）
- **Hermes 的技能系统和上下文压缩** 是 OpenClaw 的空白，可作为上层能力引入
- **OpenClaw 的 MCP loopback 审计链路** 为工具调用提供了可观测性，Hermes 若需企业级合规场景可借鉴
- **Hermes 的 Provider 无关适配器** 是 OpenClaw 值得学习的解耦模式
