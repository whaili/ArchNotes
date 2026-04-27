# 场景视图 (Scenarios / Use Cases — "+1")

> 场景视图是 4+1 架构中的驱动力。以下用例覆盖 Hermes Agent 核心功能，并与其余四个视图形成追踪关系。

---

## 核心用例总览

```mermaid
flowchart LR
    Dev(["个人用户\nCLI User"])
    IM(["IM 用户\nTelegram/Discord/Slack/...\nWhatsApp/Signal"])
    Researcher(["研究人员\nTrajectory Collector"])
    Schedule(["调度器\nCron"])

    subgraph System["Hermes Agent System"]
        UC01["UC-01\nCLI 对话"]
        UC02["UC-02\n多平台 IM 接入"]
        UC03["UC-03\n工具调用 + 终端执行"]
        UC04["UC-04\n技能自主生成与调用"]
        UC05["UC-05\n跨会话记忆与回忆"]
        UC06["UC-06\n上下文 DAG 压缩"]
        UC07["UC-07\n子 Agent 委派"]
        UC08["UC-08\n模型切换"]
        UC09["UC-09\n定时任务调度"]
        UC10["UC-10\n批量轨迹生成"]
    end

    LLM(["☁️ LLM Provider\nOpenAI / Anthropic / OpenRouter\nNous Portal / NVIDIA NIM ..."])
    Backend(["执行后端\nLocal/Docker/SSH/Modal/\nDaytona/Singularity"])

    Dev --> UC01
    IM --> UC02
    Researcher --> UC10
    Schedule --> UC09

    UC01 --> UC03
    UC02 --> UC03
    UC03 --> UC04
    UC03 --> UC07
    UC01 --> UC05
    UC02 --> UC05
    UC05 -.->|trigger| UC06
    UC01 --> UC08
    UC09 --> UC03

    UC03 --> Backend
    UC01 --> LLM
    UC02 --> LLM
    UC07 --> LLM
```

---

## UC-01：CLI 对话

**参与者**：个人用户 | **目标**：在终端启动 Hermes，进行多轮工具调用对话

```mermaid
sequenceDiagram
    actor User
    participant CLI as hermes_cli.main
    participant Cfg as Config_and_Setup
    participant Run as run_agent.AIAgent
    participant Loop as run_conversation
    participant Adp as ProviderAdapter
    participant LLM as LLM_API

    User->>CLI: hermes
    CLI->>Cfg: load env, model_catalog, tools_config
    CLI->>Run: AIAgent(model, tools, memory, ...)
    Run->>Run: 加载 MEMORY.md / USER.md / Skills
    User->>Loop: 输入 prompt（curses_ui）
    Loop->>Adp: build_messages + tools schema
    Adp->>LLM: chat.completions / messages.create
    LLM-->>Adp: assistant + tool_calls
    Adp-->>Loop: parsed response
    Loop->>Loop: 执行 tool_calls（见 UC-03）
    Loop->>Adp: 追加 tool_result，再次调用
    LLM-->>Adp: 终态回复
    Adp-->>Loop: final assistant text
    Loop-->>User: 流式 token（stream_delta_callback）
```

---

## UC-02：多平台 IM 接入

**参与者**：IM 用户 | **目标**：通过 Telegram/Discord/Slack/WhatsApp/Signal 与 Hermes 对话，会话记忆跨平台延续

```mermaid
sequenceDiagram
    actor User
    participant Plat as Platform (Telegram/Discord/...)
    participant GW as gateway/run.py
    participant SC as stream_consumer
    participant Sess as session.SessionContext
    participant Pair as pairing.py
    participant Agent as run_agent.AIAgent
    participant Del as delivery.py

    User->>Plat: 发送消息
    Plat->>GW: webhook / long-poll
    GW->>SC: ingest event
    SC->>Pair: 解析 sender，查找绑定 user
    Pair-->>SC: hermes_user_id
    SC->>Sess: load_or_create_session(user, channel)
    Sess->>Agent: dispatch(prompt, context, memory)
    Agent->>Agent: tool-calling loop
    Agent-->>Sess: response
    Sess->>Del: send(channel, payload)
    Del->>Plat: 平台 API（含 sticker_cache）
    Plat-->>User: 渲染消息
```

> **关键设计**：`pairing.py` 把不同平台的 sender → 同一个 `hermes_user_id`，所以同一用户的 MEMORY/USER/技能在 Telegram 和 Discord 之间是连续的。

---

## UC-03：工具调用 + 终端执行

**参与者**：Agent 主循环（自动触发） | **目标**：执行 LLM 返回的 `tool_call`，结果回填上下文

