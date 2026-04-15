# 两个项目可新增高频场景设计清单

> 目标：不是凭空“加功能”，而是从 `data-platform` 和 `distributed_task_platform` 现有代码与架构出发，挑出**最自然、最像真实互联网业务、最能带出分布式事务 / 缓存与 DB 一致性 / 幂等 / 补偿**的话题。

---

## 一、结论先说

### 1.1 总判断

这两个项目都还能继续长，但方向不一样：

- **`data-platform`** 最适合往“**发布 / 下发 / 回传 / 缓存一致性 / 灰度回滚**”方向长
- **`distributed_task_platform`** 最适合往“**状态机 / 编排 / 配额 / 事件可靠性 / 补偿闭环**”方向长

如果目的是面试价值最大化，不建议平均用力，建议按下面优先级做：

### 1.2 最值得优先补的 6 个场景

#### data-platform
1. **配置灰度发布 + 一键回滚**
2. **批量巡检/修复任务的幂等下发 + Agent WAL 回传**
3. **多波次发布窗口 + Saga 补偿**

#### distributed_task_platform
1. **任务暂停 / 恢复 / 取消**
2. **调度配额 / 并发令牌 / 扣减回滚**
3. **分片父子聚合 + 局部重试 + 最终完成事件**

原因很直接：
- 这 6 个场景都**非常贴合现有链路**，不是硬塞
- 每个场景都能同时讲到**状态机、事务、幂等、补偿、缓存一致性、事件可靠性**里至少 3 个点
- 面试时能讲成“我在已有系统上继续做演进”，而不是“我另起炉灶写了一个 demo”

---

## 二、判断依据：这两个项目现在各自更像什么

### 2.1 data-platform 现在更像“集群任务下发 / 执行编排平台”

当前已经具备的主链路：

```go
// Stage -> Task
for _, p := range producers {
    task := &models.Task{
        TaskId: fmt.Sprintf("%s_%s_%d", stage.StageId, p.Code(), time.Now().UnixMilli()),
    }
    db.DB.Create(task)
    w.memStore.EnqueueTask(task)
}
```

```go
// Task -> Action
if len(actions) > 0 {
    db.DB.CreateInBatches(actions, 200)
}
db.DB.Model(task).Updates(map[string]interface{}{
    "state":      models.StateRunning,
    "action_num": len(actions),
})
```

```go
// DB -> Redis -> Agent
SELECT id, hostuuid FROM action WHERE state=0 LIMIT 2000
Redis ZADD hostuuid actionId
UPDATE action SET state=1
```

这条链天然适合扩成：
- 配置/脚本/命令发布
- 波次灰度
- 批量巡检与修复
- 节点侧 WAL / 回执幂等
- DB + Redis + Agent 三段式最终一致性

### 2.2 distributed_task_platform 现在更像“强状态机的调度编排内核”

当前已经具备的主链路：

```go
acquiredTask, err := s.taskAcquirer.Acquire(ctx, task.ID, task.Version, s.nodeID)
execution, err := s.execSvc.Create(ctx, domain.TaskExecution{...})
go func() {
    state, runErr := s.invoker.Run(ctx, execution)
    if updateErr := s.execSvc.UpdateState(ctx, state); updateErr != nil { ... }
}()
```

```go
if !isShardedTask {
    s.releaseTask(ctx, execution.Task)
    if _, err3 := s.taskSvc.UpdateNextTime(ctx, execution.Task.ID); err3 != nil {
        ...
    }
}
s.sendCompletedEvent(ctx, state, execution)
```

```go
if len(tasks) == 0 {
    err = p.producer.Produce(ctx, event.Event{
        Type: domain.PlanTaskType,
    })
}
```

再叠加现有补偿器体系：
- RetryCompensator
- RescheduleCompensator
- InterruptCompensator
- ShardingCompensator

