# 过程视图 (Process View)

> 描述运行时进程/线程划分、生命周期、并发与通信，体现"系统是怎么跑起来的"。

---

## 1. 运行时进程拓扑

```mermaid
flowchart TB
    subgraph Host["运行主机 / VPS / 本地"]
        subgraph CLIProc["进程 A: hermes CLI（前台）"]
            UI["curses UI 主线程\n输入/渲染"]
            Stream1["stream worker\nLLM token 流"]
            Tools1["tool dispatcher\n（同进程）"]
            Sched1["cron scheduler\n后台线程"]
            Mem1["memory I/O\n后台线程"]
        end

        subgraph GWProc["进程 B: gateway/run.py（常驻）"]
            GMain["GatewayProcess 主线程"]
            PWk1["Telegram worker"]
            PWk2["Discord worker"]
            PWk3["Slack worker"]
            PWk4["WhatsApp worker"]
            PWk5["Signal worker"]
            DelW["delivery worker"]
        end

        subgraph AgentSvc["进程 C: agent worker（被 gateway spawn / 复用）"]
            AIA["AIAgent 实例池\n（per-user / per-session）"]
        end

        subgraph WebProc["进程 D: web_server.py（可选）"]
            HTTP["HTTP / Webhook"]
        end

        subgraph SubAgent["进程 E: subagent\n（delegate_tool 临时 spawn）"]
            ChildAgent["子 AIAgent"]
        end

        subgraph ToolProc["进程 F: 工具子进程\n（terminal_tool / browser）"]
            Shell["shell / docker exec / ssh / chrome"]
        end
    end

    subgraph Remote["远端"]
        LLM["☁️ LLM Provider HTTP API"]
        Modal["Modal 函数 / 容器"]
        Daytona["Daytona workspace"]
        Sing["Singularity HPC node"]
        SSH["SSH 远端主机"]
    end

    UI -->|in-process| Tools1
    Tools1 -->|spawn| ToolProc
    Stream1 -->|HTTPS| LLM

    GMain --> PWk1
    GMain --> PWk2
    GMain --> PWk3
    GMain --> PWk4
    GMain --> PWk5
    PWk1 -.dispatch.-> AIA
    PWk2 -.dispatch.-> AIA
    PWk3 -.dispatch.-> AIA
    PWk4 -.dispatch.-> AIA
    PWk5 -.dispatch.-> AIA

    AIA -->|HTTPS| LLM
    AIA -->|spawn| ToolProc
    AIA -->|Python RPC| ChildAgent
    AIA --> DelW
    DelW --> PWk1
    DelW --> PWk2

    Sched1 -.触发.-> AIA
    HTTP -.转发.-> AIA

    ToolProc -->|docker exec| Modal
    ToolProc -->|API| Daytona
    ToolProc -->|sbatch| Sing
    ToolProc -->|ssh| SSH
```

> **进程边界设计**：CLI 用例下 Agent 与 UI 同进程；多平台 IM 部署下 Gateway 与 Agent 解耦，Agent 池化复用，便于 OOM 隔离。

---

## 2. AIAgent 主循环（核心运行时）

```mermaid
flowchart TB
    Start([用户/Gateway 发起 run_conversation]) --> InitCtx[on_turn_start: memory_manager 注入<br/>USER.md + MEMORY.md + FTS5 召回]
    InitCtx --> Build[prompt_builder 组装 messages<br/>+ tools schema]
    Build --> CE{context_engine<br/>pre_flight}
    CE -- 超 threshold --> Compress[context_compressor.compress<br/>DAG 摘要中段]
    CE -- 在限内 --> Cache[prompt_caching: 标记<br/>Anthropic cache breakpoints]
    Compress --> Cache
    Cache --> Rate[nous_rate_guard.pre_call]
    Rate --> Adp[Adapter.build_request]
    Adp --> Stream[Adapter.stream / call LLM]
    Stream --> Parse[parse delta + tool_calls]
    Parse --> Cb[stream_delta_callback / thinking_callback]
    Cb --> Decide{有 tool_call?}
    Decide -- 否 --> Final[最终文本回复]
    Final --> EndTurn[on_session_end / trajectory.flush]
    EndTurn --> End([返回 Response])

    Decide -- 是 --> Budget{IterationBudget<br/>remaining > 0?}
    Budget -- 否 --> Stop[Stop: budget exhausted]
    Stop --> EndTurn

    Budget -- 是 --> Intr{interrupt?<br/>steer?}
    Intr -- 中断 --> Final
    Intr -- 继续 --> Dispatch[ToolRegistry.dispatch]
    Dispatch --> Guard[Guards: path/url/schema/skills_guard]
    Guard -- 拒绝 --> Append1[追加 tool_error]
    Guard -- 通过 --> Exec[执行工具<br/>同步或 await]
    Exec --> Limit[OutputLimiter 截断 + 落盘]
    Limit --> Append2[追加 tool_result 到 messages]
    Append1 --> Build
    Append2 --> Build

    Stream -.error.-> Errc[error_classifier]
    Errc --> Retry{should_retry?}
    Retry -- 是 --> Backoff[backoff sleep]
    Backoff --> Adp
    Retry -- 否 --> Throw[抛出错误 + 写 trajectory]
    Throw --> EndTurn
```

