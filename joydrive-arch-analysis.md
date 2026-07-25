# JoyDrive-Infra 架构分析（学习标杆）

> 同事的自动驾驶模型训练平台（MLOps），代号 JoyDrive。本地只读路径：`JoyDrive-Infra/`。
> 这是"产品架构 + 技术架构 + 数据架构三位一体"的活教材。下面是浓缩精华，完整逐文件分析在本地仓库 doc/ 里。

## 一句话定性

**领域驱动 + 状态机驱动 + 事件驱动**的成熟平台。栈：Go + Postgres(sqlc) + Next.js + Determined AI + Prefect + RKE2。

## 最该带走的 4 层价值

### 🏛 产品架构层
- **按"事实归属/事务边界/权限边界"切工作区，而不是按页面切**（`doc/architecture-design.html:447`）。8 个域：组织/数据/实验/评测/交付/资源/监控/工作台。
- **每个核心对象都有状态机，状态决定按钮、风险提示、可编辑范围**——"状态不是展示标签"（`doc/organization-prd.md:243`）。
- **变更单(ChangeRequest) + 审计血缘**：关键变更走 申请→审批→执行→失败回滚→撤销，审计只追加不改（`doc/organization-prd.md:291-313`）。

### ⚙️ 技术架构层
- **BFF"读侧聚合、写侧收敛"**：web-service 只做读聚合 + SSE 中继 + 权限透传，**写入只落单一领域服务**，不编排跨服务写入。
- **事件驱动三件套**：Outbox(与业务同事务) → `pg_notify`(唤醒 worker) → Reconciler(周期兜底) + 幂等键去重（`doc/architecture-design.html:576-596`）。
- **cluster-service 收口所有外部系统**（Determined/K8s/Prometheus），BFF 永不直连——这是扛千卡扩展的关键（`doc/k8s-component.md:1276-1306`）。

### 📊 数据架构层
- **完整闭环**：数据生产 → DatasetVersion → Experiment → TrainingJob → ModelPackage → EvalSet → EvaluationReport → DeliveryRequest，**每一步都挂血缘边**。
- **血缘校验是交付前置条件**：交付前必须能反查"哪个数据版本 + 哪个代码 commit + 哪次训练 + 哪个模型 + 哪份评测报告"。
- 数据服务**独立 PG 物理隔离**；artifact 三位置同步（北京/郑州 CFS + S3）。

### 🤖 AI Coding 层（最该偷的）
- **AGENTS.md 纵深防御**：6 条绝对禁令 → 每个 bash 强制 2 步预检打标签（`[READ]/[MODIFY]/[DELETE]...`）→ `BASH_ENV` 安全 wrapper 兜底。**不靠 LLM 自觉，靠工程约束把破坏力关进笼子**。
- **k8s-lessons.md 是"收敛器"**：问题 → 查 doc → 照做 → 解决后回写文档。文档即组织记忆。
- sqlc 代码生成（**绝不手改生成代码**）、pgtest 每测试独立 PG、✅/📝/❌ 三色追踪技术债。

## 偷师清单（具体到文件）

| 想学什么 | 去哪个文件 |
|---|---|
| 治理型 PRD 写法（最重要） | `doc/organization-prd.md` |
| 数据闭环全景 + 三层架构 | `doc/architecture-design.html`、`doc/domain-model.html` |
| 数据版本 / 血缘 / 资产管理 | `doc/data-artifact-registry-design.md` |
| 评测自动触发流水线 | `doc/evaluation-data-service.md` |
| 训练调度（Determined AI） | `doc/det-cli-integration.md` |
| 事件驱动三件套 / 一致性 | `doc/architecture-design.html` §数据流 |
| AI coding 工程化 | `AGENTS.md`、`doc/k8s-lessons.md`、`doc/k8s-security.md` |
| 服务调用链 / BFF / 组件 | `doc/k8s-component.md`（最长最全） |

## 对我的映射

我的盲区正好对标这里：**数据版本+血缘**（→ artifact registry）、**评测闭环**（→ evaluation-data-service）。
我的短板"影响力"的解药也在这：`organization-prd.md` 的"边界句+状态机+变更单+契约"就是让技术无法拒绝的治理结构——我那份数据资产设计落不了地，正该用这种写法重做。