它天然适合扩成：
- 暂停 / 恢复 / 取消
- DAG 变量快照与版本一致性
- 配额 / 并发令牌管理
- 分片聚合与局部失败补偿
- 可靠事件投递与幂等消费
- 调度触发去重

---

## 三、data-platform：最自然的 6 个新增场景

---

### 场景 1：配置灰度发布 + 一键回滚

#### 3.1.1 为什么这个场景最自然

`data-platform` 当前本质上已经有：
- Job
- Stage
- Task
- Action
- RedisActionLoader
- Agent Fetch / Report

这套结构和“配置发布平台”几乎一一对应：

- **Job** = 一次发布单
- **Stage** = 一个发布阶段（预检查 / 灰度 / 全量 / 验证）
- **Task** = 一个发布动作类型（下发配置、reload 服务、校验版本）
- **Action** = 对某台主机的具体执行命令

所以这不是新增一套系统，而是把现有调度链路换成一个**更互联网、更面试友好的业务壳**。

#### 3.1.2 涉及的现有文件

- `data-platform/tbds-control/internal/server/api/job_handler.go`
- `data-platform/tbds-control/internal/server/dispatcher/stage_worker.go`
- `data-platform/tbds-control/internal/server/dispatcher/task_worker.go`
- `data-platform/tbds-control/internal/server/dispatcher/task_center_worker.go`
- `data-platform/docs/implementation/step-4-redis-action-loader.md`
- `data-platform/docs/implementation/step-5-grpc-agent.md`

#### 3.1.3 当前链路里已经存在的锚点

当前代码已经支持：
- 创建 Job + Stage
- Stage 生成 Task
- Task 生成 Action
- Action 进 Redis
- Agent 拉取执行并回传
- TaskCenterWorker 聚合 Action 结果、推进 Stage

差的不是骨架，差的是：
- 业务语义
- 状态切分
- 回滚机制
- 版本与缓存一致性

#### 3.1.4 推荐怎么设计

新增核心实体：
- `config_release`
- `config_release_batch`
- `config_version`
- `config_release_outbox`

建议流程：

1. 创建发布单，生成 `release_id + idempotency_key`
2. 选择目标集群和节点范围
3. 按波次生成 Stage：
   - `PRECHECK`
   - `CANARY`
   - `FULL`
   - `VERIFY`
4. 每个 Stage 内生成 Task / Action
5. Agent 执行下发后上报版本
6. 验证失败时进入补偿：
   - 回滚到上一版本
   - 或停止后续波次
7. 发布单终态写 outbox，异步发通知 / 审计事件

#### 3.1.5 能引出的技术点

- 发布单创建幂等：`idempotency_key`
- 波次切换事务化：当前 Stage 完成 + 下一 Stage 激活原子化
- 节点配置版本缓存一致性：DB 为准，Redis / Agent 本地缓存失效广播
- 失败补偿：已执行节点回滚，未执行节点取消
- 审计事件可靠投递：Transactional Outbox

#### 3.1.6 推荐模式

- 本地事务 + Outbox
- Stage 状态迁移 CAS
- 配置版本号 + 延迟双删 / 广播失效
- 回滚 Saga（前向发布，反向补偿）

#### 3.1.7 面试价值

这个场景几乎是互联网高频题模板：
- 为什么不用 2PC
- 灰度发布怎么做
- 回滚怎么保证幂等
- 节点成功一半失败一半怎么办
- 配置缓存怎么和 DB 保持一致

这是 `data-platform` 最值得做的第一优先级场景。

---

### 场景 2：命令模板中心 + 多级缓存一致性

#### 3.2.1 为什么自然

现在 `TaskWorker` 是根据 `taskCode` 查 Producer，再由 Producer 产出 Action：

```go
p := producer.GetProducer(task.TaskCode)
actions, err := p.Produce(task, hosts)
```

这意味着项目天然已经存在“命令模板 / 动作模板”的概念，只是现在更像写死在代码里。把这层抽成“**模板中心**”非常自然。

#### 3.2.2 可以扩成什么业务

