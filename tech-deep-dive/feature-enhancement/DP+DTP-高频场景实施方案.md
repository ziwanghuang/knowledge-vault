# 两个项目可新增高频场景实施方案

> 目标：把上一份《两个项目可新增高频场景设计清单》继续往前推进一层，不再停留在“可以做什么”，而是直接回答“应该怎么做、先改哪里、状态机怎么扩、表怎么加、幂等和补偿怎么兜”。

---

## 一、文档范围

这份实施方案只覆盖最值得优先落地的 6 个 P0 场景：

### 1.1 data-platform

1. 配置灰度发布 + 一键回滚
2. 批量巡检 / 修复任务的幂等下发 + Agent WAL
3. 多波次发布窗口 + Saga 补偿

### 1.2 distributed_task_platform

1. 任务暂停 / 恢复 / 取消
2. 调度配额 / 并发令牌 / 扣减回滚
3. 分片父子聚合 + 局部重试 + 最终完成事件

不展开的部分：
- UI 页面细节
- 权限系统
- 审批流
- 完整监控面板实现
- 非核心辅助优化项

原因很简单：现在最缺的不是“功能列表更长”，而是“拿得出 6 个能直接讲状态机、事务、幂等、一致性、补偿的硬场景”。

---

## 二、总实施原则

## 2.1 两个项目的定位不要混

### data-platform
定位成：

> 面向集群的配置发布、巡检修复、批量执行平台

核心关键词：
- Job / Stage / Task / Action
- DB -> Redis -> Agent
- 发布波次
- 回滚
- Agent WAL
- DB / Redis / Agent 最终一致性

### distributed_task_platform
定位成：

> 强状态机、强补偿、强编排的调度内核

核心关键词：
- Acquire / Release
- Execution State Machine
- Plan / DAG
- Sharding
- Quota / Token
- Complete Event / Outbox

## 2.2 统一的工程原则

不管哪个场景，统一采用下面几条规则：

1. **DB 是 source of truth**
   - Redis 只做缓存、队列、快速令牌
   - Agent 本地只做 WAL / 临时缓存

2. **一个逻辑状态切换，尽量收敛到一次本地事务**
   - 例如“当前阶段完成 + 激活下一阶段”
   - 例如“父任务聚合完成 + release + updateNextTime + 写 outbox”

3. **所有跨进程副作用都走 outbox / event**
   - 不在本地事务提交前直接调远端
   - 不指望 2PC

4. **所有重试路径都先定义幂等键**
   - 创建幂等
   - 消费幂等
   - 回滚幂等
   - 回调幂等

5. **所有补偿都要可重入**
   - 重跑一次不应该把状态打乱
   - 失败补偿与人工重试不能互相污染

---

## 三、data-platform 实施方案

---

## 3.1 场景一：配置灰度发布 + 一键回滚

## 3.1.1 现有锚点

