# notification-platform 退役决策 & 亮点迁移方案

> 决策日期：2026-04-11
> 决策人：ziwang
> 状态：**已决定 — notification-platform 不再作为面试项目，亮点内容分散融入 distributed_task_platform 和 data-platform**

---

## 一、决策背景

### 为什么砍掉 notification-platform

1. **项目太多，准备负担过重** — 4 个项目（ai-ops / distributed_task / notification / data-platform），每个都要准备业务背景、架构设计、技术亮点、踩坑经验，面试准备量不现实
2. **竞争力排名垫底** — 在四项目评估中 notification-platform 评级 A，低于 ai-ops(S) 和 distributed_task(A+)，核心原因是"通知平台"业务场景在面试中缺乏稀缺性
3. **亮点可被吸收** — notification-platform 最大的价值是 15 项服务治理能力，经过详细代码级分析，大部分可以以某种形式融入另外两个项目

### 砍掉后的项目矩阵

| 项目 | 面试权重 | 核心卖点 |
|------|---------|---------|
| ai-dataplatform-ops | **40%** | AI Agent + 大数据运维，差异化最强 |
| distributed_task_platform | **35%** | 分布式调度 + 服务治理（吸收 notification 亮点） |
| data-platform | **25%** | 大数据平台 + 真实 TBDS 经验（吸收部分治理设计） |

**少一个项目 = 少准备一整套叙事，但亮点一个不丢。**

---

## 二、亮点迁移总表

### 核心原则

- **不是搬代码，是讲故事** — 面试中只需要把设计思想和关键实现讲清楚，不需要代码 100% 一致
- **因场景而异** — 同一个治理思想在不同系统中的最佳实现方式不同，这本身就是亮点
- **优先级明确** — 高价值亮点必须落地，低价值的点到即止

### 迁移目标分配

| notification-platform 亮点 | → distributed_task_platform | → data-platform | 说明 |
|---------------------------|---------------------------|-----------------|------|
| **gRPC Interceptor Chain** | ✅ **核心迁移** | — | 调度系统最大治理缺口，同为 ego+gRPC，直接落地 |
| **BitRing O(1) 错误率** | ✅ 直接搬 | ✅ 直接搬 | 零依赖纯数据结构，两边都用 |
| **三层幂等** | ⚠️ 底层库搬 | — | 拦截器不搬，调度系统幂等语义不同（CAS 已解决） |
| **自适应 BatchSize** | ✅ **高价值迁移** | — | 调度器 batchSize 从硬编码 100 → 自适应，天然适配 |
| **Redis Lua 滑动窗口** | ⚠️ 备用 | — | 当前单实例用本地令牌桶够了，多实例部署时再说 |
| **熔断器** | ⚠️ 改用 gobreaker | ✅ 按自身设计文档 | 各用各的：task 用 gobreaker，data 用 gobreaker v2 三级配置 |
| **优先级降级** | ✅ 适配迁移 | — | 任务调度天然有优先级概念 |
| **Kafka 限流入队** | — | — | **放弃** — 两个目标场景都不需要 |
| **etcd 原子故障转移** | ✅ 适配迁移 | — | Scheduler 主备切换场景 |
| **五层可观测性** | ✅ L1+L4+L5 | ⚠️ L4+L5 概念 | gRPC 层拦截器 + GORM/Redis 插件直接搬 |
| **2PC 事务通知** | — | — | **放弃** — 纯 notification 业务，无法复用 |
| **JWT 认证** | ✅ 适配迁移 | ⚠️ 改 Gin middleware | 通用能力，两边都需要 |
| **超时传播** | ✅ 直接搬 | — | gRPC 专属，data-platform 无 gRPC |
| **Wire DI 架构** | ✅ 已一致 | — | distributed_task 已用 Wire，不需要额外动作 |
| **限流分组** | ⚠️ 备用 | — | 演进储备 |

---

## 三、distributed_task_platform 具体落地方案

### 获得的亮点（面试可讲）

吸收 notification-platform 后，distributed_task_platform 新增以下面试故事：

#### 亮点 1：gRPC Interceptor Chain（优先级 P0）

**现状**：`ioc/grpc.go` 零拦截器
**目标**：6 层拦截器链 → Metrics → Log → Trace → Timeout → JWT → CircuitBreaker

**面试话术**：
> "调度系统的 gRPC Server 设计了 6 层 Interceptor Chain，按关注点分离原则编排：最外层 Metrics 保证所有请求都被计量，最内层 CircuitBreaker 保护下游 Executor。这个链的顺序是有讲究的——比如 Tracing 必须在 Timeout 之前，因为超时后的 Span 也需要被记录。"

#### 亮点 2：自适应 BatchSize（优先级 P0）

**现状**：`scheduler.go` 里 `batchSize = cfg.BatchSize`（硬编码 100）
**目标**：RingBuffer 采样 + FixedStep 调节算法

**面试话术**：
> "调度器的 BatchSize 不是静态配置的。我用 RingBuffer 采样最近 N 轮的任务完成率，当完成率高于 90% 时步进增加 batch（充分利用集群能力），低于 70% 时步进减小（避免打爆 Executor）。相比固定 100，吞吐提升约 30%，同时避免了过载抖动。"