做一个“命令模板平台”：
- 模板有版本
- 模板支持参数渲染
- 支持灰度生效
- 热点模板走 Redis + 本地 LRU
- Agent 拉取时按版本执行

例如：
- 巡检脚本模板
- 配置下发模板
- 重启服务模板
- 诊断采集模板

#### 3.2.3 涉及文件

- `data-platform/tbds-control/internal/server/dispatcher/task_worker.go`
- `data-platform/tbds-control/internal/server/producer/*`
- `data-platform/docs/implementation/step-4-redis-action-loader.md`
- `data-platform/docs/implementation/step-5-grpc-agent.md`

#### 3.2.4 推荐设计

新增：
- `command_template`
- `command_template_version`
- `template_cache_invalidate_event`

缓存分层：
- DB：source of truth
- Redis：热点模板缓存
- Server 本地 LRU：降低 Redis 压力
- Agent 本地缓存：降低 gRPC 往返

更新流程：
1. DB 更新模板版本
2. 同事务写失效事件到 outbox
3. 异步广播到 Redis Pub/Sub 或 Kafka
4. Server / Agent 收到后删除本地缓存
5. 下一次读取回源并预热

#### 3.2.5 能引出的技术点

- 多级缓存一致性
- 缓存击穿 / 热点模板预热
- 延迟双删 vs 广播失效
- 模板版本号幂等加载
- 灰度模板与回滚

#### 3.2.6 面试价值

这个场景很适合回答：
- Redis 和本地缓存怎么一致
- 为什么要版本号
- 为什么广播失效比单纯 TTL 更可控
- 如何避免 Agent 执行旧模板

如果你想让 `data-platform` 更像真实平台产品，这个场景非常加分。

---

### 场景 3：批量巡检 / 批量修复任务的幂等下发 + Agent WAL 回传

#### 3.3.1 为什么自然

现在 `Action` 已经是一种“面向主机的执行单元”，而 `RedisActionLoader -> Agent Fetch -> Report` 已经天然是一条“命令投递总线”。

这最适合扩成：
- 批量巡检
- 批量修复
- 一键诊断
- 一键止损

#### 3.3.2 当前链路里的自然落点

从文档看，现有链路是：

```text
DB(state=Init) -> Redis Sorted Set -> Agent Fetch -> DB Update
```

以及：

```text
Agent fetchLoop -> actionQueue -> 执行 -> reportLoop -> CmdReportChannel
```

这其实已经非常接近一个“批量运维执行平台”。

#### 3.3.3 需要补的关键能力

1. **Action 下发幂等键**
   - `(task_id, hostuuid, command_hash)` 唯一约束
2. **Agent 本地 WAL**
   - Fetch 成功后先写本地 WAL，再执行业务
3. **Result 回传幂等**
   - `(action_id, attempt_no)` 做条件更新
4. **重复拉取防重执行**
   - Action 状态从 `Cached -> Executing` 用 CAS
5. **结果最终一致性**
   - Agent 崩溃后依据 WAL 恢复待上报结果

#### 3.3.4 推荐设计

Action 状态扩展为：
- `Init`
- `Cached`
- `Fetched`
- `Executing`
- `Reported`
- `Success / Failed / Timeout`

Agent 本地增加：
- `action_wal.log`
- `report_retry_queue`

服务端增加：
- `last_report_seq`
- `action_attempt`

#### 3.3.5 能引出的技术点

- at-least-once 下发
- 消费端幂等
- 本地 WAL 恢复
- DB / Redis / Agent 三段式一致性
- 失败重放与去重

#### 3.3.6 面试价值

这个场景很适合讲“为什么没有 exactly-once，也能做出业务上的 exactly-once 语义”：
- 服务端幂等
- Agent WAL
- CAS 状态迁移
- 重试安全

这是 `data-platform` 第二优先级非常高的场景。

---

### 场景 4：多波次发布窗口 + Saga 补偿