当前 `data-platform` 已经具备完整骨架：

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
// Stage complete -> activate next stage
func (w *TaskCenterWorker) completeStage(stage *models.Stage) {
    db.DB.Model(stage).Updates(map[string]interface{}{
        "state": models.StateSuccess,
    })
    if stage.NextStageId != "" {
        db.DB.Model(&models.Stage{}).
            Where("stage_id = ?", stage.NextStageId).
            Update("state", models.StateRunning)
    }
}
```

所以这个场景不是另起系统，而是对现有 Job / Stage / Task / Action 加业务语义。

## 3.1.2 要补的业务能力

把当前调度链路包装为“配置发布单”：

- 发布单创建
- 波次划分
- 节点配置下发
- 版本校验
- 失败后停止扩散
- 已变更节点一键回滚
- 发布完成写审计事件

## 3.1.3 目标状态机

### 发布单状态
- `CREATED`
- `APPROVED`
- `RUNNING`
- `PAUSED`
- `ROLLING_BACK`
- `SUCCESS`
- `FAILED`
- `ROLLED_BACK`

### 波次状态
- `PENDING`
- `RUNNING`
- `SUCCESS`
- `FAILED`
- `ROLLING_BACK`
- `ROLLED_BACK`

### 节点动作状态
沿用现有 `Action` 状态机，但建议扩展为：
- `Init`
- `Cached`
- `Fetched`
- `Executing`
- `Reported`
- `Success`
- `Failed`
- `Timeout`
- `Cancelled`

## 3.1.4 数据模型

建议新增表：

### 1）config_release
字段建议：
- `id`
- `release_id`：业务发布单号
- `cluster_id`
- `app_name`
- `target_version`
- `source_version`
- `status`
- `idempotency_key`
- `operator`
- `rollback_policy`
- `created_at`
- `updated_at`

约束：
- 唯一索引：`uk_release_id`
- 唯一索引：`uk_idempotency_key`

### 2）config_release_batch
字段建议：
- `id`
- `release_id`
- `batch_no`
- `batch_type`：`PRECHECK/CANARY/FULL/VERIFY`
- `batch_percent`
- `host_count`
- `success_threshold`
- `error_threshold`
- `status`
- `rollback_status`

### 3）config_version
字段建议：
- `id`
- `app_name`
- `version`
- `content_hash`
- `storage_path`
- `status`
- `published_at`

### 4）release_outbox
字段建议：
- `id`
- `aggregate_type`
- `aggregate_id`
- `event_type`
- `payload`
- `status`
- `retry_count`
- `next_retry_time`

## 3.1.5 接口设计

建议新增：

- `POST /api/releases`
  - 创建发布单
- `POST /api/releases/{releaseId}/approve`
  - 审批通过并开始执行
- `POST /api/releases/{releaseId}/pause`
  - 暂停后续波次
- `POST /api/releases/{releaseId}/resume`
  - 继续执行
- `POST /api/releases/{releaseId}/rollback`
  - 一键回滚
- `GET /api/releases/{releaseId}`
  - 查看发布详情
- `GET /api/releases/{releaseId}/batches`
  - 查看各波次状态

## 3.1.6 核心事务边界

### 事务 A：创建发布单
同一事务内完成：
- 插入 `config_release`
- 插入首批 `config_release_batch`
- 创建对应 Job / Stage 元数据
- 写 `release_created` outbox

### 事务 B：波次完成切换
同一事务内完成：
- 当前 batch / stage 标记 success
- 判断阈值是否满足
- 激活下一 batch / stage 或标记发布单完成
- 写 `release_batch_completed` / `release_completed` outbox

### 事务 C：触发回滚
同一事务内完成：
- `config_release.status -> ROLLING_BACK`
- 生成 rollback Stage / Task / Action
- 写 `release_rollback_started` outbox

## 3.1.7 幂等与一致性设计

### 创建幂等
- 客户端传 `idempotency_key`
- `config_release.uk_idempotency_key` 防重
- 若重复请求，直接返回已有发布单

### 阶段推进幂等
- `completeStage` 改成 CAS 更新
- 只有 `state=RUNNING` 的 stage 才允许切到 `SUCCESS`
- 避免重复扫描导致重复激活下一阶段

### 配置缓存一致性
建议采用：
- DB 记录当前版本
- Redis 缓存版本元信息
- Agent 本地缓存具体配置内容

更新顺序：
1. DB 提交新版本
2. 同事务写 outbox 失效事件
3. relay 投递失效通知
4. Server / Agent 删除本地缓存
5. 下一次拉取回源加载

不是强一致，但业务上足够稳，而且面试上能讲明白“为什么不用 2PC”。

## 3.1.8 补偿策略

### 场景一：部分节点成功，验证失败
- 已成功节点：下发回滚 Action
- 未执行节点：直接取消待执行 Action
- 当前 batch 标记 `FAILED`
- 发布单标记 `ROLLING_BACK`

### 场景二：回滚任务也失败
- 标记 `PARTIAL_ROLLBACK_FAILED`
- 进入人工介入队列
- 记录失败节点列表，支持按节点重试回滚

### 场景三：Redis 队列丢失
- 根据 DB 中 `Action.state=Cached/Fetched/Executing` 重建 Redis 待执行集合

## 3.1.9 代码改动点

### 修改现有文件
- `data-platform/tbds-control/internal/server/dispatcher/stage_worker.go`
  - 支持按发布波次生成 Task
  - `TaskId` 改为更稳定的确定性 ID，而不是纯时间戳
- `data-platform/tbds-control/internal/server/dispatcher/task_worker.go`
  - 生成下发配置 / 验证版本 / 回滚配置等 Action
  - 落库与状态推进尽量收敛到事务
- `data-platform/tbds-control/internal/server/dispatcher/task_center_worker.go`
  - `completeStage` 改为事务化推进
  - 支持阈值判断、暂停、回滚分支
- `data-platform/tbds-control/internal/server/api/job_handler.go`
  - 增加发布单相关接口

### 建议新增文件
- `internal/service/release_service.go`
- `internal/repository/release_repo.go`
- `internal/repository/outbox_repo.go`
- `internal/server/api/release_handler.go`
- `internal/dispatcher/release_relay.go`

## 3.1.10 实施顺序

### 第一步
先只做：
- 创建发布单
- canary / full 两波次
- 失败后停止后续波次

### 第二步
再补：
- 一键回滚
- 配置版本校验
- 发布 outbox 事件

### 第三步
最后补：
- 暂停 / 恢复
- 指标阈值自动判断
- 人工确认继续

---

## 3.2 场景二：批量巡检 / 修复任务的幂等下发 + Agent WAL

## 3.2.1 现有锚点

当前链路已经很接近一条批量运维总线：

```text
DB(state=Init) -> Redis Sorted Set -> Agent Fetch -> DB Update
```

`TaskWorker` 当前生成 Action：

```go
actions, err := p.Produce(task, hosts)
if len(actions) > 0 {
    db.DB.CreateInBatches(actions, 200)
}
```

真正欠缺的不是“能不能下发”，而是：
- 重复下发怎么办
- Agent 拉到命令后崩溃怎么办
- 结果重复上报怎么办

## 3.2.2 目标能力

把它补成可讲的可靠执行链：

- 服务端幂等生成 Action
- Redis 投递允许至少一次
- Agent Fetch 后本地 WAL 落盘
- Agent 执行结果可重试上报
- 服务端消费结果幂等
- Agent 崩溃恢复后自动补报

## 3.2.3 状态机扩展

建议 `Action` 状态变为：
- `Init`
- `Cached`
- `Fetched`
- `Executing`
- `Reported`
- `Success`
- `Failed`
- `Timeout`
- `Cancelled`

状态迁移规则：
- `Init -> Cached`：装载进 Redis
- `Cached -> Fetched`：Agent 成功取到并落 WAL
- `Fetched -> Executing`：Agent 真正开始执行
- `Executing -> Reported`：Agent 已把终态写入重试队列
- `Reported -> Success/Failed/Timeout`：服务端确认入库成功

## 3.2.4 数据结构变更

### 服务端建议新增字段
在 `action` 表增加：
- `biz_key`：如 `(task_id, hostuuid, command_hash)` 计算结果
- `attempt_no`
- `fetch_token`
- `last_report_seq`
- `report_status`
- `report_time`

索引建议：
- 唯一索引：`uk_biz_key`
- 唯一索引：`uk_action_attempt` = `(id, attempt_no)`

### Agent 侧建议新增
- `action_wal.log`
- `report_retry_queue`
- `dedup_index`

`action_wal.log` 建议记录：
- `action_id`
- `fetch_token`
- `command_hash`
- `attempt_no`
- `status`
- `result_digest`
- `last_update_time`

## 3.2.5 核心协议变化

### Fetch 响应建议返回
- `action_id`
- `fetch_token`
- `attempt_no`
- `command`
- `command_hash`
- `deadline`

### Report 请求建议带上
- `action_id`
- `attempt_no`
- `fetch_token`
- `result_status`
- `stdout_digest`
- `stderr_digest`
- `report_seq`

## 3.2.6 幂等规则

### 服务端下发幂等
- Action 生成前计算 `biz_key`
- 如果已存在相同 `biz_key` 且仍有效，则直接复用
- 避免同一批巡检任务重复生成命令

### Fetch 幂等
- 只有 `state=Cached` 的 Action 可以通过 CAS 切到 `Fetched`
- Redis 重复投递时，CAS 失败直接跳过

### Report 幂等
- 服务端以 `(action_id, attempt_no, report_seq)` 做条件更新
- 小于等于 `last_report_seq` 的上报直接丢弃

### WAL 恢复幂等
- Agent 重启后扫描 WAL
- 对 `status=Executing/Reported` 的记录重新补报
- 服务端因幂等条件不会重复写终态

## 3.2.7 补偿策略

### Redis 已投递但 Agent 未真正执行
- WAL 中只有 `Fetched` 没有 `Executing`
- 超时后允许回收并重新投递

### Agent 已执行成功但上报前崩溃
- WAL 保留 `Executing -> ReportedPending`
- 启动恢复线程补报

### 服务端收到重复上报
- 基于 `last_report_seq` 丢弃重复包

### 长时间没有心跳 / 结果
- 标记 `Timeout`
- 进入人工重试或自动重派策略

## 3.2.8 代码改动点

### 修改现有文件
- `data-platform/tbds-control/internal/server/dispatcher/task_worker.go`
  - Action 落库时计算 `biz_key`
- `data-platform/docs/implementation/step-4-redis-action-loader.md`
  - 扩充为 `Init -> Cached -> Fetched`
- `data-platform/docs/implementation/step-5-grpc-agent.md`
  - 增加 WAL、retry queue、幂等 report 协议

### 建议新增文件
- `internal/agent/wal.go`
- `internal/agent/report_retry.go`
- `internal/service/action_report_service.go`
- `internal/repository/action_attempt_repo.go`

## 3.2.9 面试话术价值

这个场景最适合回答：
- 为什么不用 exactly-once，也能做出业务上接近 exactly-once
- 为什么“服务端幂等 + Agent WAL + CAS”比盲目追求 MQ exactly-once 更现实
- DB / Redis / Agent 三段式链路怎么兜底

---

## 3.3 场景三：多波次发布窗口 + Saga 补偿

## 3.3.1 现有锚点

当前 `TaskCenterWorker.completeStage()` 已经在做链式推进：

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

这条链本质上就是一个“阶段推进器”，只差把它升级成：
- 带阈值
- 带暂停点
- 带回滚动作
- 带自动 / 手动继续策略

## 3.3.2 业务目标

支持这样一条真实发布链路：
- 5 台 canary
- 50 台小流量
- 200 台半量
- 剩余全量

每个波次结束都做：
- 成功率检查
- 失败率检查
- 关键指标检查
- 自动继续 / 暂停等待 / 自动回滚

## 3.3.3 关键字段设计

在 `stage` 或新扩展表里增加：
- `batch_no`
- `batch_percent`
- `success_threshold`
- `error_threshold`
- `manual_gate`
- `rollback_policy`
- `rollback_stage_id`

## 3.3.4 核心流程

### 正向流程
1. Stage N 完成
2. 聚合成功率 / 失败率
3. 若达标：激活 Stage N+1
4. 若需要人工确认：置为 `WAITING_APPROVAL`
5. 若不达标：触发 rollback Stage

### 反向补偿流程
1. 当前 Stage 标记 `FAILED`
2. 发布单标记 `ROLLING_BACK`
3. 生成一组“反向 Action”
4. 只对已成功节点发回滚任务
5. 回滚完成后写 `release_rolled_back` outbox

## 3.3.5 一致性和事务建议

建议把下面几步放一个事务：
- 当前 Stage 状态更新
- 下一 Stage 激活 / rollback Stage 创建
- 发布单状态更新
- outbox 写入

避免现在这种非原子写法导致：
- 当前 stage 已成功
- 但下一 stage 没激活
- 或 rollback event 没发出去

## 3.3.6 补偿规则

### 自动回滚触发条件
- 失败率 > `error_threshold`
- 关键校验任务失败
- 核心节点失败数超过阈值

### 部分回滚失败
- 标记 `PARTIAL_ROLLBACK_FAILED`
- 支持按节点补回滚
- 按波次保留失败节点明细

### 人工确认
- `manual_gate=true` 时，不自动激活下一波
- 状态停留在 `WAITING_APPROVAL`
- 由接口驱动继续执行

## 3.3.7 代码改动点

- `data-platform/tbds-control/internal/server/dispatcher/task_center_worker.go`
  - 改成事务式 stage promotion
  - 引入阈值判断
  - 增加 rollback stage 构造逻辑
- `data-platform/tbds-control/internal/server/dispatcher/stage_worker.go`
  - 支持不同 batch 策略生成 task
- `data-platform/tbds-control/internal/server/api/job_handler.go`
  - 增加继续 / 暂停 / 回滚接口

## 3.3.8 这块为什么必须做

因为“配置发布 + 一键回滚”讲的是业务壳，
而“多波次 + Saga 补偿”讲的是设计深度。

面试官真正会追问的，其实是这一层。

---

## 四、distributed_task_platform 实施方案

---

## 4.1 场景一：任务暂停 / 恢复 / 取消

## 4.1.1 现有锚点

当前普通任务主链：

```go
acquiredTask, err := s.taskAcquirer.Acquire(ctx, task.ID, task.Version, s.nodeID)
execution, err := s.execSvc.Create(ctx, domain.TaskExecution{...})
go func() {
    state, runErr := s.invoker.Run(ctx, execution)
    if updateErr := s.execSvc.UpdateState(ctx, state); updateErr != nil { ... }
}()
```

终态处理：

```go
if !isShardedTask {
    s.releaseTask(ctx, execution.Task)
    if _, err3 := s.taskSvc.UpdateNextTime(ctx, execution.Task.ID); err3 != nil {
        ...
    }
}
s.sendCompletedEvent(ctx, state, execution)
```

所以 Pause / Resume / Cancel 不是平地起楼，而是在现有状态机上扩状态。

## 4.1.2 目标状态机

### 任务状态建议扩展
当前 DAO 里是：
- `ACTIVE`
- `PREEMPTED`
- `INACTIVE`

建议扩展为：
- `ACTIVE`
- `PREEMPTED`
- `PAUSING`
- `PAUSED`
- `CANCELLING`
- `CANCELLED`
- `INACTIVE`

### 执行记录状态建议补充
在现有 `Prepare / Running / WaitingRetry / Terminal` 体系上增加：
- `Pausing`
- `Paused`
- `Cancelling`
- `Cancelled`

## 4.1.3 API 设计

建议新增：
- `POST /tasks/{id}/pause`
- `POST /tasks/{id}/resume`
- `POST /tasks/{id}/cancel`
- `POST /executions/{id}/ack-pause`
- `POST /executions/{id}/ack-cancel`

## 4.1.4 核心流程

### 暂停流程
1. 用户发起暂停
2. `tasks.status: PREEMPTED -> PAUSING`（CAS）
3. 写 `pause_requested` outbox
4. relay 通知执行节点
5. 节点返回 ACK
6. execution 状态切到 `Paused`
7. task 状态切到 `PAUSED`

### 恢复流程
1. 用户发起恢复
2. `PAUSED -> ACTIVE`
3. 重算 `next_time`
4. 下次调度重新抢占

### 取消流程
1. 用户发起取消
2. `PREEMPTED/RUNNING -> CANCELLING`
3. 写 `cancel_requested` outbox
4. 节点收到后停止执行
5. 回 ACK 后 execution -> `Cancelled`
6. release 任务、归还配额、写完成事件

## 4.1.5 一致性要点

### 取消幂等
- 同一个 execution 的 cancel request 要有 request_id
- 已经处于 `CANCELLING/CANCELLED` 时重复请求直接返回当前状态

### release 一致性
取消成功后，要把下面几步收敛到一次事务或单个聚合方法里：
- execution -> Cancelled
- task release
- quota release
- outbox 写入 `task_cancelled`

### 恢复安全
- 只有 `PAUSED` 才能恢复到 `ACTIVE`
- 恢复前校验上一次 execution 是否已真正终止或冻结

## 4.1.6 代码改动点

### 修改现有文件
- `distributed_task_platform/internal/repository/dao/task.go`
  - 扩展状态枚举
  - 增加 Pause / Cancel / Resume 的 CAS 方法
- `distributed_task_platform/internal/service/task/execution_service.go`
  - 增加 pause / cancel 相关状态迁移
  - 终态后统一处理 release、next_time、event
- `distributed_task_platform/internal/service/runner/normal_task_runner.go`
  - 调度前跳过 `PAUSED/CANCELLING/CANCELLED`

### 建议新增文件
- `internal/service/task/control_service.go`
- `internal/event/control/producer.go`
- `internal/event/control/consumer.go`

## 4.1.7 风险点

- 节点不支持 pause，只支持 cancel：那就把 pause 降级为“冻结后续，不中断当前执行”
- 节点丢 ACK：通过补偿器扫描 `PAUSING/CANCELLING` 超时记录，重发控制事件

---

## 4.2 场景二：调度配额 / 并发令牌 / 扣减回滚

## 4.2.1 现有锚点

这个项目已经有负载检测基础：
- `pkg/loadchecker/*.go`
- `Acquire`
- `Create execution`
- `Release`

但还没有真正的“业务配额层”。

## 4.2.2 目标能力

支持下面这类真实约束：
- 某租户同时最多 100 个任务
- 某任务组最多占用 20 个 executor
- 某 plan 同时最多推进 3 个分支
- 高优任务可以抢占低优任务配额

## 4.2.2.1 这里的 quota / token 到底指什么

先把边界说清楚：这里不是在做 Spark / YARN / K8s 这类底层大数据引擎的 CPU / 内存资源调度，而是在做 `distributed_task_platform` 自己的**调度准入控制层**。

也就是说，它解决的不是“底层集群还能不能跑”，而是：
- 这个租户现在还能不能继续占用调度并发预算
- 这个任务组是不是已经跑太多了
- 这个 Plan 是不是应该限制同时推进的分支数
- 高优任务要不要保留专用预算

建议统一口径：
- **quota** = 某个 owner 的并发预算上限
- **token** = 某次 execution 实际占用的一份预算凭证
- **扣减回滚** = token 已预扣，但后续 `Acquire / CreateExecution / 下发执行` 任一步失败时，必须把预算原路退回，避免配额泄漏

所以它本质上是：

> 让调度平台从“能调度”升级为“会治理并发预算”。

## 4.2.2.2 它和现有 loadchecker 的区别

这块很容易和 `pkg/loadchecker` 混掉，但两者不是一回事：

- `loadchecker` 解决的是：**系统当前忙不忙、节点健不健康、数据库压不压得住**
- `quota / token` 解决的是：**即使系统还能跑，也不代表这个 owner 还能继续占预算**

可以把两者理解为两层门：
1. **健康门**：机器和系统扛不扛得住
2. **治理门**：你配不配继续占用并发预算

只有两道门都过了，任务才真正进入 `Acquire -> CreateExecution -> Run`。

## 4.2.2.3 这项能力有没有必要做

我的判断是：**如果你想把 `distributed_task_platform` 讲成成熟调度内核，这项很有必要做，而且优先级高。**

原因很直接：
- 它比 `pause / resume / cancel` 更不依赖外部引擎能力，不容易讲翻车
- 它能自然带出多租户隔离、公平性、背压、高优抢占、lease 回收这些中高级话题
- 它和现有 `Acquire / Release / Execution / Compensator / LoadChecker` 的拼接点非常自然
- 它非常适合做 MVP：即使只做单一维度 quota，也足够撑住面试追问

但也别做重：
- 如果当前项目只是单租户、小规模 demo，可以降级为 P1
- 如果你准备把它作为“调度内核成熟度”的核心卖点，就应该优先做

我更建议的 V1 收敛方式是：
- 先只做一个 owner 维度，比如 `task_group` 或 `tenant`
- 先只做 4 个动作：`acquire token -> create execution -> terminal release -> lease reclaim`
- 先不做复杂的跨维度叠加和高优抢占，把复杂度留到 V2

## 4.2.3 数据模型

建议新增：

### quota_bucket
- `id`
- `owner_type`：tenant/task_group/plan
- `owner_id`
- `capacity`
- `used`
- `version`

### quota_token_lease
- `id`
- `bucket_id`
- `execution_id`
- `token_count`
- `lease_owner`
- `expire_at`
- `status`

### quota_change_log
- `id`
- `bucket_id`
- `execution_id`
- `change_type`：acquire/release/reclaim/preempt
- `delta`
- `trace_id`

## 4.2.4 调度流程

1. 调度器发现任务可执行
2. 先尝试获取 quota token
3. token 成功才进入 `Acquire`
4. 抢占成功后创建 execution
5. 执行终态时 release token
6. 若节点宕机导致 token 泄漏，补偿器按 lease 超时回收

## 4.2.5 Redis + DB 双层设计

### 快路径
- Redis semaphore / token bucket
- 提供高并发快速扣减

### 慢路径
- DB 的 `quota_bucket + quota_token_lease + quota_change_log`
- 负责最终对账

### 推荐策略
- 先 Redis 预扣
- 再 DB 落 lease
- 任一步失败都回滚 Redis 扣减

## 4.2.6 事务与幂等

### acquireQuota 幂等
- 以 `(bucket_id, execution_id)` 唯一约束
- 同一 execution 重试扣减时直接返回已有 lease

### 终态释放幂等
- 只有 `lease.status=ACQUIRED` 才允许变 `RELEASED`
- 重复 release 不再二次归还

### 超时回收
- 扫描 `expire_at < now and status=ACQUIRED`
- 执行 `RECLAIMED`
- Redis 和 DB 同步回收

## 4.2.7 代码改动点

- `distributed_task_platform/internal/service/runner/normal_task_runner.go`
  - 在 `Acquire` 前增加 quota 获取
- `distributed_task_platform/internal/service/task/execution_service.go`
  - 终态统一 release quota
- `distributed_task_platform/pkg/loadchecker/*.go`
  - 将 quota checker 接入现有 CompositeChecker

### 建议新增文件
- `internal/service/quota/service.go`
- `internal/repository/quota_repo.go`
- `internal/compensator/quota_reclaimer.go`

## 4.2.8 为什么值得做

这个场景一旦补上，能讲的题会非常多：
- 公平性
- 背压
- 限流
- 多租户隔离
- 高优抢占
- 令牌泄漏补偿

这类题很像中高级后端面试官喜欢问的“平台治理能力”。

---

## 4.3 场景三：分片父子聚合 + 局部重试 + 最终完成事件

## 4.3.1 现有锚点

当前已有父子执行创建：

```go
created, err := s.repo.CreateShardingParent(ctx, parent)
if err != nil {
    return nil, err
}
return s.repo.CreateShardingChildren(ctx, created, executorNodeIDs, scheduleParams)
```

`NormalTaskRunner` 对子任务并行执行：

```go
for i := range executions {
    execution := executions[i]
    go func() {
        state, runErr := s.invoker.Run(s.WithSpecificNodeIDContext(ctx, execution.ExecutorNodeID), execution)
        if updateErr := s.execSvc.UpdateState(ctx, state); updateErr != nil { ... }
    }()
}
```

说明分片基础已经有了，下一步该补的是“生产级聚合闭环”。

## 4.3.2 目标能力

支持下面这些真实策略：
- 全成功才算成功
- 允许 1% 失败仍整体成功
- 失败分片单独重试
- 只重跑失败分片，不全量重跑
- 父任务完成时可靠投递最终完成事件

## 4.3.3 聚合策略模型

建议新增：
- `ALL_SUCCESS`
- `THRESHOLD_SUCCESS`
- `PARTIAL_ACCEPT`

字段建议：
- `success_threshold`
- `max_failed_shards`
- `max_retry_per_shard`
- `emit_summary_snapshot`

## 4.3.4 聚合流程

1. 子分片进入终态
2. 聚合器扫描该父任务所有子分片
3. 根据聚合策略判断父状态
4. 若有失败但允许局部重试，则仅重试失败分片
5. 若父任务进入终态，则：
   - 更新父 execution 状态
   - release task
   - updateNextTime
   - 写 `sharding_parent_completed` outbox
6. relay 异步投递最终完成事件

## 4.3.5 事务边界

这几步应该放同一事务：
- 父任务状态更新
- task release
- next_time 更新
- outbox 写入

原因是 `execution_service.go` 当前这几步是串行调用，不是原子提交：

```go
if !isShardedTask {
    s.releaseTask(ctx, execution.Task)
    if _, err3 := s.taskSvc.UpdateNextTime(ctx, execution.Task.ID); err3 != nil {
        ...
    }
}
s.sendCompletedEvent(ctx, state, execution)
```

分片聚合做完后，正好可以把这块也重构成统一事务入口。

## 4.3.6 数据模型

建议新增：

### sharding_aggregate_policy
- `task_id`
- `policy_type`
- `success_threshold`
- `max_retry_per_shard`

### sharding_result_snapshot
- `parent_execution_id`
- `total_shards`
- `success_count`
- `failed_count`
- `timeout_count`
- `snapshot_version`
- `summary_payload`

### sharding_outbox
- `aggregate_id`
- `event_type`
- `payload`
- `status`

## 4.3.7 幂等与补偿

### 子任务重复上报
- `execution_id + state_version` 做幂等控制

### 聚合重复执行
- 对父任务加 `aggregate_version`
- 只有当前版本匹配才允许写入新聚合结果

### 完成事件重复投递
- outbox relay 至少一次投递
- 消费端基于 `event_id` 幂等

### 局部重试
- 只对 `FAILED/TIMEOUT` 的 shard 创建 retry execution
- 父任务保留原聚合快照，新的 retry 结果覆盖失败分片结果

## 4.3.8 代码改动点

- `distributed_task_platform/internal/service/task/execution_service.go`
  - 增加 `completeShardingParentTx` 统一事务方法
- `distributed_task_platform/internal/service/runner/normal_task_runner.go`
  - 分片重试时注入 specific node / excluded node 策略
- `distributed_task_platform/internal/repository/dao/task.go`
  - 提供事务友好的 release + nextTime 更新接口

### 建议新增文件
- `internal/service/sharding/aggregate_service.go`
- `internal/repository/sharding_snapshot_repo.go`
- `internal/event/complete/sharding_relay.go`

## 4.3.9 为什么这块特别值钱

因为“我支持分片”不难，
真正值钱的是：
- 父子状态怎么聚合
- 局部失败怎么处理
- 最终事件怎么可靠发出去
- 为什么不用全量重跑

这几个问题答好，项目成熟度会明显更像真实生产系统。

---

## 五、两周实施排期建议

## 5.1 第一周：先做最能闭环的骨架

### data-platform

#### Day 1-2
- 建 `config_release / config_release_batch / release_outbox`
- 抽 `release_service`
- 打通创建发布单 API

#### Day 3-4
- `stage_worker / task_worker / task_center_worker` 支持 canary -> full
- 引入 stage CAS 和事务式推进

#### Day 5
- 做最小版 rollback
- 先不做自动阈值，先支持手工触发回滚

### distributed_task_platform

#### Day 1-2
- 扩 `task` 和 `execution` 状态机
- 增 pause / resume / cancel 控制接口

#### Day 3-4
- 增加 quota 表、lease 表、reclaimer 补偿器
- 把 quota 接到 `NormalTaskRunner` 前置流程

#### Day 5
- 分片聚合统一事务收口
- 父任务完成 outbox 打通

## 5.2 第二周：补可靠性和面试深水区

### data-platform
- Agent WAL
- report retry queue
- action attempt 幂等
- 多波次阈值控制 + Saga 补偿

### distributed_task_platform
- 局部 shard retry
- threshold success
- cancel/pause 超时补偿
- quota 抢占与高优降级策略

---

## 六、推荐的落地顺序

如果只能先做 3 个，我建议：

1. `data-platform`：配置灰度发布 + 一键回滚
2. `distributed_task_platform`：任务暂停 / 恢复 / 取消
3. `distributed_task_platform`：调度配额 / 并发令牌 / 扣减回滚

原因：
- 这三个最像真实生产系统
- 也最容易被面试官追问
- 讲出来业务味、系统味、工程味都够

如果能做满 6 个，整个面试叙事就完整了：
- 一个项目负责“业务执行平台”
- 一个项目负责“调度内核成熟度”

---

## 七、最后的判断

这份实施方案最核心的价值，不是又多了 6 个功能点，而是把两个项目的演进方向彻底拉开：

### data-platform
讲：
- 灰度发布
- 回滚
- Agent WAL
- DB / Redis / Agent 一致性
- 发布波次与 Saga

### distributed_task_platform
讲：
- Pause / Resume / Cancel
- Quota / Token
- 分片聚合
- 局部重试
- Outbox 可靠事件

这样面试时不会变成“两套相似的调度 demo”，而会变成：

> 一个负责业务执行闭环，一个负责调度内核闭环。

这才是最强组合。

---

## 八、这些场景真正实现后，对面试能力提升到底有多大

### 8.1 先给结论

结论很明确：**提升大，而且是结构性提升。**

这里的“结构性提升”，不是简历上多写几个功能点，而是会直接改变两个项目在面试中的形象：

- `data-platform` 会从“任务下发骨架”升级成“集群配置发布 / 巡检修复 / 批量变更执行平台”
- `distributed_task_platform` 会从“会调度任务的框架”升级成“强状态机 / 强补偿 / 强编排的调度内核”

这会明显改善当前一个核心短板：**文档和设计很强，但代码兑现度偏弱。**

只要这些场景真的落到代码、测试和演示链路里，面试官对项目的判断会从：

> 设计感不错，但更像文档型项目

变成：

> 这个人真的做过复杂状态机、一致性、补偿、缓存和可靠性闭环

### 8.2 为什么提升会这么明显

#### 1）两个项目终于形成清晰分工，而不是两个相似的调度 demo

这批场景最大的价值，是把两个项目的定位彻底拉开：

##### `data-platform`
更适合承载：
- 灰度发布
- 一键回滚
- 波次推进
- Agent WAL
- DB / Redis / Agent 三段式一致性
- 发布 Saga

##### `distributed_task_platform`
更适合承载：
- 状态机
- 配额治理
- 分片聚合
- 局部重试
- Outbox 可靠事件
- Trigger Dedup

这样面试时你讲的是两个互补系统：
- 一个是**业务执行平台**
- 一个是**调度内核平台**

辨识度会明显变强。

#### 2）把“概念”变成“可证明的实现”

当前你已经能讲很多设计概念，但如果真把这 6 个场景落下去，能直接证明你做过下面这些东西：

- 复杂状态机建模
- 本地事务边界收敛
- Transactional Outbox
- 业务幂等
- Redis / DB / Agent 一致性
- Saga 补偿
- Token lease 回收
- 分片父子聚合

也就是说，面试官听到的不再只是关键词，而是一整条能跑通的链路。

#### 3）能明显补“深挖时容易虚”的点

这些场景一旦实现，很多常见追问你都会更稳：

##### `data-platform` 方向
- 为什么不用 2PC
- 波次推进为什么一定要事务化
- 只回滚已成功节点怎么做
- Agent 执行成功但上报失败怎么办
- DB / Redis / Agent 三层状态怎么兜底

##### `distributed_task_platform` 方向
- quota 为什么不能只用 Redis
- token 泄漏怎么回收
- 父子分片什么时候算完成
- 为什么局部重试比全量重跑更合理
- outbox 为什么比直接发 MQ 更稳

### 8.3 对不同面试环节的帮助大小

#### 1）技术一面 / 二面：**提升最大**

这是收益最大的环节。因为这类面试最看重：
- 系统是不是有完整状态机
- 事务边界收没收住
- 幂等和补偿是不是能讲细
- 一致性问题是不是讲得出来

这 6 个场景正好都打在这里。

#### 2）系统设计面：**提升非常大**

因为这里补的不是页面或普通 CRUD，而是高频系统设计题：
- 灰度发布
- 回滚
- 缓存一致性
- 至少一次 + 业务幂等
- 配额治理
- 分片聚合
- 可靠事件
- Trigger 去重

如果这些都能回到项目里讲，系统设计面会明显更有说服力。

#### 3）简历筛选：**中等提升**

会提升项目关键词质量、业务语义和平台感，但简历初筛依然受：
- 年限
- 公司背景
- 技术栈匹配
- 项目包装标题
影响，所以收益有，但不是最大头。

#### 4）代码面 / Go 基础面：**提升有限到中等**

因为这些场景主要提升的是架构能力和系统设计能力，不会自动替代：
- Go 基础
- 并发模型
- 常见语言坑
- 算法与手写代码能力

#### 5）BQ / 行为面：**几乎不直接提升**

除非你再把这些实现包装成：
- 为什么要做
- 如何推动落地
- 遇到什么权衡
- 哪些风险是怎么控制的

否则它们对 BQ 的帮助有限。

### 8.4 分项目看，提升具体体现在哪

#### `data-platform`：提升非常明显，而且最有业务味

最值钱的是：
1. 配置灰度发布 + 一键回滚
2. 批量巡检 / 修复任务的幂等下发 + Agent WAL
3. 多波次发布窗口 + Saga 补偿

这三项一旦落下来，`data-platform` 的项目质感会明显变：

- 不再只是 Job / Stage / Task / Action 的骨架
- 而是一个能讲“发布单 / 波次 / 节点目标 / 配置版本 / 回滚范围 / 变更审计”的平台

它最能拉高的，是项目的业务真实感和平台感。

#### `distributed_task_platform`：提升也大，但核心要靠“调度内核成熟度”来抬

最值钱的是：
1. 调度配额 / 并发令牌 / 扣减回滚
2. 分片父子聚合 + 局部重试 + 最终完成事件
3. 调度触发去重

`Pause / Resume / Cancel` 也有价值，但必须控制语义边界，不能讲得太满。

### 8.5 关于 `Pause / Resume / Cancel` 的面试口径，必须收敛

这个点单独强调一下。

对于轻量任务、平台内部任务、协作式中断任务，这套能力是成立的。

但如果任务已经真正提交到外部大数据引擎，例如：
- Spark
- Flink
- YARN Application
- K8s Job

那当前设计不能宣称支持完整的引擎级 pause / resume / cancel。原因包括：
- 现有 executor 协议只有 `Execute / Interrupt / Query / Prepare`
- 没有真正的 `Pause / Cancel` 原语
- `task_execution` 没有保存远端 `job_id / application_id / savepoint / checkpoint`
- 当前 resume 更接近 `rescheduled_params` 驱动的重新调度 / 断点续跑

所以更准确的表述是：

#### V1 能讲
- 调度冻结
- 协作式中断
- 基于 checkpoint / reschedule 的恢复

#### V2 才能讲
- 引擎级 cancel
- savepoint resume
- Spark / Flink / YARN / K8s Job adapter

这块如果口径不收，会有面试翻车风险。

### 8.6 这批实现会不会直接把档位抬一档

不会简单粗暴地变成：
- 稳 P8
- 稳 T10
- 稳 2-3

但它会带来两个非常实在的变化：

#### 1）让当前主档更稳
结合你现在的整体情况，这批实现会让：
- 阿里 / 蚂蚁 P7
- 腾讯 T9
- 字节 2-2

从“有机会、但要看发挥”更往“主档更稳、深挖更能扛”方向走。

#### 2）提高上浮空间
这类实现会增加你冲更高一档时的底气，因为：
- 系统设计面更稳
- 项目深挖更抗打
- 面试官更容易相信你是做过而不是只会设计

### 8.7 但它不能替代下面这些硬短板

就算 6 个场景全做了，也不能替代：

1. **真实压测和量化数据**
   - 阈值怎么来的
   - token 扣减吞吐多少
   - WAL 恢复时间多少
   - outbox relay 峰值多大

2. **Go 基础和底层功底**
   - 项目强不代表八股和底层一定能扛

3. **BQ 和业务影响力**
   - 为什么做
   - 推动难点
   - 协作方式
   - 风险取舍

4. **工程成熟度**
   - 测试
   - 演示
   - 日志
   - 故障注入
   - 最小量化结果

### 8.8 最后一句判断

如果你只是把这些内容继续停留在文档层，提升是有的，但有限。

如果你把其中高价值场景真正做到：
- 有状态机
- 有表结构
- 有接口
- 有最小演示链路
- 有测试和少量量化数据

那这批场景对面试能力提升就是**明显且成体系的**。

更直接地说：

> 它们会显著提高你技术一面、系统设计面和项目深挖面的说服力，让两个项目从“文档和设计很强”走到“实现和叙事都很强”。