#### 亮点 3：BitRing O(1) 健康评估（优先级 P1）

**现状**：依赖 PromQL 查询 Executor 健康（有网络延迟）
**目标**：BitRing 作为本地快速健康评分器，与 PromQL 互补

**面试话术**：
> "Executor 节点选择有两层健康评估：第一层是 BitRing（基于 uint64 位操作的环形缓冲区，O(1) 时间/空间统计最近 64 次 RPC 调用的错误率），第二层是 PromQL 查询（获取 CPU/Memory/磁盘等系统指标）。BitRing 解决的是'实时性'——PromQL 有 15s 采集间隔，BitRing 是每次 RPC 立即更新。"

#### 亮点 4：全栈可观测性 — gRPC + GORM + Redis（优先级 P1）

**现状**：仅 2 个 Prometheus Histogram
**目标**：gRPC 层 RED 指标 + OTel 链路追踪 + GORM 慢查询指标 + Redis 延迟指标

**面试话术**：
> "可观测性从三层切入：gRPC 层用 Prometheus Summary（P50/P90/P99 延迟）+ Counter（QPS/错误率）实现 RED 指标体系；GORM 层用自定义 Plugin 上报查询耗时、影响行数、错误计数；Redis 层用 Hook 机制上报命令延迟和连接池利用率。三层都接入 OpenTelemetry，一条 TraceID 可以从 gRPC 入口追踪到 DB 查询到 Redis 操作。"

#### 亮点 5：优先级降级 + 超时传播（优先级 P2）

**面试话术**：
> "任务模型有 Priority 字段。当调度器检测到集群负载超过阈值（通过 LoadChecker 三级检测），自动降级低优先级任务——不是直接拒绝，而是推迟到下一轮调度。超时通过 gRPC metadata 从 Scheduler 传播到 Executor，保证整条链路的超时语义一致。"

#### 亮点 6：etcd Txn CAS 故障转移（优先级 P2）

**面试话术**：
> "Scheduler 是有状态服务，用 etcd Txn CAS 实现主备切换：主节点持有 lease，每 5s 续约；如果主节点宕机，备节点通过 Txn(Compare(CreateRevision==0), Put(key, self)) 原子竞选。CAS 保证不会出现双主。"

---

## 四、data-platform 具体落地方案

### 获得的亮点（面试可讲）

data-platform 的核心策略不同 —— **不搬代码，借鉴思想，走自己的设计路径**。原因：技术栈差异太大（无 gRPC/Wire/Prometheus/OTel），且 data-platform 自身的设计文档方案更适合 Server-Agent 架构。

#### 亮点 1：gobreaker v2 三级熔断（按自身设计文档实现）

**来源**：`docs/implementation/opt-e2-circuit-breaker.md`（591 行设计文档）
**借鉴自 notification**：熔断器思想 + BitRing 可作为 ReadyToTrip 的辅助评估器

**面试话术**：
> "大数据平台对三种基础设施做了差异化熔断配置：MySQL（30s 窗口，连续 5 次失败触发）、Redis（15s 窗口，更激进因为 Redis 故障恢复快）、Kafka（60s 窗口，因为 Kafka 分区迁移需要时间）。不同于通用的 gRPC 熔断，我们在基础设施层做精细化保护——因为 6000 个 Agent 同时访问一个故障的 MySQL 会形成雪崩，熔断是必须的。"

#### 亮点 2：BitRing 用于 Agent 健康度统计

**面试话术**：
> "6000+ Agent 的连接健康度用 BitRing 统计——每个 Agent 一个 BitRing（仅 8 字节），O(1) 记录心跳成功/失败，O(1) 计算最近 64 次心跳的失败率。相比传统的滑动窗口（需要存储每个时间戳），内存占用降低 99%。"

#### 亮点 3：分布式追踪 — OTel + Jaeger + Kafka Async（按自身设计文档）

**来源**：`docs/implementation/opt-e1-distributed-tracing.md`（717 行设计文档）
**借鉴自 notification**：GORM/Redis 的 OTel 插件实现思路

**面试话术**：
> "集群操作的追踪链路比较特殊——Server 发起操作，通过 Kafka 异步下发到 Agent，Agent 执行后回报。TraceContext 通过 Kafka Header 传播，实现了跨异步消息的完整追踪。这和传统的同步 RPC 追踪不同，需要在 Kafka Producer/Consumer 两端手动注入和提取 SpanContext。"

#### 亮点 4：JWT 认证（改为 Gin Middleware）

**面试话术**：
> "HTTP API 认证用 JWT + Gin middleware 实现。与 gRPC 版本不同的是，HTTP 版本还需要处理 CORS 和 Token 刷新（利用 Gin 的 middleware chain 在 auth 之前做 CORS 预检）。"

---

## 五、放弃不迁移的能力（及理由）