#### 3.4.1 为什么自然

`TaskCenterWorker.completeStage()` 已经在做“当前阶段完成 -> 激活下一个阶段”：

```go
db.DB.Model(stage).Updates(map[string]interface{}{
    "state":    models.StateSuccess,
    "progress": 100.0,
})

if stage.NextStageId != "" {
    db.DB.Model(&models.Stage{}).
        Where("stage_id = ?", stage.NextStageId).
        Update("state", models.StateRunning)
}
```

这条链天生适合扩成“波次发布”。

#### 3.4.2 可以长成什么业务

例如一个 1000 节点集群发布：
- 第 1 波：5 台 canary
- 第 2 波：50 台小流量
- 第 3 波：200 台半量
- 第 4 波：剩余全量

每个波次结束都要：
- 检查成功率
- 检查关键指标
- 决定继续 / 暂停 / 回滚

#### 3.4.3 推荐设计

Stage 不再只是线性流程，而是带策略：
- `batch_percent`
- `batch_size`
- `success_threshold`
- `error_threshold`
- `rollback_policy`

补偿逻辑：
- 某波次失败率超阈值：停止下一波
- 如果已变更节点超过阈值：触发反向回滚 Stage
- 回滚也是一套 Task / Action，只是模板相反

#### 3.4.4 能引出的技术点

- Saga 模式
- 阶段切换原子性
- 阈值驱动自动回滚
- 补偿链路幂等
- 手工确认 + 自动恢复

#### 3.4.5 面试价值

这类场景非常像真实发布系统，能把 `data-platform` 从“任务调度 demo”拉到“发布平台 / 运维平台”层次。

---

### 场景 5：节点上下线 / 拓扑变更后的缓存预热与孤儿 Action 回补

#### 3.5.1 为什么自然

`TaskWorker` 当前依赖在线主机列表：

```go
var hosts []models.Host
if err := db.DB.Where("cluster_id = ? AND status = ?",
    task.ClusterId, models.HostOnline).Find(&hosts).Error; err != nil {
    return err
}
```

而 Redis 中的 Action 又是按 `hostuuid` 下发的。节点上下线、主机重装、hostuuid 变更、Redis 重启，这些都天然会引发一致性问题。

#### 3.5.2 可以扩成什么业务

- 节点新加入集群时自动预热任务
- 节点长时间离线时回收未执行 Action
- 节点更换 hostuuid 时迁移待执行队列
- Redis 丢数据后按 DB 状态重建 Redis 队列

#### 3.5.3 推荐设计

新增后台补偿器：
- `OrphanActionCompensator`
- `HostQueueRebuilder`
- `ClusterWarmupPlanner`

核心规则：
- `state=Cached` 但 Redis 无记录 -> 重建入队
- `state=Executing` 但节点已离线超过阈值 -> 标记超时 / 允许重派
- 节点重新注册 -> 触发缓存预热和未完成 Action 补拉

#### 3.5.4 能引出的技术点

- Redis / DB 队列一致性
- 孤儿任务识别
- 主机级补偿
- 热点节点预热
- 节点身份变更幂等迁移

#### 3.5.5 面试价值

这是典型“线上环境一定会遇到”的问题。比单纯讲 CRUD 更像生产系统。

---

### 场景 6：发布结果聚合看板 + 缓存读模型

#### 3.6.1 为什么自然

`TaskCenterWorker.checkTaskActions()` 本来就在聚合 Action 状态：

```go
if anyFailed {
    db.DB.Model(task).Updates(map[string]interface{}{
        "state":   models.StateFailed,
    })
}
if allDone {
    db.DB.Model(task).Updates(map[string]interface{}{
        "state":    models.StateSuccess,
        "progress": 100.0,
    })
}
```

这说明项目已经有“聚合视图”的基础，只是现在是实时查 DB，没有独立读模型。

#### 3.6.2 可以扩成什么业务

