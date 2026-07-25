# Function Owner Learning

> cheche 的个人成长仓 —— 从数据闭环 PM 走向 **综合架构师 / Function Owner** 的学习记录。
> 配套两个**只读**对标仓库（本地）：
> - `JoyDrive-Infra` —— 同事的自动驾驶模型训练平台，架构标杆
> - `HilPerformancePlatform` —— 自己的 onboard 实时性能平台 + 数仓，主练习场

## 📌 元规则：我怎么学（所有 AI 协作的前提）

我是**结构与因果优先型、模型先行**的学习者。先建因果骨架（架构图/数据流/状态机/时序图），再挂术语。
- 教学顺序：用户结果 → owner → 操作对象 → 状态与数据流 → 失败层 → 原因 → 下一动作 → 完成证据 → 术语
- 两遍法 / 学习循环 / 掌握度三阶（introduced → practiced → can-decide）—— 详见 [认知契约](learning-contract.md)

## 📚 文档索引

| 文档 | 内容 | 掌握度 |
|---|---|---|
| [认知契约](learning-contract.md) | 我是谁、我怎么学 —— AI 协作元规则 | — |
| [职业整理与精进路线](career-roadmap.md) | 能力图谱、定位校准、4 阶段精进路线 | — |
| [JoyDrive-Infra 架构分析](joydrive-arch-analysis.md) | 三层架构（产品/技术/数据）+ aicoding 工程实践 | introduced |
| [数仓物理存储三件套](warehouse-physical-storage.md) | ClickHouse 分区/排序键/TTL —— 从因果模型到代码证据 | practiced |
| [设计能力校准报告](joyspace-design-calibration.md) | 22 篇 joyspace 文档实测：19 项能力图谱、影响力诊断、3 个月路线 | — |

## ⚠️ 机密提醒

本仓含 **JD 内部系统**的架构分析（表名、拓扑、设计权衡）。若仓库为 public，**强烈建议改为 private**，避免泄露内部设计细节。两个 JD 源码仓库（JoyDrive-Infra、HilPerformancePlatform）的**源代码不进入本仓**，只有学习用的分析与教学文档。