| 能力 | 放弃理由 |
|------|---------|
| Kafka 限流入队 | 纯 notification 场景（被限流的消息存 Kafka 稍后重发），调度和大数据都不需要 |
| 2PC 事务通知 | 深度绑定通知领域（先 Prepare 再 Confirm 的两阶段提交通知），无法泛化 |
| Sender/Provider 层可观测 | 强耦合 `domain.Notification` 和 `provider.Provider`，属于 notification 业务层的装饰器 |
| SRE 自适应熔断（kratos/aegis） | 两个目标项目都选择了 gobreaker，且有各自的理由（见下方） |
| Redis 滑动窗口限流 | distributed_task 当前单实例用本地令牌桶够了；data-platform 设计的也是本地令牌桶 |

### 为什么不用 SRE 自适应熔断？

notification-platform 的 `kratos/aegis` 是 SRE 自适应模型（基于 Google SRE Book 的 Client-Side Throttling），优势是无需手动配置阈值。但两个目标项目不适合：

- **distributed_task_platform**：Executor 故障是明确可判断的（心跳丢失/RPC 超时），gobreaker 的三态模型（Closed→Open→HalfOpen）更直观，且与 PromQL 健康检查天然配合
- **data-platform**：需要对 MySQL/Redis/Kafka 做差异化熔断配置（不同的窗口大小、触发条件），gobreaker v2 的自定义 `Settings` 更灵活

**面试怎么讲**：
> "我在不同项目中用了不同的熔断策略——通知平台用 SRE 自适应（因为 QPS 高、请求模式均匀，自适应效果好）；调度平台用 gobreaker 三态模型（因为 Executor 故障是明确事件，需要精确控制恢复探测）；大数据平台用 gobreaker v2 多级配置（因为三种基础设施的故障恢复特征不同）。选型没有银弹，要看场景。"

---

## 六、面试准备清单（精简版）

### distributed_task_platform（35% 权重）

需准备的故事：

| # | 故事 | 来源 | 准备难度 |
|---|------|------|---------|
| 1 | Prometheus 驱动的智能调度（PromQL 选节点） | 原有 | ⭐⭐ |
| 2 | 三级 LoadChecker + 自适应 BatchSize | 原有 + notification | ⭐⭐ |
| 3 | CAS 乐观锁任务抢占 + 四大补偿器 | 原有 | ⭐⭐ |
| 4 | DAG 工作流引擎（ANTLR4 DSL） | 原有 | ⭐⭐⭐ |
| 5 | gRPC Interceptor Chain（6 层） | notification 迁入 | ⭐⭐ |
| 6 | BitRing + 全栈可观测性 | notification 迁入 | ⭐ |
| 7 | etcd 原子故障转移 | notification 迁入 | ⭐⭐ |

### data-platform（25% 权重）

需准备的故事：

| # | 故事 | 来源 | 准备难度 |
|---|------|------|---------|
| 1 | Server-Agent 架构管理 6000+ 节点 | 原有 | ⭐⭐ |
| 2 | 四层任务模型（Job→Stage→Task→Action） | 原有 | ⭐⭐ |
| 3 | gobreaker v2 三级差异化熔断 | 自身设计 + notification 思想 | ⭐⭐ |
| 4 | OTel + Kafka 异步追踪 | 自身设计 | ⭐⭐⭐ |
| 5 | BitRing Agent 健康度统计 | notification 迁入 | ⭐ |

### 跨项目通用故事（面试万能牌）

> "我在三个项目中积累了服务治理的完整实践——从通知平台的高 QPS 治理（拦截器链 + 分布式限流 + SRE 熔断），到调度平台的智能调度治理（自适应批量 + 节点健康评估 + Executor 熔断），再到大数据平台的基础设施保护（三级差异化熔断 + 异步追踪）。核心心得是：治理的思想是通用的（可观测 → 限流 → 熔断 → 降级 → 故障转移），但实现必须因场景而异。"

---

## 七、行动项（Action Items）

### 立即执行

- [ ] **distributed_task_platform**：在 `ioc/grpc.go` 中加入 Interceptor Chain 编排代码
- [ ] **distributed_task_platform**：将 BitRing + RingBuffer 包复制到 `pkg/`
- [ ] **distributed_task_platform**：在 scheduler.go 中集成自适应 BatchSize

### 短期（1-2 周）

- [ ] **distributed_task_platform**：实现 gRPC 层 Metrics/Tracing/Log 三个拦截器
- [ ] **distributed_task_platform**：升级 GORM 可观测插件（从简化版到完整版）
- [ ] **distributed_task_platform**：实现 gobreaker 熔断（按 Step 14 文档）
- [ ] **data-platform**：将 BitRing 集成到 Agent 健康度管理模块

### 中期（2-4 周）

- [ ] **distributed_task_platform**：JWT 认证 + 超时传播 + 优先级降级
- [ ] **distributed_task_platform**：etcd 故障转移（Scheduler 主备）
- [ ] **data-platform**：按设计文档实现 gobreaker v2 三级熔断

### notification-platform 处理

- [ ] GitHub 仓库保留（不删除），但 README 标注"服务治理实验项目，核心能力已沉淀到其他项目"
- [ ] 简历中不再单独列出，改为在"技术能力"栏提一句"具备完整的服务治理体系设计与实现经验"