---

## 3. 工具调用 → 终端后端的运行时分派

```mermaid
sequenceDiagram
    participant Agent as AIAgent
    participant Reg as ToolRegistry
    participant Term as TerminalTool
    participant Sel as backend selector<br/>(env: TERMINAL_ENV)
    participant Local as LocalBackend
    participant Docker as DockerBackend
    participant SSH as SshBackend
    participant Modal as ModalBackend
    participant Daytona as DaytonaBackend
    participant Sing as SingularityBackend

    Agent->>Reg: dispatch("terminal", {cmd: "..."})
    Reg->>Term: invoke(args, ctx)
    Term->>Sel: 根据 TERMINAL_ENV 选 backend
    alt local
        Sel->>Local: subprocess.Popen
        Local-->>Term: ToolResult
    else docker
        Sel->>Docker: docker exec <session container>
        Docker-->>Term: ToolResult
    else ssh
        Sel->>SSH: ssh user@host "cmd"
        SSH-->>Term: ToolResult
    else modal
        Sel->>Modal: Modal.Function.remote(cmd)
        Note right of Modal: 冷启动 vs 复用 sandbox
        Modal-->>Term: ToolResult
    else daytona
        Sel->>Daytona: REST API exec
        Daytona-->>Term: ToolResult
    else singularity
        Sel->>Sing: singularity exec / sbatch
        Sing-->>Term: ToolResult
    end
    Term-->>Reg: ToolResult
    Reg-->>Agent: 写回 messages
```

---

## 4. Gateway 消息流（多平台 fan-in / fan-out）

```mermaid
sequenceDiagram
    actor TgUser as Telegram User
    actor SlackUser as Slack User
    participant TgPoll as Telegram poller
    participant SlackWS as Slack WebSocket
    participant Stream as stream_consumer
    participant Pair as pairing
    participant Sess as session_context
    participant Agent as AIAgent (池中实例)
    participant Del as delivery worker
    participant TgAPI as Telegram API
    participant SlackAPI as Slack API

    par 入站并发
        TgUser->>TgPoll: 消息 A
        TgPoll->>Stream: event(A, telegram)
    and
        SlackUser->>SlackWS: 消息 B
        SlackWS->>Stream: event(B, slack)
    end

    Stream->>Pair: resolve(telegram, sender_A)
    Pair-->>Stream: user_X
    Stream->>Pair: resolve(slack, sender_B)
    Pair-->>Stream: user_Y

    Stream->>Sess: get_or_create(user_X, telegram)
    Stream->>Sess: get_or_create(user_Y, slack)

    par 并发处理
        Sess->>Agent: dispatch(prompt_A, ctx_X)
        Agent-->>Sess: response_A
        Sess->>Del: enqueue(telegram, user_X, response_A)
        Del->>TgAPI: sendMessage
    and
        Sess->>Agent: dispatch(prompt_B, ctx_Y)
        Agent-->>Sess: response_B
        Sess->>Del: enqueue(slack, user_Y, response_B)
        Del->>SlackAPI: chat.postMessage
    end
```

> **关键**：`stream_consumer` 是单写者多读者；`Agent 池` 通过 user_id 哈希分桶，保证同一用户的多次请求顺序处理；不同用户并行。

---

## 5. 子 Agent 委派（独立 budget + 独立上下文）

```mermaid
sequenceDiagram
    participant P as Parent AIAgent
    participant DT as delegate_tool
    participant RPC as Python RPC channel
    participant CW as Child Worker (subprocess)
    participant CA as Child AIAgent
    participant LLM

    P->>DT: invoke(task="搜索并比对 N 篇论文", model="claude-haiku")
    DT->>RPC: spawn child(task, budget=20)
    RPC->>CW: fork / Modal.Function
    CW->>CA: AIAgent(model, budget=20, parent_trajectory_ref)
    activate CA

    loop child loop
        CA->>LLM: chat
        LLM-->>CA: tool_call
        CA->>CA: 工具执行（独立沙箱）
    end

    CA-->>CW: 最终结论 + child trajectory
    deactivate CA
    CW-->>RPC: result
    RPC-->>DT: ToolResult(只含结论 + token 总计)
    DT-->>P: tool_result（写父 messages）

    Note over P,CA: 父 Agent 上下文只增加约 1 条 tool_result，<br/>子 Agent 内部数十轮交互不污染父 token 预算
```

---

## 6. 上下文压缩触发时序

