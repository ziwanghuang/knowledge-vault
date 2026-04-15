# 📚 Knowledge Vault — 知识库总索引

> 所有 AI 对话沉淀、技术分析、面试准备、项目研究文档的统一归档处。
>
> 更新时间：2026-04-15

---

## 目录结构

```
knowledge-vault/
├── interview-prep/          # 🎯 面试准备
│   ├── strategy/            # 策略与行动规划
│   ├── highlights/          # 亮点梳理与话术
│   └── assessment/          # 质量评估与风险分析
├── project-analysis/        # 🔍 项目分析与决策
│   ├── competitiveness/     # 竞争力评估
│   └── architecture-decisions/ # 架构与业务决策
├── tech-deep-dive/          # 🔬 技术深度研究
│   ├── distributed-systems/ # 分布式系统（事务、一致性）
│   ├── service-governance/  # 服务治理
│   ├── feature-enhancement/ # 功能增强规划
│   └── tbds-billing/        # TBDS 交易计费系统
├── ai-knowledge/            # 🤖 AI 知识体系（→ 软链接到 ../ai-knowledge）
├── career-thinking/         # 💡 职业规划与行业思考（→ 软链接到 ../ai-thinking）
└── talks-and-shares/        # 🎤 技术分享与演讲
```

---

## 一、🎯 面试准备 `interview-prep/`

### 策略与行动规划 `strategy/`

| 文件 | 内容概要 |
|------|----------|
| [面试准备-完整行动清单](interview-prep/strategy/面试准备-完整行动清单.md) | 汇总 6 份分析文档，7 个维度 32 项行动，目标 T9-T10 / 2-2 到 2-3 |
| [面试准备-重点与文档盲区分析](interview-prep/strategy/面试准备-重点与文档盲区分析.md) | 三项目 520+ 题全量扫描，识别已覆盖维度和文档盲区 |
| [面试准备-六场景ROI排名与最小实现集](interview-prep/strategy/面试准备-六场景ROI排名与最小实现集.md) | 6 个场景按面试收益/成本/风险综合 ROI 排名 |
| [面试准备-项目包装策略与服务治理融入](interview-prep/strategy/面试准备-项目包装策略与服务治理融入.md) | TBDS 管控项目面试包装策略、服务治理文档融入分析 |
| [团队工程化建设-Leadership叙事素材](interview-prep/strategy/团队工程化建设-Leadership叙事素材.md) | 4 个方向（API 规范/错误码/API 自动录入/AI 提效）的背景、做法、话术、串联叙事 |
| [三项目-实现路径规划](interview-prep/strategy/三项目-实现路径规划.md) | 基于代码摸底的 21 天 4 阶段实现路径，含工作量评估、甘特图、优先级调整策略 |
| [三项目-跨项目实现顺序规划](interview-prep/strategy/三项目-跨项目实现顺序规划.md) | 三项目间优先级排序：AIOps 打磨→DP 核心链路→团队工程化，14 天精简方案 |

### 亮点梳理与话术 `highlights/`

| 文件 | 内容概要 |
|------|----------|
| [面试亮点-基于自身项目](interview-prep/highlights/面试亮点-基于自身项目.md) | 基于三个项目的面试亮点，标注频率和面试官兴趣等级 |
| [面试亮点-通用版-后端2到8年](interview-prep/highlights/面试亮点-通用版-后端2到8年.md) | 通用版亮点框架，S/A/B/C 评级，面试官关注的 7 件事 |
| [TBDS管控-业务亮点匹配分析](interview-prep/highlights/TBDS管控-业务亮点匹配分析.md) | TBDS 管控业务 × 面试亮点交叉分析，话术建议 |

### 质量评估与风险 `assessment/`

| 文件 | 内容概要 |
|------|----------|
| [三项目-文档质量评估与面试风险](interview-prep/assessment/三项目-文档质量评估与面试风险.md) | 三项目 docs/ 文档质量评估，均获 5 星面试印象分 |
| [三项目-竞争力定位与求职分析](interview-prep/assessment/三项目-竞争力定位与求职分析.md) | 全市场前 3-5% / 大厂前 10-15% 定位，公司梯队 Offer 概率，求职策略 |

---

## 二、🔍 项目分析与决策 `project-analysis/`

### 竞争力评估 `competitiveness/`

| 文件 | 内容概要 |
|------|----------|
| [四项目-竞争力深度分析与新项目建议](project-analysis/competitiveness/四项目-竞争力深度分析与新项目建议.md) | 代码级审查 + 市场竞争力评分矩阵 + 新项目方向推荐 |
| [三项目-全量实现后竞争力评估](project-analysis/competitiveness/三项目-全量实现后竞争力评估.md) | 三项目 docs 全量实现后 S 级（9.5/10）竞争力评估，60 项亮点 + 面试叙事策略 |

### 架构与业务决策 `architecture-decisions/`

| 文件 | 内容概要 |
|------|----------|
| [notification-platform-退役决策与亮点迁移](project-analysis/architecture-decisions/notification-platform-退役决策与亮点迁移.md) | 砍掉 notification-platform，15 项亮点分散融入其他项目 |
| [TBDS管控-核心业务梳理](project-analysis/architecture-decisions/TBDS管控-核心业务梳理.md) | TBDS 管控平台全业务领域梳理（集群、计费、扩缩容、联邦等） |

---

## 三、🔬 技术深度研究 `tech-deep-dive/`

