# 物理视图 (Physical / Deployment View)

> 描述系统在基础设施上的部署拓扑、节点分配、网络结构和运维特性。

---

## 1. 标准 Docker Compose 部署

> 适用于单机自托管场景（小团队 / 快速启动）

```plantuml
@startuml Physical-DockerCompose
skinparam nodeStyle rectangle

node "Docker Host (单机)" {

  node "litellm Container\n:4000" as LC {
    component "FastAPI + Uvicorn\n(proxy_server.py)" as API
    component "APScheduler\n(后台 Jobs)" as SCHED
    component "LiteLLM SDK\n(litellm/)" as SDK
  }

  node "PostgreSQL Container\n:5432" as PG {
    database "litellm DB\n(keys, teams, spend_logs)" as PGDB
  }

  node "Prometheus Container\n:9090" as PROM {
    component "Metrics Scraper" as MS
  }

  volume "postgres_data" as PGVOL
  volume "prometheus_data" as PROMVOL
}

cloud "External LLM Providers" as PROVIDERS {
  component "OpenAI API"
  component "Anthropic API"
  component "Bedrock / Vertex AI"
  component "Azure OpenAI"
}

cloud "Optional Redis\n(External)" as REDIS

actor "Client App" as CLIENT
actor "Admin UI" as ADMIN_UI

CLIENT --> LC : HTTP :4000\n/v1/chat/completions
ADMIN_UI --> LC : HTTP :4000\n/ui (React SPA)
LC --> PG : TCP :5432\nPrisma Client
LC --> REDIS : TCP :6379\n(Redis optional)
LC --> PROVIDERS : HTTPS (outbound)
PROM --> LC : GET :4000/metrics
PG --> PGVOL
PROM --> PROMVOL
@enduml
```

---

## 2. Kubernetes 生产部署

> 适用于企业级高可用场景

```plantuml
@startuml Physical-K8s
skinparam nodeStyle rectangle

cloud "Kubernetes Cluster" {

  node "Ingress / Load Balancer" as ING {
    component "nginx-ingress\n/ AWS ALB" as LB
  }

  node "litellm-proxy Deployment\n(ReplicaSet)" as DEPLOY {
    node "Pod 1" {
      component "litellm-proxy\n:4000" as P1
    }
    node "Pod 2" {
      component "litellm-proxy\n:4000" as P2
    }
    node "Pod N\n(HPA)" {
      component "litellm-proxy\n:4000" as PN
    }
  }

  node "PostgreSQL StatefulSet\n(或 RDS / Cloud SQL)" as PGSS {
    database "litellm-db" as PGD
  }

  node "Redis StatefulSet\n(或 ElastiCache)" as REDIS {
    database "litellm-cache" as RD
  }

  node "Monitoring Namespace" {
    component "Prometheus" as PROM
    component "Grafana" as GRAF
    component "AlertManager" as ALERT
  }

  node "Secret Store\n(Vault / k8s Secret)" as SEC {
    component "API Keys\nDB Credentials" as CREDS
  }

  ING --> P1
  ING --> P2
  ING --> PN

  P1 --> PGD
  P2 --> PGD
  PN --> PGD

  P1 --> RD
  P2 --> RD
  PN --> RD

  P1 --> SEC
  P2 --> SEC

  PROM --> P1
  PROM --> P2
  GRAF --> PROM
  ALERT --> PROM
}

cloud "External LLM Providers" as EXT
cloud "Observability SaaS\n(Langfuse / Datadog / Slack)" as OBS

P1 --> EXT : HTTPS (egress)
P2 --> EXT : HTTPS (egress)
P1 --> OBS : async callbacks
P2 --> OBS : async callbacks

actor "Clients" as C
C --> ING : HTTPS :443
@enduml
```

---

## 3. 云托管架构（AWS 参考）

```plantuml
@startuml Physical-AWS
skinparam nodeStyle rectangle

cloud "AWS Cloud" {

  node "VPC" {

    node "Public Subnet" {
      component "ALB\n(Application Load Balancer)" as ALB
    }

    node "Private Subnet - App Tier" {
      node "ECS Fargate / EKS" {
        component "litellm-proxy Task\n(多副本)" as ECS
      }
    }

    node "Private Subnet - Data Tier" {
      database "RDS PostgreSQL\n(Multi-AZ)" as RDS
      database "ElastiCache Redis\n(Cluster Mode)" as EC
    }

    node "AWS Services" {
      component "Secrets Manager\n(API Keys)" as SM
      component "CloudWatch\n(Logs + Metrics)" as CW
      component "S3\n(LLM Response Cache)" as S3
    }
  }
}

cloud "LLM Providers\n(OpenAI / Bedrock / Anthropic)" as LLMS
cloud "Observability\n(Datadog / Langfuse)" as OBS
actor "Client" as C

C --> ALB : HTTPS :443
ALB --> ECS : HTTP :4000
ECS --> RDS : TCP :5432
ECS --> EC : TCP :6379
ECS --> SM : HTTPS (get secrets)
ECS --> S3 : HTTPS (cache)
ECS --> CW : logs/metrics
ECS --> LLMS : HTTPS (egress)
ECS --> OBS : HTTPS (async)
@enduml
```

