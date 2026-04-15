# 6 个场景对面试收益的 ROI 排名 + 最小实现集建议

> 目标：不是继续铺更多功能，而是回答一个更现实的问题：**如果时间有限，哪几个场景最值得先落？每个场景做到什么程度，面试收益就已经足够高？**

---

## 一、结论先说

如果只从“面试收益 / 实现成本 / 翻车风险 / demo 可做性 / 深挖抗打度”综合看，这 6 个场景的 ROI 排名我给出下面这个版本：

| 排名 | 场景 | 项目 | ROI 判断 | 一句话结论 |
|---|---|---|---|---|
| 1 | 配置灰度发布 + 一键回滚 | data-platform | 极高 | 业务味最强，最容易把 `data-platform` 拉成平台级项目 |
| 2 | 调度配额 / 并发令牌 / 扣减回滚 | distributed_task_platform | 极高 | 中高级面试官最爱问的平台治理题，系统味很足 |
| 3 | 分片父子聚合 + 局部重试 + 最终完成事件 | distributed_task_platform | 很高 | 最能体现调度内核成熟度，追问空间大 |
| 4 | 批量巡检 / 修复任务的幂等下发 + Agent WAL | data-platform | 很高 | 可靠执行链最硬，容易讲清一致性和业务上的 exactly-once-ish |
| 5 | 多波次发布窗口 + Saga 补偿 | data-platform | 中高 | 技术深度很强，但依赖前置发布模型，适合做第二阶段增强 |
| 6 | 任务暂停 / 恢复 / 取消 | distributed_task_platform | 中高 | 有价值，但语义边界复杂，对大数据任务容易讲过头 |

### 为什么不是把 pause / resume / cancel 排得更高

不是它不值钱，而是它有两个问题：

1. **语义边界容易翻车**
   - 对轻任务成立
   - 对真正下发到 Spark / Flink / YARN / K8s Job 的任务，当前只能讲调度冻结 / 协作式中断 / checkpoint 恢复

2. **实现成本不只在状态机**
   - 真要做强，要补协议、远端句柄、引擎适配、savepoint/checkpoint
   - 很容易做着做着投入过大

所以它更适合排在第二阶段，而不是一上来压第一优先级。

---

## 二、ROI 排名方法说明

这份排名不是拍脑袋，主要按 5 个维度综合判断：

### 2.1 面试收益
看这个场景是否能明显提升：
- 技术一面深挖
- 系统设计面
- 项目辨识度
- 中高级问题抗打度

### 2.2 实现成本
看它是否需要：
- 大量基础设施改造
- 多模块联动
- 很高的联调成本
- 很久才能看到可演示结果

### 2.3 翻车风险
看它是否容易出现：
- 语义说不清
- 面试官一问就露怯
- 设计很大但代码落不下来

### 2.4 Demo 可做性
看它能不能较快做出：
- 可视化状态变化
- 故障注入
- 最小闭环
- 面试演示素材

### 2.5 叙事闭环完整度
看它能不能自然讲到：
- 业务语义
- 状态机
- 事务边界
- 幂等
- 补偿
- 为什么不用 2PC / exactly-once

---

## 三、排名详情

---

## 3.1 Top 1：配置灰度发布 + 一键回滚（data-platform）

### 为什么排第一

这是 6 个场景里**综合收益最高**的一个。

因为它同时满足：
- 业务语义最真实
- 平台感最强
- 技术点最完整
- 面试官最容易理解价值
- demo 最好做

它能直接把 `data-platform` 从“任务下发骨架”提升成：

> 面向集群的配置发布 / 灰度控制 / 回滚执行平台

### 能讲的面试点

- 发布单建模
- canary -> full 波次推进
- 当前阶段完成 + 下一阶段激活的事务边界
- 为什么不用 2PC
- 已成功节点回滚、未执行节点取消
- DB / Redis / Agent 三段式一致性
- Agent 成功执行但上报失败怎么兜底

### 为什么 ROI 极高

因为它不是单点功能，而是一整条闭环：
- 有业务壳
- 有状态机
- 有一致性
- 有补偿
- 有回滚
- 有 outbox

### 最小实现集建议

如果时间有限，这个场景不用一开始就做满，先做下面这些，面试收益就已经很高：