做一个发布 / 巡检大盘：
- Job 级成功率
- Stage 级耗时
- 集群维度失败率
- 主机维度 Top N 异常
- Action 执行热力分布

#### 3.6.3 推荐设计

引入只读聚合表或缓存：
- `job_dashboard_snapshot`
- `cluster_action_summary`
- `host_failure_rank`

更新方式：
- Task / Stage / Job 状态变更时写 outbox
- 聚合消费者异步更新读模型
- 热点看板缓存到 Redis

#### 3.6.4 能引出的技术点

- CQRS 读写分离
- 异步聚合
- 看板缓存一致性
- TopN / 排行榜缓存
- 热点 Key 预热

#### 3.6.5 面试价值

这个场景更偏“平台产品化”，适合补充业务完整度，但优先级略低于前 5 个场景。

---

## 四、distributed_task_platform：最自然的 6 个新增场景

---

### 场景 1：任务暂停 / 恢复 / 取消

#### 4.1.1 为什么这是第一优先级

调度平台没有 Pause / Resume / Cancel，面试官很容易追问“那线上任务失控怎么办”。

而这个项目现在已经有：
- 任务抢占
- 执行记录状态机
- 中断补偿器
- Release / UpdateNextTime

所以新增这套能力非常自然。

#### 4.1.2 涉及文件

- `distributed_task_platform/internal/service/task/execution_service.go`
- `distributed_task_platform/internal/service/runner/normal_task_runner.go`
- `distributed_task_platform/internal/repository/dao/task.go`
- `distributed_task_platform/docs/12-补偿器功能文档.md`

#### 4.1.3 当前代码里的自然落点

终态处理当前已经做了：

```go
if !isShardedTask {
    s.releaseTask(ctx, execution.Task)
    if _, err3 := s.taskSvc.UpdateNextTime(ctx, execution.Task.ID); err3 != nil {
        ...
    }
}
s.sendCompletedEvent(ctx, state, execution)
```

也有超时中断补偿器：
- `RUNNING + deadline <= now -> Interrupt`

所以 Pause / Cancel 的差别，本质只是：
- 状态机新增状态
- 中断请求新增原因与语义
- 补偿器和调度器识别这些状态

#### 4.1.4 推荐设计

新增任务状态：
- `PAUSING`
- `PAUSED`
- `CANCELLING`
- `CANCELLED`

流程：
1. 用户发起暂停
2. DB CAS 更新 `RUNNING -> PAUSING`
3. 写 outbox 通知执行节点发送 interrupt/pause
4. 执行节点回 ACK
5. ACK 成功后切到 `PAUSED`
6. 恢复时 `PAUSED -> IDLE / READY`
7. 取消时 `CANCELLING -> CANCELLED`，释放配额和锁

#### 4.1.5 能引出的技术点

- 状态机扩展
- 中断命令可靠投递
- 取消幂等
- 任务锁释放一致性
- 恢复时 next_time 重新计算

#### 4.1.6 面试价值

这是调度系统非常高频的追问项。做出来以后，系统成熟度会明显上一个档次。

---

### 场景 2：DAG 变量快照 + 版本化缓存一致性

#### 4.2.1 为什么自然

`PlanTaskRunner` 已经在做 DAG 推进：

```go
plan, err := p.planService.GetPlan(ctx, task.ID)
rootTasks := plan.RootTask()
```

```go
plan, err := p.planService.GetPlan(ctx, task.PlanID)
tasks := planTask.NextStep()
```

只要有 DAG，就一定会被问：
- DAG 运行时如果定义被修改怎么办？
- 变量是在运行时实时读取，还是启动时快照？
- 多个节点看到的 DAG 版本不一致怎么办？

#### 4.2.2 可以扩成什么业务

- 工作流模板平台
- 节点级变量渲染
- 运行时上下文快照
- DAG 模板版本管理

#### 4.2.3 推荐设计

新增：
- `plan_version`
- `plan_execution_snapshot`
- `plan_runtime_context`