---

## 4. 节点职责说明

```plantuml
@startuml Physical-Responsibilities

package "LiteLLM Proxy Node" {
  component "FastAPI\n(HTTP Server)" as FASTAPI
  component "APScheduler\n后台任务" as SCHED
  component "LiteLLM SDK\n(Provider Adapters)" as SDK
  component "Redis Client\n(caching/redis_cache.py)" as RC
  component "Prisma Client\n(DB ORM)" as PC
}

note right of FASTAPI
  - 接收 OpenAI 格式请求
  - 认证 + 限流
  - 路由 + 调用 SDK
  - 返回响应 + 写 headers
end note

note right of SCHED
  - update_spend (60s)
  - reset_budget (10-12min)
  - add_deployment (10s)
  - health_check (continuous)
  - rotate_keys (1hr)
end note

note right of SDK
  - Provider 协议转换
  - 流式响应处理
  - 成本计算
  - 缓存查询/存储
end note

package "Redis Node" {
  database "API Key Cache" as AKC
  database "RPM/TPM Counters" as RTC
  database "Spend Queue" as SQ
  database "Deployment Cooldowns" as DC
  database "LLM Response Cache" as LRC
}

package "PostgreSQL Node" {
  database "LiteLLM_VerificationToken\n(Virtual Keys)" as VT
  database "LiteLLM_TeamTable" as TT
  database "LiteLLM_UserTable" as UT
  database "LiteLLM_SpendLogs" as SL
  database "LiteLLM_ProxyModelTable" as MT
  database "LiteLLM_OrganizationTable" as OT
}

FASTAPI --> RC : key auth cache
FASTAPI --> PC : key lookup (cache miss)
SCHED --> PC : batch spend writes
SDK --> RC : response cache + TPM/RPM
PC --> VT
PC --> TT
PC --> UT
PC --> SL
PC --> MT
RC --> AKC
RC --> RTC
RC --> SQ
RC --> DC
RC --> LRC
@enduml
```

---

## 5. 网络流量与端口

| 方向 | 源 | 目标 | 端口 | 协议 | 说明 |
|------|-----|------|------|------|------|
| Inbound | Client | LiteLLM Proxy | 4000 | HTTP/HTTPS | API 请求 |
| Inbound | Admin Browser | LiteLLM Proxy | 4000/ui | HTTP/HTTPS | 管理 UI |
| Inbound | Prometheus | LiteLLM Proxy | 4000/metrics | HTTP | 指标采集 |
| Outbound | LiteLLM Proxy | PostgreSQL | 5432 | TCP | 持久化 |
| Outbound | LiteLLM Proxy | Redis | 6379 | TCP | 缓存/队列 |
| Outbound | LiteLLM Proxy | OpenAI API | 443 | HTTPS | LLM 调用 |
| Outbound | LiteLLM Proxy | Anthropic API | 443 | HTTPS | LLM 调用 |
| Outbound | LiteLLM Proxy | AWS Bedrock | 443 | HTTPS | LLM 调用 |
| Outbound | LiteLLM Proxy | Langfuse/Datadog | 443 | HTTPS | 异步日志 |

---

## 6. 扩展性与容灾特性

```plantuml
@startuml Physical-Scalability

package "水平扩展 (Stateless Proxy)" {
  component "Proxy 实例 1" as I1
  component "Proxy 实例 2" as I2
  component "Proxy 实例 N\n(HPA)" as IN

  note right of IN
    Proxy 无本地状态
    - API Key 状态存 Redis
    - 账单数据存 PostgreSQL
    - 实例间不通信
    可无限水平扩展
  end note
}

package "Redis 高可用" {
  component "Redis Primary" as RP
  component "Redis Replica 1" as RR1
  component "Redis Replica 2" as RR2
  RP --> RR1 : replication
  RP --> RR2 : replication

  note bottom
    Redis Cluster Mode 支持
    分片存储 TPM/RPM 计数
  end note
}

package "PostgreSQL 高可用" {
  database "Primary\n(Read/Write)" as PGP
  database "Standby\n(Hot Standby)" as PGS
  PGP --> PGS : streaming replication

  note bottom
    写操作 → Primary
    可配置 Read Replica
    分担查询压力
  end note
}

I1 --> RP
I2 --> RP
IN --> RP
I1 --> PGP
I2 --> PGP
IN --> PGP
@enduml
```