```mermaid
sequenceDiagram
    participant Loop as run_conversation
    participant Reg as tools.registry
    participant Tool as ToolHandler
    participant Sec as Guards
    participant Term as terminal_tool
    participant Env as TerminalBackend
    participant Store as tool_result_storage

    Loop->>Reg: dispatch(tool_name, args)
    Reg->>Sec: 校验路径 / URL / 权限
    Sec-->>Reg: pass / approval_required
    Reg->>Tool: invoke(args)
    alt 是终端工具
        Tool->>Term: run(cmd)
        Term->>Env: exec via local/docker/ssh/modal/...
        Env-->>Term: stdout/stderr/exit_code
        Term-->>Tool: ToolResult
    else 是其他工具（vision/web/feishu/...）
        Tool->>Tool: 直接执行 Python 函数
    end
    Tool-->>Reg: ToolResult
    Reg->>Store: 持久化大结果（OutputLimiter 截断）
    Reg-->>Loop: tool_result（写回 messages）
```

---

## UC-04：技能自主生成与调用

**参与者**：Agent / 个人用户 | **目标**：完成复杂任务后沉淀为可复用的 Skill；后续以 `/skill-name` 直接调用

```mermaid
sequenceDiagram
    actor User
    participant Agent
    participant Mgr as skill_manager_tool
    participant Pre as skill_preprocessing
    participant FS as ~/.hermes/skills/<cat>/SKILL.md
    participant Guard as skills_guard

    Note over User,Agent: ① 完成复杂任务后沉淀
    User->>Agent: "把这个流程保存为技能"
    Agent->>Mgr: create_skill(name, steps, frontmatter)
    Mgr->>Pre: 模板变量展开 (${HERMES_SKILL_DIR})
    Mgr->>FS: 写 SKILL.md（YAML frontmatter + markdown）

    Note over User,Agent: ② 后续调用
    User->>Agent: /report-weekly
    Agent->>FS: 加载 SKILL.md
    Agent->>Guard: skills_guard 校验来源 + 签名
    Guard-->>Agent: pass
    Agent->>Pre: 内联 shell 展开 (!`date`)
    Pre-->>Agent: 注入到 prompt
    Agent->>Agent: 按技能指令执行 tool-calling

    Note over User,Agent: ③ 改进（closed learning loop）
    Agent->>Mgr: amend_skill(name, improvement)
    Mgr->>FS: 覆写 SKILL.md
```

---

## UC-05：跨会话记忆与回忆

**参与者**：MemoryManager / FTS5 索引 | **目标**：跨多个历史会话检索，并构建持久用户画像

```mermaid
sequenceDiagram
    participant Turn as on_turn_start hook
    participant Mem as memory_manager
    participant USER as USER.md
    participant MEM as MEMORY.md
    participant FTS as session_search_tool (SQLite FTS5)
    participant Honcho as Honcho dialectic
    participant LLM

    Turn->>Mem: load_context(user_id, prompt)
    Mem->>USER: 读取用户偏好/画像
    Mem->>MEM: 读取 agent 自维护笔记
    Mem->>FTS: query(prompt, top_k)
    FTS-->>Mem: 命中的历史会话片段
    Mem->>Honcho: get_user_model()
    Honcho-->>Mem: dialectic snapshot
    Mem-->>Turn: assembled context block
    Turn->>LLM: prompt + memory block

    Note over Mem,FTS: 会话结束时<br/>on_session_end → LLM 摘要 → 写入 FTS5
```

---

## UC-06：上下文 DAG 压缩

**参与者**：context_engine / context_compressor | **目标**：当上下文逼近模型 token 上限时，自动压缩中段消息

```mermaid
sequenceDiagram
    participant Loop
    participant CE as context_engine
    participant Est as token_estimator
    participant Cmp as context_compressor
    participant LLM

    Loop->>CE: pre_flight(messages, threshold_tokens)
    CE->>Est: estimate(messages)
    Est-->>CE: total_tokens

    alt total_tokens 在阈值内
        CE-->>Loop: pass-through
    else 超出阈值
        CE->>Cmp: build_dag(messages, protect_first_n, protect_last_n)
        Cmp->>Cmp: 识别工具调用簇 / 引用关系
        Cmp->>LLM: summarize(中段簇)
        LLM-->>Cmp: 压缩后摘要节点
        Cmp-->>CE: 压缩后 messages
        CE-->>Loop: compressed messages
    end
```

> 头部（system + 早期关键消息）和尾部（最近 N 条）始终保留，中段以 DAG 节点为单位摘要，引用关系（`context_references`）保证可回溯。

---

## UC-07：子 Agent 委派（零上下文成本）

**参与者**：父 Agent / delegate_tool / 子 AIAgent | **目标**：把可独立完成的子任务委派给子 Agent，结果回写父上下文，但子 Agent 的中间步骤不污染父上下文

