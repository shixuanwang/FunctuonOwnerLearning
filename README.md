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
| [教材A·总纲](textbooks/A-master-textbook.html) | 数据闭环×数据基建全景：三大闭环/AD七段飞轮/仿真场景/基建栈/具身/能力图谱/职业跃迁/学习路线（107KB·11 mermaid） | — |
| [教材B·JoyDrive-Infra 0→100](textbooks/B-joydrive-infra-0to100.html) | 产品/工程/训练/评测/数据/基建六视角，吃透训练+评测（单 HTML+mermaid） | — |
| [教材C·JoyDriver 0→100](textbooks/C-joydriver-0to100.html) | 端到端模型本质+训练实战（SIMPL/StreamPETR/DDP+RDMA/ONNX 上车）（单 HTML+mermaid） | — |
| [教材D·ROVER5.0 发版与问题收敛黄金路径](textbooks/D-rover5-release-golden-path.html) | MR 体系/发版/问题收敛诊断+黄金路径+四流对齐+30-60-90 行动（单 HTML+mermaid） | — |
| [教材F·世界模型 0→100](textbooks/F-world-model-0to100.html) | NVIDIA Cosmos/GAIA/DriveDreamer/OccWorld/Vista 盘点+架构横评+转战补习（单 HTML+mermaid） | — |
| [教材G·Zero to Hero](textbooks/G-nn-zero-to-hero.html) | Karpathy nn-zero-to-hero 保姆式（micrograd→MLP→GPT），吸烟刻肺吃透神经网络底层（单 HTML+mermaid） | — |

## ⚠️ 机密提醒

本仓含 **JD 内部系统**的架构分析（表名、拓扑、设计权衡）。若仓库为 public，**强烈建议改为 private**，避免泄露内部设计细节。两个 JD 源码仓库（JoyDrive-Infra、HilPerformancePlatform）的**源代码不进入本仓**，只有学习用的分析与教学文档。