```mermaid
sequenceDiagram
    participant Loop as run_conversation
    participant CE as context_engine
    participant Ref as context_references
    participant Cmp as context_compressor
    participant LLM as Summarizer LLM
    participant MM as memory_manager

    Loop->>CE: pre_flight(messages, model.context_window)
    CE->>CE: 估算 token，比较 threshold

    alt 超阈值
        CE->>MM: on_pre_compress(messages)
        Note right of MM: 给 memory 一次 "保存重要信息" 机会
        MM-->>CE: 已写 MEMORY.md
        CE->>Ref: 提取引用关系（tool_call ↔ tool_result）
        CE->>Cmp: build_dag(messages, protect_first_n, protect_last_n)
        Cmp->>Cmp: 把中段消息按 DAG 簇划分
        loop 每个簇
            Cmp->>LLM: summarize(cluster)
            LLM-->>Cmp: 摘要节点
        end
        Cmp-->>CE: compressed messages
        CE-->>Loop: 替换 messages
    else 未超
        CE-->>Loop: pass-through
    end
```

---

## 7. 中断与 Steer（用户在 tool loop 中插话）

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Running: run_conversation()
    Running --> WaitingLLM: call LLM
    WaitingLLM --> Streaming: first delta
    Streaming --> ToolCalling: tool_call detected
    ToolCalling --> WaitingLLM: tool_result appended

    Streaming --> Final: stop
    Final --> [*]

    state Interrupted {
        [*] --> Pending: user presses Ctrl-C / sends "/stop"
    }

    Streaming --> Interrupted: _interrupt_requested = true
    ToolCalling --> Interrupted: _interrupt_requested = true
    Interrupted --> Final: 终止当前轮

    state Steered {
        [*] --> SteerQueued: user types guidance
    }

    ToolCalling --> Steered: _pending_steer set
    Steered --> WaitingLLM: 把 guidance 注入下一次 LLM 调用
```

---

## 8. Cron 调度时序

```mermaid
sequenceDiagram
    participant Tick as scheduler tick (1/min)
    participant Sched as cron/scheduler
    participant Jobs as cron/jobs
    participant Reg as process_registry
    participant Agent as AIAgent
    participant Del as delivery
    participant User

    loop 每分钟
        Tick->>Sched: heartbeat
        Sched->>Jobs: due_jobs(now)
        Jobs-->>Sched: [job1, job2]
    end

    par 并发
        Sched->>Reg: claim(job1) (避免重复)
        Reg-->>Sched: ok
        Sched->>Agent: run_once(job1.prompt, user1)
        Agent-->>Sched: response
        Sched->>Del: send(job1.channel, user1, response)
        Del->>User: 投递
    and
        Sched->>Reg: claim(job2)
        Sched->>Agent: run_once(job2.prompt, user2)
    end
```

---

## 9. 错误处理与 Provider 限流

```mermaid
sequenceDiagram
    participant Loop as AIAgent loop
    participant Adp as ProviderAdapter
    participant Guard as nous_rate_guard
    participant Track as rate_limit_tracker
    participant Cls as error_classifier
    participant LLM

    Loop->>Guard: pre_call(provider)
    Guard->>Track: 当前 RPM/TPM 已用?
    Track-->>Guard: usage
    alt 接近上限
        Guard-->>Loop: sleep(jitter)
    end
    Loop->>Adp: stream(messages)
    Adp->>LLM: HTTP

    alt 200 OK
        LLM-->>Adp: stream
        Adp-->>Loop: deltas
        Loop->>Track: track(provider, usage_headers, cost)
    else 429 / 5xx / network
        LLM-->>Adp: error
        Adp->>Cls: classify(exc)
        Cls-->>Adp: ErrorClass{retryable, backoff_s}
        alt retryable
            Adp->>Adp: sleep(backoff)
            Adp->>LLM: 重试（最多 N 次）
        else 不可重试
            Adp-->>Loop: raise
            Loop->>Loop: 写入 trajectory + memory
        end
    end
```

---

## 10. 启动 / 关闭顺序

```mermaid
sequenceDiagram
    participant SH as install.sh / hermes 命令
    participant CLI as hermes_cli/main
    participant Setup as setup.py
    participant Env as env_loader
    participant Cred as credential_pool
    participant Catalog as model_catalog
    participant Tools as tools/registry
    participant Skills as skills/
    participant MM as memory_manager
    participant UI as curses_ui
    participant Sched as cron scheduler

    SH->>CLI: hermes
    CLI->>Setup: 检查 ~/.hermes 目录
    CLI->>Env: 加载 .env / 系统环境
    CLI->>Cred: 收集所有 credential_sources
    CLI->>Catalog: 载入 model_catalog (含别名)
    CLI->>Tools: register 内建工具 + Guards
    CLI->>Skills: 扫描 ~/.hermes/skills/ 注册技能
    CLI->>MM: 加载 USER.md / MEMORY.md
    CLI->>Sched: 启动后台线程（如有 cron job）
    CLI->>UI: curses_ui.run()

    Note over UI,Sched: 用户交互...

    UI->>Sched: 关闭信号
    Sched-->>UI: stop
    UI->>MM: flush
    UI->>Tools: cleanup（关闭子进程 / browser）
    UI-->>SH: exit
```