```mermaid
sequenceDiagram
    participant Parent as Parent AIAgent
    participant Del as delegate_tool
    participant RPC as Python RPC 通道
    participant Child as Subagent (独立进程/线程)
    participant Budget as IterationBudget

    Parent->>Del: delegate(task, model, max_iters)
    Del->>Budget: 切分配额
    Del->>RPC: spawn Child(task)
    RPC->>Child: 启动独立 AIAgent 实例
    Child->>Child: 自有 tool-calling loop（独立 budget）
    Child-->>RPC: 最终结果（仅结论）
    RPC-->>Del: result
    Del-->>Parent: tool_result（仅最终结论进入父上下文）

    Note over Parent,Child: 子 Agent 的工具调用、中间推理<br/>不写入父对话历史
```

---

## UC-08：模型切换

**参与者**：CLI 用户 | **目标**：在 200+ 模型间无代码切换

```mermaid
sequenceDiagram
    actor User
    participant CLI as hermes model
    participant Cat as model_catalog
    participant Sw as model_switch
    participant Norm as model_normalize
    participant Cfg as ~/.hermes/config

    User->>CLI: hermes model anthropic/claude-opus-4-7
    CLI->>Cat: lookup(provider, name)
    Cat-->>CLI: ModelEntry(api_mode, base_url, ...)
    CLI->>Norm: 规范化别名（如 nous/hermes-3 → 实际 endpoint）
    CLI->>Sw: 写入活动模型
    Sw->>Cfg: 持久化默认模型
    CLI-->>User: ✓ 已切换

    Note over CLI,Cfg: 下次启动时<br/>AIAgent.__init__ 读取该配置<br/>选择对应 *_adapter.py
```

---

## UC-09：定时任务调度

**参与者**：cron Scheduler | **目标**：在固定时间触发 Agent 任务（如每日报告、定期巡检）

```mermaid
sequenceDiagram
    participant Sched as cron/scheduler.py
    participant Jobs as cron/jobs.py
    participant Agent as run_agent.AIAgent
    participant GW as gateway/delivery
    participant User

    loop 每分钟 tick
        Sched->>Jobs: due_jobs(now)
        Jobs-->>Sched: [Job(prompt, channel, user)]
    end

    alt 命中
        Sched->>Agent: run_once(prompt, context)
        Agent->>Agent: tool-calling loop
        Agent-->>Sched: response
        Sched->>GW: deliver(channel, user, response)
        GW->>User: 投递（IM / 邮件 / CLI 通知）
    end
```

---

## UC-10：批量轨迹生成（研究场景）

**参与者**：研究人员 | **目标**：批量跑 prompts，收集结构化 trajectory 用于 RL/SFT 训练

```mermaid
sequenceDiagram
    actor Researcher
    participant Batch as batch_runner.py
    participant Pool as 进程池
    participant Agent as AIAgent (多个实例)
    participant Traj as agent/trajectory.py
    participant Out as JSONL 输出

    Researcher->>Batch: batch_runner --config datagen-config-examples/x.yaml
    Batch->>Pool: 启动 N 个 worker
    par 并发
        Pool->>Agent: run(prompt_i, model_i)
        Agent->>Traj: record(step, tool_call, result, tokens)
        Agent-->>Pool: trajectory_i
    end
    Pool-->>Batch: 汇总
    Batch->>Out: 写入 JSONL（含 messages, tool_calls, rewards）
    Out-->>Researcher: 训练数据集
```

---

## 用例与其他视图的追踪

| 用例 | 关联逻辑组件 | 关联进程 | 部署节点 |
|------|--------------|----------|----------|
| UC-01 CLI 对话 | hermes_cli, AIAgent | Main process | 用户主机 |
| UC-02 IM 接入 | gateway/, AIAgent | Gateway process + Agent process | 服务器 / VPS |
| UC-03 工具执行 | tools/registry, terminal_tool | Tool dispatcher | 用户主机 + 远端 backend |
| UC-04 技能 | skill_manager_tool, skills/ | Inline | 文件系统 |
| UC-05 记忆 | memory_manager, session_search_tool | Inline | SQLite + Honcho 服务 |
| UC-06 压缩 | context_engine, context_compressor | Pre-call hook | Inline |
| UC-07 子 Agent | delegate_tool | Subprocess / RPC | 同机或 Modal |
| UC-08 模型切换 | model_catalog, *_adapter.py | Config 修改 | 用户主机 |
| UC-09 调度 | cron/scheduler | 后台线程 | 同 Agent 主机 |
| UC-10 批量 | batch_runner, trajectory | 多进程 | 用户主机 / Modal 集群 |
