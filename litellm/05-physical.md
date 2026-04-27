# 物理视图 (Physical / Deployment View)

> 描述系统在基础设施上的部署拓扑、节点分配、网络结构和运维特性。

---

## 1. 标准 Docker Compose 部署

> 适用于单机自托管场景（小团队 / 快速启动）

```mermaid
flowchart TB
    Client(["🖥️ Client App"])
    AdminUI(["🖥️ Admin Browser"])

    subgraph DockerHost["Docker Host（单机）"]
        subgraph LC["litellm Container :4000"]
            API["FastAPI + Uvicorn\nproxy_server.py"]
            SCHED["APScheduler\n后台 Jobs"]
            SDK2["LiteLLM SDK\nlitellm/"]
        end

        subgraph PG["PostgreSQL Container :5432"]
            PGDB[("litellm DB\nkeys / teams / spend_logs")]
        end

        subgraph PROM["Prometheus Container :9090"]
            MS["Metrics Scraper"]
        end

        PGVOL[/"postgres_data volume"/]
        PROMVOL[/"prometheus_data volume"/]
    end

    subgraph External["External"]
        PROVIDERS["☁️ LLM Providers\nOpenAI / Anthropic / Bedrock / Azure"]
        REDIS_EXT[("Redis（可选）")]
    end

    Client -->|HTTP :4000| LC
    AdminUI -->|HTTP :4000/ui| LC
    LC -->|TCP :5432 Prisma| PGDB
    LC -->|TCP :6379 optional| REDIS_EXT
    LC -->|HTTPS outbound| PROVIDERS
    MS -->|GET :4000/metrics| LC
    PGDB --- PGVOL
    MS --- PROMVOL
```

---

## 2. Kubernetes 生产部署

> 适用于企业级高可用场景

```mermaid
flowchart TB
    Clients(["🌐 Clients"])

    subgraph K8s["Kubernetes Cluster"]
        subgraph Ingress["Ingress / Load Balancer"]
            LB["nginx-ingress / AWS ALB"]
        end

        subgraph Deploy["litellm-proxy Deployment（ReplicaSet）"]
            P1["Pod 1\nlitellm-proxy :4000"]
            P2["Pod 2\nlitellm-proxy :4000"]
            PN["Pod N\n（HPA 自动扩缩）"]
        end

        subgraph Data["Data Tier"]
            PGS[("PostgreSQL StatefulSet\n或 RDS / Cloud SQL")]
            REDIS_K8S[("Redis StatefulSet\n或 ElastiCache")]
        end

        subgraph Monitor["Monitoring Namespace"]
            PROM2["Prometheus"]
            GRAF["Grafana"]
            ALERT["AlertManager"]
        end

        subgraph Secrets["Secret Store\nVault / k8s Secret"]
            CREDS["API Keys\nDB Credentials"]
        end
    end

    subgraph ExtSvc["External Services"]
        LLMS2["☁️ LLM Providers"]
        OBS["Observability SaaS\nDatadog / Langfuse"]
    end

    Clients -->|HTTPS :443| LB
    LB --> P1 & P2 & PN
    P1 & P2 & PN --> PGS
    P1 & P2 & PN --> REDIS_K8S
    P1 & P2 & PN --> CREDS
    PROM2 --> P1 & P2 & PN
    GRAF --> PROM2
    ALERT --> PROM2
    P1 & P2 & PN -->|HTTPS egress| LLMS2
    P1 & P2 & PN -->|async callbacks| OBS
```

---

## 3. AWS 云托管架构

```mermaid
flowchart TB
    C(["🌐 Clients"])

    subgraph VPC["AWS VPC"]
        subgraph PubNet["Public Subnet"]
            ALB["ALB\nApplication Load Balancer"]
        end

        subgraph AppNet["Private Subnet — App Tier"]
            subgraph ECS["ECS Fargate / EKS"]
                TASK["litellm-proxy Tasks（多副本）"]
            end
        end

        subgraph DataNet["Private Subnet — Data Tier"]
            RDS[("RDS PostgreSQL\nMulti-AZ")]
            EC[("ElastiCache Redis\nCluster Mode")]
        end

        subgraph AwsSvc["AWS Services"]
            SM["Secrets Manager\nAPI Keys"]
            CW["CloudWatch\nLogs + Metrics"]
            S3[("S3\nLLM Response Cache")]
        end
    end

    subgraph ExtCloud["External"]
        LLMS3["☁️ LLM Providers\nOpenAI / Bedrock / Anthropic"]
        OBS2["Observability\nDatadog / Langfuse"]
    end

    C -->|HTTPS :443| ALB
    ALB -->|HTTP :4000| TASK
    TASK --> RDS
    TASK --> EC
    TASK --> SM
    TASK --> S3
    TASK --> CW
    TASK -->|HTTPS egress| LLMS3
    TASK -->|HTTPS async| OBS2
```