规则：
- Plan 启动时把 DAG 定义和关键变量做快照
- 后续节点执行一律读取 snapshot，不直接读在线配置
- 在线 DAG 更新只影响新执行，不影响运行中的执行
- 热点 DAG 走 Redis + 本地缓存，但缓存 key 必须带 version

#### 4.2.4 能引出的技术点

- 配置 / 缓存版本一致性
- snapshot 隔离
- 缓存失效广播
- 并发更新冲突控制
- DAG 运行时可重放性

#### 4.2.5 面试价值

这个场景能把你从“会写 DAG 调度”拉到“理解工作流引擎一致性语义”的层面。

---

### 场景 3：调度配额 / 并发令牌 / 扣减回滚

#### 4.3.1 为什么自然

这个项目已经有：
- 任务抢占
- 节点负载检查
- 数据库负载检查
- 集群负载检查
- CompositeChecker

但还缺一个互联网高频能力：**配额与令牌控制**。

这非常适合加在调度前面。

#### 4.3.2 可以扩成什么业务

例如：
- 一个租户同一时间最多跑 100 个任务
- 一个任务组最多占用 20 个 executor
- 一个 DAG 同时最多推进 3 个分支
- 高优先级任务可以抢占低优先级配额

#### 4.3.3 推荐设计

新增：
- `quota_bucket`
- `quota_token_lease`
- `quota_change_log`

调度流程：
1. 任务准备运行时先尝试扣减 quota token
2. 扣减成功才进入抢占 / 创建 execution
3. 执行失败或取消时归还 token
4. 节点宕机导致 token 泄漏时，补偿器扫描 lease 超时归还

#### 4.3.4 涉及文件

- `distributed_task_platform/internal/service/runner/normal_task_runner.go`
- `distributed_task_platform/internal/service/task/execution_service.go`
- `distributed_task_platform/pkg/loadchecker/*.go`
- `distributed_task_platform/docs/12-补偿器功能文档.md`

#### 4.3.5 能引出的技术点

- Redis/DB 配额一致性
- lease 超时回收
- 扣减与执行创建原子性
- 高优先级抢占
- 流量削峰和背压

#### 4.3.6 推荐模式

- 快路径：Redis token bucket / semaphore
- 慢路径：DB ledger 做最终对账
- 令牌 lease + 补偿回收
- 扣减成功后写 execution/outbox

#### 4.3.7 面试价值

这是很典型的“高并发平台如何做配额和公平性控制”的题，面试收益很高。

---

### 场景 4：分片父子聚合 + 局部重试 + 最终完成事件

#### 4.4.1 为什么自然

`CreateShardingChildren` 已经在做父子执行记录创建：

```go
created, err := s.repo.CreateShardingParent(ctx, parent)
if err != nil {
    return nil, err
}
return s.repo.CreateShardingChildren(ctx, created, executorNodeIDs, scheduleParams)
```

同时，补偿器文档里也已经定义了：
- `ShardingCompensator`
- 聚合子任务状态
- 更新父任务状态

这说明“分片聚合”本来就是系统重点能力，只是还能再往前走一步。

#### 4.4.2 现在还缺什么

当前聚合更偏“全成功 / 有失败 / 未完成”三段式，互联网场景下还常见：
- 允许 1% 失败但整体成功
- 局部失败自动重试
- 仅重跑失败分片
- 最终完成事件可靠投递

#### 4.4.3 推荐设计

新增能力：
- 父任务聚合策略：`ALL_SUCCESS / THRESHOLD_SUCCESS / PARTIAL_ACCEPT`
- 子分片 retry policy
- 聚合完成 outbox event
- 分片结果汇总快照

流程：
1. 子任务完成后更新子状态
2. 聚合器读取所有子状态
3. 根据聚合策略判断父状态
4. 父状态更新 + task release + next_time + outbox 写入放同事务
5. Relay 异步投递完成事件

#### 4.4.4 能引出的技术点