#### M1：最小可讲版
- `config_release`
- `config_release_batch`
- `release_outbox`
- 创建发布单 API
- canary / full 两波次
- `completeStage` 事务化推进
- 手工触发 rollback

#### M2：进阶版
- 配置版本校验
- 只回滚已成功节点
- 发布完成 outbox relay
- 基础审计事件

#### M3：加强版
- 自动阈值判断
- 暂停 / 恢复波次
- 人工 gate
- 发布指标看板

### 我的建议

这个场景**必须做**，而且要优先做。

---

## 3.2 Top 2：调度配额 / 并发令牌 / 扣减回滚（distributed_task_platform）

### 为什么排第二

这是最像中高级平台治理题的一个场景。

它能把 `distributed_task_platform` 从“会跑任务”升级成“会治理资源”。

这类题很容易打到面试官的专业点：
- 多租户隔离
- 公平性
- 背压
- 高优先级抢占
- token 泄漏回收
- Redis 快路径 + DB 对账

### 能讲的面试点

- 为什么不能只靠 loadchecker
- 为什么 quota 要单独建模
- 为什么不能只用 Redis
- 为什么 lease 比纯计数更稳
- 节点宕机后 token 怎么回收
- 高优任务如何抢占低优任务资源

### 为什么 ROI 极高

因为它的实现相对聚焦，但技术密度很高。

它不像 pause/resume/cancel 那样容易陷入语义争议，也不像多波次 Saga 那样前置依赖更强。

### 这个能力在系统里具体指什么

这里的 `quota / token` 不是底层大数据引擎的 CPU / 内存资源调度，而是 `distributed_task_platform` 自己的**调度准入控制和并发治理层**。

更准确地说：
- **quota** = 某个 owner 的并发预算上限
- **token** = 某个 execution 实际占用的一份预算凭证
- **扣减回滚** = token 已预扣，但后续 `Acquire / CreateExecution / 下发执行` 失败时，必须把预算退回去

它管的是：
- 某租户还能不能继续跑
- 某任务组是不是已经占满
- 某个 Plan 是否要限制并行推进数
- 高优任务是否要预留预算

### 它和 loadchecker 不是一回事

- `loadchecker` 管系统健康：节点忙不忙、数据库压不压得住
- `quota / token` 管治理边界：即使系统还能跑，也不代表你还能继续占预算

一句话：
- `loadchecker` 决定“系统扛不扛得住”
- `quota / token` 决定“你配不配继续占资源”

### 到底有没有必要做

如果只是单租户、小规模 demo，可以不放到最优先；但如果要把 `distributed_task_platform` 讲成成熟调度内核，这项非常值得做，而且优先级高于 pause/resume/cancel。

原因：
- 更不依赖外部引擎能力，不容易讲翻车
- 能稳定带出多租户隔离、公平性、背压、lease 回收这些中高级话题
- 和现有 `Acquire / Release / Execution / Compensator` 的拼接非常自然
- 适合先做 MVP，不需要一开始就做成复杂资源系统

### 最小实现集建议

#### M1：最小可讲版
- `quota_bucket`
- `quota_token_lease`
- `quota_change_log`
- 调度前获取 token
- 执行终态释放 token
- 超时 lease reclaim 补偿器

#### M2：进阶版
- Redis semaphore 快路径
- DB 最终对账
- `(bucket_id, execution_id)` 幂等约束
- quota checker 接入现有 CompositeChecker

#### M3：加强版
- 高优先级抢占低优 quota
- 分租户 / 分任务组 / 分 plan 配额
- 配额命中监控和告警

### 我的建议

这个场景应该排在 `distributed_task_platform` 的第一优先级。

---

## 3.3 Top 3：分片父子聚合 + 局部重试 + 最终完成事件（distributed_task_platform）

### 为什么排第三

这个点最能体现“调度内核成熟度”。

因为“支持分片”本身不稀奇，真正值钱的是：
- 父任务怎么收口
- 局部失败怎么处理
- 为什么只重试失败分片
- 最终完成事件怎么可靠发出去

这类问题很适合中高级面试深挖。

### 能讲的面试点

- 父子状态聚合一致性
- threshold success / partial accept
- 局部重试和全量重跑的取舍
- 父任务完成时 release + nextTime + outbox 为什么要放同事务
- shard 结果快照为什么要版本化