---

## 4. 节点职责与内部组件

```mermaid
flowchart LR
    subgraph ProxyNode["LiteLLM Proxy Node"]
        FASTAPI["FastAPI\nHTTP Server\n接收请求 / 认证 / 路由 / 响应"]
        SCHED2["APScheduler\nupdate_spend 60s\nreset_budget 10min\nhealth_check continuous\nrotate_keys 1hr"]
        SDK3["LiteLLM SDK\nProvider 协议转换\n流式响应\n成本计算\n缓存查询"]
        RC["Redis Client\ncaching/redis_cache.py"]
        PC["Prisma Client\nDB ORM"]
    end

    subgraph RedisNode["Redis Node"]
        AKC[("API Key Cache")]
        RTC[("RPM/TPM Counters")]
        SQ[("Spend Queue")]
        DC2[("Deployment Cooldowns")]
        LRC[("LLM Response Cache")]
    end

    subgraph PGNode["PostgreSQL Node"]
        VT[("VerificationToken\nVirtual Keys")]
        TT[("TeamTable")]
        UT[("UserTable")]
        SL[("SpendLogs")]
        MT[("ProxyModelTable")]
    end

    FASTAPI --> RC
    FASTAPI --> PC
    SCHED2 --> PC
    SDK3 --> RC
    PC --> VT & TT & UT & SL & MT
    RC --> AKC & RTC & SQ & DC2 & LRC
```

---

## 5. 网络流量与端口

| 方向 | 源 | 目标 | 端口 | 协议 | 说明 |
|------|-----|------|------|------|------|
| Inbound | Client App | LiteLLM Proxy | 4000 | HTTP/HTTPS | API 请求 |
| Inbound | Admin Browser | LiteLLM Proxy | 4000/ui | HTTP/HTTPS | 管理 UI |
| Inbound | Prometheus | LiteLLM Proxy | 4000/metrics | HTTP | 指标采集 |
| Outbound | LiteLLM Proxy | PostgreSQL | 5432 | TCP | 持久化 |
| Outbound | LiteLLM Proxy | Redis | 6379 | TCP | 缓存/队列 |
| Outbound | LiteLLM Proxy | OpenAI API | 443 | HTTPS | LLM 调用 |
| Outbound | LiteLLM Proxy | Anthropic API | 443 | HTTPS | LLM 调用 |
| Outbound | LiteLLM Proxy | AWS Bedrock | 443 | HTTPS | LLM 调用 |
| Outbound | LiteLLM Proxy | Langfuse/Datadog | 443 | HTTPS | 异步日志 |

---

## 6. 扩展性与高可用设计

```mermaid
flowchart TB
    subgraph StatelessProxy["水平扩展 — Stateless Proxy"]
        I1["Proxy 实例 1"]
        I2["Proxy 实例 2"]
        IN["Proxy 实例 N\nHPA 自动扩缩"]
        NOTE1["Proxy 无本地状态\nAPI Key 状态存 Redis\n账单数据存 PostgreSQL\n实例间不通信\n可无限水平扩展"]
    end

    subgraph RedisHA["Redis 高可用"]
        RP["Redis Primary"]
        RR1["Redis Replica 1"]
        RR2["Redis Replica 2"]
        NOTE2["Redis Cluster Mode 支持\n分片存储 TPM/RPM 计数"]
        RP -->|replication| RR1 & RR2
    end

    subgraph PGHA["PostgreSQL 高可用"]
        PGP[("Primary\nRead/Write")]
        PGS2[("Standby\nHot Standby")]
        NOTE3["写操作 → Primary\n可配置 Read Replica\n分担查询压力"]
        PGP -->|streaming replication| PGS2
    end

    I1 & I2 & IN --> RP
    I1 & I2 & IN --> PGP
```