### 分布式系统 `distributed-systems/`

| 文件 | 内容概要 |
|------|----------|
| [分布式事务-一致性与幂等性分析](tech-deep-dive/distributed-systems/分布式事务-一致性与幂等性分析.md) | DP + DTP 全场景事务/一致性分析，含 Outbox 改进方案 |
| [组合商品一致性-业界方案全景](tech-deep-dive/distributed-systems/组合商品一致性-业界方案全景.md) | Saga/TCC/本地消息表/事务消息等业界方案完整对比 |

### 服务治理 `service-governance/`

| 文件 | 内容概要 |
|------|----------|
| [服务治理-三项目评分对比与补齐方案](tech-deep-dive/service-governance/服务治理-三项目评分对比与补齐方案.md) | 三项目 21 项服务治理能力全面盘点，定量评分 |

### 功能增强规划 `feature-enhancement/`

| 文件 | 内容概要 |
|------|----------|
| [DP+DTP-高频场景设计清单](tech-deep-dive/feature-enhancement/DP+DTP-高频场景设计清单.md) | 6 个 P0 优先场景设计（灰度发布、配额、WAL 等） |
| [DP+DTP-高频场景实施方案](tech-deep-dive/feature-enhancement/DP+DTP-高频场景实施方案.md) | 6 个场景的落地实施：表设计、状态机扩展、幂等兜底 |

### TBDS 交易计费系统 `tbds-billing/`

| 文件 | 阅读顺序 | 内容概要 |
|------|:--------:|----------|
| [TBDS交易计费-详细业务梳理](tech-deep-dive/tbds-billing/TBDS交易计费-详细业务梳理.md) | ① | 预付费/后付费完整业务流程、资源生命周期、数据模型 |
| [TBDS交易计费-业务流程与失败处理](tech-deep-dive/tbds-billing/TBDS交易计费-业务流程与失败处理.md) | ② | 从下单到交付完整链路，每步失败的处理机制 |
| [TBDS交易计费-优化设计方案与实现](tech-deep-dive/tbds-billing/TBDS交易计费-优化设计方案与实现.md) | ③ | 融合简化设计 + 公有云改造 + 技术亮点 |
| [TBDS交易计费-分布式一致性方案分析](tech-deep-dive/tbds-billing/TBDS交易计费-分布式一致性方案分析.md) | ④ | 是否需要 Saga 的技术选型分析 |
| [TBDS交易计费-抽象技术方案讲解](tech-deep-dive/tbds-billing/TBDS交易计费-抽象技术方案讲解.md) | ⑤ | 剥离业务细节，用通用技术术语讲解设计 |
| [TBDS交易计费-面试讲解指南](tech-deep-dive/tbds-billing/TBDS交易计费-面试讲解指南.md) | ⑥ | 面试讲解策略、话术模板、追问应对 |

---

## 四、🎤 技术分享与演讲 `talks-and-shares/`

| 文件 | 内容概要 |
|------|----------|
| [HDFS联邦开发看AI编程助手落地实践-讲稿](talks-and-shares/HDFS联邦开发看AI编程助手落地实践-讲稿.md) | 基于 HDFS 联邦实例的 AI 编程助手实践分享讲稿 |
| [HDFS联邦开发看AI编程助手落地实践.pptx](talks-and-shares/HDFS联邦开发看AI编程助手落地实践.pptx) | 对应 PPT 演示文件 |

---

## 五、关联知识库（独立仓库）

以下内容保留在各自独立仓库中，不重复复制：

| 仓库 | 路径 | 内容 | 文档数 |
|------|------|------|--------|
| **ai-knowledge** | `../ai-knowledge/docs/` | AI 全领域知识体系（Agent/LLM/RAG/MCP/伦理/搜索等 16 个方向） | 35+ 篇 |
| **ai-thinking** | `../ai-thinking/docs/` | AI 行业思考与职业规划（就业分析/冲击评估/转体制内等） | 17 篇 |

---

## 命名规范

- **目录名**：小写英文 + 连字符，如 `distributed-systems`、`tbds-billing`
- **文件名**：`主题前缀-具体内容.md`，如 `TBDS交易计费-面试讲解指南.md`
- **系列文档**：共享前缀，如 `TBDS交易计费-` 系列、`面试准备-` 系列、`面试亮点-` 系列
- **新增文档**：先确定属于哪个目录，文件名遵循同目录已有文件的命名风格

---

## 🚢 部署到远端服务器

```bash
# rsync 推送（自动排除不需要的文件）
rsync -avz \
    --exclude='.git/' \
    --exclude='.github/' \
    --exclude='__pycache__/' \
    --exclude='.workbuddy' \
    --exclude='vendor' \
    --exclude='python/.venv' \
    /Users/ziwh666/GitHub/knowledge-vault \
    root@182.43.22.165:/data/github/

# 服务器上拉取最新代码
git fetch origin && git reset --hard origin/main
```

> 💡 **免密推送**：建议先配置 SSH 密钥认证，执行一次 `ssh-copy-id root@your-server-ip` 即可免密。

---

## 统计

| 分类 | 文档数 |
|------|--------|
| 面试准备 | 12 篇 |
| 项目分析与决策 | 4 篇 |
| 技术深度研究 | 11 篇 |
| 技术分享 | 2 篇（1 md + 1 pptx） |
| **总计** | **29 篇** |