### 为什么 ROI 很高

因为它基本是在已有 sharding 基础上做增强，不是完全另起炉灶。

### 最小实现集建议

#### M1：最小可讲版
- `sharding_aggregate_policy`
- `sharding_result_snapshot`
- `completeShardingParentTx`
- 父任务完成时统一事务收口
- outbox 完成事件

#### M2：进阶版
- `ALL_SUCCESS / THRESHOLD_SUCCESS / PARTIAL_ACCEPT`
- 失败 shard 单独重试
- 聚合版本号控制
- snapshot summary

#### M3：加强版
- 按失败类型区别重试策略
- 失败分片定向换节点
- 聚合结果看板

### 我的建议

这个场景和 quota 放在一起做，会非常像“真正的调度平台内核”。

---

## 3.4 Top 4：批量巡检 / 修复任务的幂等下发 + Agent WAL（data-platform）

### 为什么排第四

这个场景的技术含金量很高，但业务可感知度略低于“灰度发布 + 回滚”。

它最值钱的地方在于：
- 能讲业务上的 exactly-once-ish
- 能讲 DB / Redis / Agent 三段式一致性
- 能讲 Agent 崩溃恢复
- 能讲 report 幂等

### 能讲的面试点

- 为什么允许 at-least-once 下发
- 为什么服务端幂等 + Agent WAL 比盲目追求 MQ exactly-once 更现实
- Redis 重复投递怎么避免重复执行
- Agent 成功执行但上报失败怎么办

### 为什么 ROI 仍然很高

因为这套链路非常硬，面试官一听就知道不是普通 CRUD。

### 最小实现集建议

#### M1：最小可讲版
- `biz_key`
- `attempt_no`
- `fetch_token`
- `last_report_seq`
- `Action: Init -> Cached -> Fetched -> Executing -> Reported -> Success/Failed`
- Agent `action_wal.log`
- report retry queue

#### M2：进阶版
- `uk_biz_key`
- `(action_id, attempt_no, report_seq)` 幂等更新
- WAL 启动恢复线程
- Redis 丢失后的 DB 重建策略

#### M3：加强版
- 按命令类型区分 WAL 策略
- 结果摘要 / 大结果存储分离
- 批量重放和补报监控

### 我的建议

这个场景非常值得做，但可以排在灰度发布之后。

---

## 3.5 Top 5：多波次发布窗口 + Saga 补偿（data-platform）

### 为什么排第五

这个场景技术深度很强，但它比较依赖前面的“发布单模型”和“回滚基础设施”。

也就是说，它不是不值钱，而是**更适合做成第二阶段增强项**。

### 能讲的面试点

- success_threshold / error_threshold
- 自动停止扩散
- 自动回滚和人工 gate
- Saga 前向动作与反向补偿
- 部分回滚失败后的人工接管

### 为什么 ROI 低于前四

因为没有前面的发布单、波次、回滚基本盘，这个点很难单独成立。

### 最小实现集建议

#### M1：最小可讲版
- `batch_no`
- `batch_percent`
- `success_threshold`
- `error_threshold`
- `manual_gate`
- rollback stage 构造逻辑

#### M2：进阶版
- 自动继续 / 自动暂停 / 自动回滚
- 波次失败明细
- 回滚只针对已成功节点

#### M3：加强版
- 指标驱动 gate
- 人工审批继续
- 不同批次不同策略模板

### 我的建议

不要一上来先做它。先把发布骨架和回滚跑通，再补它，收益更高。

---

## 3.6 Top 6：任务暂停 / 恢复 / 取消（distributed_task_platform）

### 为什么排第六

不是这个点不重要，而是**性价比没前五那么稳**。

### 主要问题

#### 1）语义边界复杂

对轻量任务成立；
对真正下发到外部大数据引擎的任务，不成立。

当前最多能讲：
- 调度冻结
- 协作式中断
- checkpoint / reschedule 恢复

不能轻易讲成：
- 引擎级 pause
- 原地 resume
- 任意任务 cancel

#### 2）实现容易越做越大

如果真想做强：
- 协议要扩
- 远端 job handle 要持久化
- savepoint / checkpoint 要建模
- 每种引擎都要 adapter

这会快速膨胀。

