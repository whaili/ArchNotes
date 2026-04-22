# OpenClaw 架构调研笔记

> 调研日期：2026-04-22  
> 代码版本：openclaw main 分支  

---

## 目录

1. [多用户支持](#1-多用户支持)
2. [执行环境与 Sandbox](#2-执行环境与-sandbox)
3. [资源消耗与镜像大小](#3-资源消耗与镜像大小)
4. [镜像 Default vs Slim 差异](#4-镜像-default-vs-slim-差异)
5. [本地缓存后镜像大小的影响](#5-本地缓存后镜像大小的影响)
6. [资源消耗分布：主进程 vs Sandbox](#6-资源消耗分布主进程-vs-sandbox)
7. [Sandbox 资源动态调整](#7-sandbox-资源动态调整)
8. [Sandbox 懒启动机制](#8-sandbox-懒启动机制)
9. [工具调用执行位置澄清](#9-工具调用执行位置澄清)
10. [Prune 配置](#10-prune-配置)
11. [外部执行环境与 SSH Backend](#11-外部执行环境与-ssh-backend)
12. [MCP Loopback 服务](#12-mcp-loopback-服务)

---

## 1. 多用户支持

### 现状

OpenClaw 原生是**单用户 / 单 Operator 架构**，核心层面没有多用户隔离：

| 层面 | 现状 |
|------|------|
| 配置 | 单一 `~/.openclaw/config.json`，无用户分区 |
| 数据目录 | 单一 `~/.openclaw/`，按 agent 分层而非按用户 |
| 网关认证 | 设备级 token，非用户级 |
| Session | 按 `(agentId, channel, conversation)` 索引，无用户维度 |
| 凭据 / 内存 | 全局或 agent 级，无用户隔离 |

`session.scope: "per-sender"` 有有限的"按发送者分组 session"能力，但不是真正的用户隔离。

### 推荐方案：进程级隔离 + 管理层包装

不修改 core 代码、可持续跟随上游的最干净路径：

```
用户请求 ──► 网关代理（User Router / Auth Proxy）
                    │              │
          ┌─────────▼──┐  ┌────────▼──┐
          │ openclaw   │  │ openclaw  │  ...
          │ (user A)   │  │ (user B)  │
          │ 独立进程   │  │ 独立进程  │
          └─────────┬──┘  └────┬──────┘
                    │          │
          ~/.openclaw/users/A  ~/.openclaw/users/B
```

每个用户通过环境变量指向独立数据目录：

```bash
OPENCLAW_STATE_DIR=/var/openclaw/users/alice \
OPENCLAW_CONFIG_PATH=/var/openclaw/users/alice/config.json \
OPENCLAW_GATEWAY_PORT=3101 \
  pnpm openclaw start
```

**优点**：零 core 修改、完全隔离、可按需启停。  
**缺点**：资源随用户数线性增长，需自建用户管理层。

---

## 2. 执行环境与 Sandbox

### 执行模型：混合架构

```
Agent 逻辑（工具路由、策略检查）  ← 在进程内运行
        ↓
MCP JSON-RPC 分发（HTTP loopback, 127.0.0.1:随机端口）
        ↓
exec / process 工具
        ↓  spawn 子进程
  ┌─────────────────────────────────┐
  │ 无沙箱：直接 child_process.spawn │
  │ 有沙箱：docker run ...           │
  │ SSH：  ssh <target> ...          │
  └─────────────────────────────────┘
```

**并非所有操作都在 openclaw 进程内**：agent 判断逻辑和策略检查在进程内，实际 shell 命令通过子进程执行，执行位置由 `host=` 配置决定（`sandbox` / `gateway` / `node`）。

### Sandbox 支持（基于 Docker）

| 能力 | 支持 |
|------|------|
| 独立文件系统 | ✅ |
| 内存 / CPU / PID 限制（cgroups） | ✅ |
| 网络隔离 | ✅ |
| 只读根文件系统 | ✅ |
| Capability dropping | ✅ |
| Seccomp / AppArmor | ✅（通过 Docker 配置） |
| gVisor / nsjail / bubblewrap | ❌ |

沙箱是**可选的**，默认关闭。

### 工具调用的精确执行边界

| 发生在哪 | 做什么 |
|---------|-------|
| openclaw 主进程 | 接收 tool_call、路由、审批、构造 docker exec 参数、捕获 stdout/stderr |
| 主进程的子进程 | 执行 `docker exec ...` 命令本身 |
| Docker 容器内 | 实际的 shell 命令（`npm test`、`git clone` 等） |

---

## 3. 资源消耗与镜像大小

### 镜像大小估算

| 镜像 | 基础镜像 | 估算压缩后大小 |
|------|---------|--------------|
| openclaw 主进程（default） | `node:24-bookworm` | ~700 MB–1.1 GB |
| openclaw 主进程（slim） | `node:24-bookworm-slim` | ~200–350 MB |
| + 浏览器（Chromium + Xvfb） | — | +300 MB |
| + Docker CLI | — | +50 MB |
| sandbox-base | `debian:bookworm-slim` | ~80–120 MB |
| sandbox-common（含 golang/rust/node/python） | sandbox-base | ~800 MB–2 GB |
| sandbox-browser（Chromium + VNC） | `debian:bookworm-slim` | ~500–800 MB |

### openclaw 主进程运行时内存

```
空载 RSS：~150–300 MB
活跃对话：~300–600 MB
Node.js 默认堆上限：~1.5 GB（无显式运行时限制）
```

**注意**：LanceDB 向量库和 sqlite-vec 以 native addon 形式跑在主进程内，记忆功能开启时会有额外内存和 CPU 开销。

### Sandbox 容器资源（完全由工作负载决定）

| 任务类型 | CPU | 内存 | 存活时长 |
|---------|-----|------|---------|
| 轻量（ls, grep） | <1% | <20 MB | 秒级 |
| 中等（跑测试、编译 JS） | 单核 | 100–500 MB | 分钟级 |
| 重型（Rust/C++ 编译） | 多核满载 | 1–4 GB | 数分钟 |
| 浏览器自动化 | 脉冲式 | 300–800 MB | 按需 |

### Sandbox 默认资源参数

`src/config/types.sandbox.ts`：

| 参数 | 默认值 |
|------|-------|
| `memory` | **未设置**（需手动配置） |
| `cpus` | **未设置**（需手动配置） |
| `pidsLimit` | 0（无限制） |
| `network` | `"none"` |
| `readOnlyRoot` | `true` |
| `capDrop` | `["ALL"]` |
| `tmpfs` | `/tmp`, `/var/tmp`, `/run` |

> ⚠️ 多用户场景下**必须**显式设置 `memory`、`cpus`、`pidsLimit`，否则等于无限制。

### 计费基础设施

`src/infra/session-cost-usage.ts`（1300+ 行）已内置完整 token 追踪：

- input / output / cache_read / cache_write tokens
- 按模型、Provider、天维度聚合
- 支持阶梯定价（tiered pricing）
- Gateway 暴露 `sessions.usage()` 查询端点（30s LRU 缓存）

**缺失**：无原生限额执行、无用户维度 `userId`、无计费 webhook。

---

## 4. 镜像 Default vs Slim 差异

**唯一区别是 Runtime 基础镜像**，构建阶段两者完全相同（都用 `node:24-bookworm`）：

| | default | slim |
|--|---------|------|
| 基础镜像 | `node:24-bookworm` | `node:24-bookworm-slim` |
| 基础镜像大小 | ~950 MB | ~250 MB |
| Debian 完整工具链 | ✅ | ❌ 裁剪 |
| 额外安装的包 | `procps hostname curl git lsof openssl`（两者相同） | 同左 |
| openclaw 应用代码 | 完全相同 | 完全相同 |
| 功能差异 | **无** | **无** |

多用户部署推荐 **slim**，openclaw 运行时不依赖额外的 Debian 工具。

---

## 5. 本地缓存后镜像大小的影响

镜像缓存后，**slim vs default 对运行时性能几乎没有实质影响**：

| 维度 | 影响 | 原因 |
|------|------|------|
| 运行时 RAM | 基本无差距 | overlayfs 按需映射，未访问的文件不进 page cache |
| CPU | 无差距 | 与镜像大小无关 |
| 容器启动时间 | 微小（<100ms） | overlayfs 挂载层略多 |
| **磁盘占用** | ✅ 有差距 | 多占 ~650 MB，但层共享，同机多容器只存一份 |
| **安全攻击面** | ✅ 有差距 | 多 300+ 个包 = 更多潜在 CVE |

选 slim 的真实理由：**减小安全攻击面** + **无本地缓存时节省拉取时间**。

---

## 6. 资源消耗分布：主进程 vs Sandbox

### 消耗重心随场景变化

| 场景 | 主进程 | Sandbox |
|------|--------|---------|
| 纯对话 / 问答（无工具调用） | 几乎全部 | 基本空闲 |
| 代码执行 / 文件操作（密集） | 仅编排层 | **大头** |
| 向量记忆开启 + 长对话 | 内存压力显著 | 视任务而定 |

### 主进程 CPU 模式

```
LLM 等待期间 ────── 接近 0 ──────
LLM 返回 token 期间 ─── 脉冲 ───
工具路由 / 审批 ────── 短暂脉冲 ──
向量搜索（LanceDB）─── 短暂脉冲 ──
```

推理本身在 LLM Provider 服务器上计算，不消耗本地 CPU。

### 多用户资源规划建议

- **主进程**：每进程硬限 1–2 GB 内存，防止单用户泄漏影响他人
- **Sandbox**：按任务类型设不同资源 profile，不能用全局单一限制

---

## 7. Sandbox 资源动态调整

### 结论：不支持运行时动态调整

资源限制在**容器创建时**通过 `docker run --memory=... --cpus=...` 写死，运行期间无法热改：

```
config.json（静态配置）
        ↓
resolveSandboxDockerConfig()    ← 启动时解析一次
        ↓
ensureSandboxContainer()        ← 创建容器时传入
        ↓
docker run --memory=... --cpus=...   ← 不可更改
```

`exec` 工具的调用参数中**没有** `memory`、`cpus`、`pidsLimit` 字段，LLM 无法在 tool_call 时指定资源。

### 现有灵活性：per-agent 静态分层

```json
{
  "agents": {
    "defaults": { "sandbox": { "docker": { "memory": "512m" } } },
    "heavy-build": { "sandbox": { "docker": { "memory": "8g", "cpus": 4 } } }
  }
}
```

预定义多个资源 profile，由 LLM 决定 spawn 哪个子 agent，间接实现"按任务选规格"——但本质是配置时预定义，非运行时动态。

---

## 8. Sandbox 懒启动机制

**容器在第一次 agent run 开始时才创建，不是服务器启动时。**

```
openclaw 服务器启动     ← 无任何容器，零资源占用
        ↓
用户发来第一条消息
        ↓
agent run 开始（attempt.ts:432）
        ↓
resolveSandboxContext()
  ├─ 容器不存在 → docker create ... sleep infinity → docker start
  └─ 容器已存在 → 直接复用（检查 config hash）
        ↓
后续工具调用 → docker exec <container> <command>
        ↓
空闲 idleHours（默认 24h）→ 自动 prune
最大 maxAgeDays（默认 7d）→ 强制 prune
```

### scope 决定容器复用粒度

| scope | 容器数量 | 触发时机 |
|-------|---------|---------|
| `session` | 每个 session 一个 | 该 session 第一次 run |
| `agent`（默认） | 每个 agent 一个 | 该 agent 第一次 run |
| `shared` | 全局一个 | 任何 agent 第一次 run |

---

## 9. 工具调用执行位置澄清

"工具调用在 openclaw 中"指的是**控制逻辑**，不是 shell 命令的实际执行：

```
LLM 返回 tool_call: { name: "exec", params: { command: "npm test" } }
                ↓
        openclaw 主进程（控制层）
          ├─ 路由：host=sandbox → 走 docker exec 路径
          ├─ 审批：allowlist / policy 检查
          ├─ 构造：["docker", "exec", "<container>", "/bin/sh", "-lc", "npm test"]
          └─ supervisor.spawn(argv)
                        ↓
              主进程子进程执行 docker exec
                        ↓
                  Docker 容器内
                    └─ /bin/sh -lc "npm test"  ← 真正在这里跑
```

---

## 10. Prune 配置

空闲清理和最大存活时间均**可配置**，通过 `prune` 字段覆盖默认值：

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "prune": {
          "idleHours": 2,    // 默认 24
          "maxAgeDays": 1    // 默认 7
        }
      }
    }
  }
}
```

支持全局配置和 per-agent 级别独立配置。

---

## 11. 外部执行环境与 SSH Backend

### SSH Backend：连接外部已有环境

openclaw 不负责启动 SSH 目标，只负责连接并执行命令：

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "backend": "ssh",
        "ssh": {
          "target": "user@192.168.1.100",
          "identityFile": "~/.ssh/id_ed25519",
          "workspaceRoot": "/tmp/openclaw-sandboxes"
        }
      }
    }
  }
}
```

目标可以是裸机、VM、已在运行的容器（开了 SSH 即可），openclaw 完全不管其生命周期。

### Backend 是插件化的

`src/agents/sandbox/backend.ts` 暴露 `registerSandboxBackend()`，插件可注册任意 backend：

```typescript
registerSandboxBackend("my-k8s", {
  factory: createK8sPodBackend,   // 连接 K8s Pod、Firecracker、任意执行环境
  manager: k8sBackendManager,
});
```

### 三种 Backend 对比

| backend | 环境由谁启动 | 适合场景 |
|---------|------------|---------|
| `docker`（默认） | openclaw 自己创建管理 | 本机隔离执行 |
| `ssh` | **外部已有**，openclaw 只连接 | 远端服务器、已有 VM、CI 机器 |
| 自定义插件 | 完全由插件决定 | K8s Pod、Firecracker、任意执行环境 |

---

## 12. MCP Loopback 服务

**MCP（Model Context Protocol）** 是 Anthropic 定义的工具调用标准协议。  
**Loopback** 指只监听 `127.0.0.1`，不对外暴露。

### 作用

充当 **"LLM ↔ 工具实现"** 之间的中间层：

```
LLM 返回 tool_call JSON
        ↓
  agent runner（推理主循环）
        ↓  HTTP POST 127.0.0.1:<随机端口>
        ↓  Authorization: Bearer <token>
        ↓  JSON-RPC 2.0
  MCP loopback server（跑在 openclaw 主进程里）
        ↓  工具路由、权限检查、策略执行
  tool.execute()
        ↓
  exec → docker exec / ssh / 直接 spawn
```

| 特性 | 说明 |
|------|------|
| 协议 | JSON-RPC 2.0 over HTTP |
| 监听地址 | `127.0.0.1:随机端口`（每次启动随机） |
| 认证 | Bearer Token（owner / non-owner 两级） |
| 工具隔离 | 按 `sessionKey + accountId` 做 scoped 工具集 |

### 为何 Sandbox 容器无法访问 MCP

MCP loopback 绑定宿主机的 `127.0.0.1`。Docker 容器有独立网络命名空间，容器内的 `127.0.0.1` 指向容器自身，无法访问宿主机的 loopback 端口——这也是"把工具层搬入容器"改动量大的根本原因。