- 父子状态聚合一致性
- 局部重试 vs 全量重跑
- 阈值成功策略
- 分片完成事件去重
- Cursor 扫描避免漏补偿

#### 4.4.5 面试价值

这个场景能把“我支持分片”升级成“我理解分片任务在生产场景里的完整闭环”。

---

### 场景 5：事件通知 / Webhook / 回调中心

#### 4.5.1 为什么自然

`distributed_task_platform` 已经有：
- 任务完成事件
- Kafka 投递
- Outbox 设计文档

那继续长出一个“通知 / 回调中心”几乎顺手就能做。

#### 4.5.2 可以扩成什么业务

- 任务成功 / 失败回调业务系统
- DAG 完成后触发下游系统
- SLA 失败后自动通知群机器人 / 工单系统
- 失败重试结果通知

#### 4.5.3 推荐设计

新增：
- `callback_subscription`
- `callback_delivery`
- `callback_outbox`

流程：
1. 任务终态写 `task_completed` outbox
2. 通知中心消费后查询订阅关系
3. 生成 webhook / MQ / 短信 / 站内信等 delivery
4. delivery 自己也是重试 + 幂等的

#### 4.5.4 能引出的技术点

- 事件可靠投递
- Webhook 重试退避
- 回调幂等签名
- 死信队列
- 订阅读模型缓存

#### 4.5.5 面试价值

这会让调度系统更像“平台中台”而不是单机执行器，业务感更强。

---

### 场景 6：调度触发去重（Exactly-once-ish Trigger）

#### 4.6.1 为什么自然

调度系统一个很高频的问题是：
- 同一个 cron 时间点，任务会不会被触发两次？
- leader 切换或重试时，会不会重复创建 execution？

当前系统已经有：
- Acquire CAS
- Execution Create
- UpdateNextTime

只差把“触发层幂等”补上。

#### 4.6.2 推荐设计

新增唯一键：
- `(task_id, schedule_time_bucket)`
- 或 `(task_id, logical_trigger_id)`

规则：
- 每次调度前先计算逻辑触发 ID
- 创建 execution 时带上该 trigger_id
- DB 唯一索引拦截重复创建
- 如果重复触发，直接返回已有 execution

#### 4.6.3 涉及文件

- `distributed_task_platform/internal/service/runner/normal_task_runner.go`
- `distributed_task_platform/internal/repository/dao/task.go`
- `distributed_task_platform/internal/repository/dao/task_execution.go`

#### 4.6.4 能引出的技术点

- 调度层幂等
- leader 切换安全
- 重试安全
- 唯一键防重
- 时间窗口触发语义

#### 4.6.5 面试价值

这属于调度系统非常典型、非常专业的追问项。补上后，系统可信度会明显上升。

---

## 五、两个项目分别最适合承载哪些高频技术点

| 技术点 | 更适合 data-platform | 更适合 distributed_task_platform | 原因 |
|---|---|---|---|
| 配置发布 / 回滚 | 是 | 一般 | `data-platform` 已有 Job/Stage/Task/Action 链路 |
| 缓存与 DB 一致性 | 很适合 | 适合 | `data-platform` 有 RedisActionLoader + Agent 缓存链 |
| 幂等下发 / 去重执行 | 很适合 | 适合 | `Action` 粒度天然适合做幂等键 |
| DAG 快照 / 编排一致性 | 一般 | 很适合 | `distributed_task_platform` 已有 Plan/DAG |
| 配额 / 并发令牌 | 一般 | 很适合 | 已有 loadchecker、acquire、execution 模型 |
| 暂停 / 恢复 / 取消 | 一般 | 很适合 | 状态机和补偿器已具备基础 |
| 分片聚合 / 局部重试 | 一般 | 很适合 | 已有 sharding parent/child 和补偿器 |
| Outbox / 事件可靠性 | 适合 | 很适合 | `distributed_task_platform` 文档和链路更完整 |
| Agent WAL | 很适合 | 一般 | `data-platform` 的 Agent 拉取/上报模型更直接 |
| 灰度波次 / Saga | 很适合 | 适合 | `data-platform` 的 Stage 模型更容易讲 |