### 但它仍然有价值

因为调度平台如果完全没有 pause/cancel 语义，也确实容易被追问。

所以正确策略不是不做，而是：

> **做一个语义收敛、可防守的版本。**

### 最小实现集建议

#### M1：最小可讲版
- `desired_status`
- `task_control_operations`
- `task_control_outbox`
- 任务级 pause / resume / cancel API
- 对轻量任务支持 pause / cancel
- 对不支持 pause 的任务降级为“冻结未来调度，不中断当前执行”

#### M2：进阶版
- execution 增加 `PAUSED / CANCELLED`
- control relay
- control compensator
- request_id 幂等

#### M3：加强版
- 引擎适配器
- job handle 持久化
- checkpoint/savepoint resume
- Spark / Flink / YARN / K8s Job 差异化控制

### 我的建议

这个点不要做成第一优先级。除非你非常明确只面向轻量任务场景，否则很容易投入大、收益不如预期。

---

## 四、如果时间只够做 3 个，怎么选

### 最优 3 选

1. `data-platform`：配置灰度发布 + 一键回滚
2. `distributed_task_platform`：调度配额 / 并发令牌 / 扣减回滚
3. `distributed_task_platform`：分片父子聚合 + 局部重试 + 最终完成事件

### 原因

这三个组合起来，正好覆盖：
- 真实业务平台语义
- 调度内核治理能力
- 一致性 / 幂等 / 补偿 / outbox
- 状态机和事务边界
- 高概率面试追问项

这已经足够撑起一轮很强的项目深挖。

---

## 五、如果时间够做 4 个，最优组合是什么

1. `data-platform`：配置灰度发布 + 一键回滚
2. `data-platform`：批量巡检 / 修复任务的幂等下发 + Agent WAL
3. `distributed_task_platform`：调度配额 / 并发令牌 / 扣减回滚
4. `distributed_task_platform`：分片父子聚合 + 局部重试 + 最终完成事件

### 这个组合的价值最大

因为它正好把两个项目打磨成：

#### `data-platform`
- 业务执行闭环
- 灰度、回滚、WAL、一致性

#### `distributed_task_platform`
- 调度内核闭环
- 配额、分片聚合、重试、可靠事件

这四个做完，项目矩阵已经很强了。

---

## 六、每个场景的“最小可面试实现线”

这里给一个更务实的标准：**做到什么程度，就足够拿去面试讲，而不是必须做满。**

| 场景 | 最小可面试实现线 |
|---|---|
| 配置灰度发布 + 一键回滚 | 发布单 + canary/full 两波次 + 手工 rollback + stage 事务推进 |
| 调度配额 / 并发令牌 / 扣减回滚 | quota bucket + lease + acquire/release + reclaim 补偿器 |
| 分片父子聚合 + 局部重试 + 最终完成事件 | 父子聚合事务收口 + outbox 完成事件 + 失败 shard 单独重试 |
| 批量巡检 / 修复任务的幂等下发 + Agent WAL | biz_key + fetch_token + WAL + report 幂等 |
| 多波次发布窗口 + Saga 补偿 | success/error threshold + manual gate + rollback stage |
| 任务暂停 / 恢复 / 取消 | desired_status + control_operation + 轻任务 pause/cancel + 大数据任务语义收敛 |

---

## 七、我给你的最终建议

### 7.1 先做什么

按这个顺序最合适：

#### 第一批，必须先做
1. 配置灰度发布 + 一键回滚
2. 调度配额 / 并发令牌 / 扣减回滚
3. 分片父子聚合 + 局部重试 + 最终完成事件

#### 第二批，再做增强
4. 批量巡检 / 修复任务的幂等下发 + Agent WAL
5. 多波次发布窗口 + Saga 补偿

#### 第三批，谨慎做
6. 任务暂停 / 恢复 / 取消

### 7.2 为什么这么排

因为这个排序同时兼顾：
- 面试收益
- 实现成本
- 语义风险
- demo 速度
- 项目差异化

### 7.3 最后一条最实在的建议

不要追求“6 个都做满”。

更好的打法是：

> **先把前 3 个做到能演示、能测试、能讲事务边界和补偿，再补第 4 个。**

做到这一步，你拿这两个项目出去，面试竞争力就已经会明显上一个台阶。