---

## 六、如果只做 2 周，建议怎么排优先级

### 6.1 第一阶段：先把最有面试价值的闭环做出来

#### data-platform
1. 配置灰度发布 + 一键回滚
2. 批量巡检/修复任务的幂等下发 + Agent WAL
3. 多波次发布窗口 + Saga 补偿

#### distributed_task_platform
1. 任务暂停 / 恢复 / 取消
2. 调度配额 / 并发令牌 / 扣减回滚
3. 分片父子聚合 + 局部重试 + outbox 完成事件

### 6.2 第二阶段：再补“平台感”和“产品感”

#### data-platform
- 模板中心 + 多级缓存一致性
- 节点上下线 / 缓存预热 / orphan action 回补
- 发布结果聚合看板

#### distributed_task_platform
- DAG 变量快照 + 版本化缓存
- 事件通知 / Webhook 中心
- 调度触发去重

---

## 七、我给你的明确建议

### 7.1 data-platform 不要再往“泛调度引擎”方向补了

因为它现在最强的不是“通用调度”，而是：
- 对集群主机下发动作
- 通过 Redis + Agent 做实际执行
- 有 Stage/Task/Action 天然适合表达灰度、波次、回滚

所以最优路线是把它包装成：

> **面向集群配置发布、巡检修复、批量变更执行的平台**

这个定位比“我也做了一个调度系统”更有辨识度。

### 7.2 distributed_task_platform 要继续往“调度内核成熟度”方向补

它已经有：
- Acquire/Release
- execution state machine
- DAG
- sharding
- compensator
- outbox 文档

所以最优路线不是加更多业务壳，而是把这些基础能力补成熟：
- pause/resume/cancel
- quota/token
- sharding partial retry
- trigger dedup
- event reliability

这会让它更像一个真正的调度平台内核。

### 7.3 最佳组合打法

如果面试只允许重点讲两个项目，我建议这么讲：

- **`distributed_task_platform`**：讲“调度内核成熟度”
  - 状态机
  - 抢占
  - DAG
  - 分片
  - 补偿
  - 配额
  - 可靠事件

- **`data-platform`**：讲“集群发布与执行平台”
  - 灰度发布
  - 批量巡检修复
  - Agent WAL
  - DB/Redis/Agent 一致性
  - 回滚 Saga
  - 模板缓存

这两个项目的叙事会互补，而不会打架。

---

## 八、可直接转成后续实施任务的版本

### 8.1 data-platform 推荐实施顺序

#### P0
- 配置灰度发布 + 一键回滚
- 批量巡检/修复任务的幂等下发 + Agent WAL
- 多波次发布窗口 + Saga 补偿

#### P1
- 模板中心 + 多级缓存一致性
- 节点上下线 / orphan action 回补

#### P2
- 发布结果聚合看板 + CQRS 读模型

### 8.2 distributed_task_platform 推荐实施顺序

#### P0
- 任务暂停 / 恢复 / 取消
- 调度配额 / 并发令牌 / 扣减回滚
- 分片父子聚合 + 局部重试 + outbox 完成事件

#### P1
- DAG 变量快照 + 版本化缓存一致性
- 调度触发去重

#### P2
- 事件通知 / Webhook / 回调中心

---

## 九、最后一句判断

如果你问我：**这两个项目里，哪些新增场景最“像真的”、最值得投入？**

我的答案很明确：

- `data-platform` 就做成 **发布 / 巡检 / 修复 / 回滚平台**
- `distributed_task_platform` 就做成 **强状态机 + 强补偿 + 强编排的调度内核**

别把两个项目都做成“万能调度 demo”。那样会稀释亮点。

把一个做成**业务执行平台**，一个做成**调度内核平台**，面试叙事最强。
