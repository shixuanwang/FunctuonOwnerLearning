# 自动驾驶数据闭环 + 数据基建 + 具身智能数据闭环：研究简报（素材底稿 A）

> 综合 6 个 facet 的调研结果，作为后续总纲教材的素材底稿。所有论断保留来源 URL 供教材引用。
> 覆盖范围：自动驾驶数据闭环 / 数据引擎 → 仿真·场景库·世界模型 → 数据基建栈 → 流处理·OLAP·编排·MLOps → 具身智能数据基建 → 负责人能力框架。
> 日期：2026-07-26。

---

## 1. 领域全景：三大闭环的关系图

```
                         ┌─────────────────────────────────────────────────┐
                         │       自动驾驶数据闭环（成熟度最高）              │
                         │  采集→存储→选择/挖掘→标注→训练→仿真评测→部署回流 │
                         │  关键特征：量产车队 = 天然被动飞轮（冷启动已解）  │
                         └────────────┬──────────────────────┬───────────────┘
                                      │ 共享：仿真引擎        │ 共享：VLA/世界模型/
                                      │ 场景库标准(OSC2.0)    │       端到端范式
                                      │ sim2real              │       生成式补数据
                                      ▼                       ▼
        ┌───────────────────────────────────┐   ┌─────────────────────────────────┐
        │     数据基建栈（横向底座）         │   │   具身智能数据闭环（追赶者）     │
        │ lakehouse/版本化/血缘/质量/元数据  │   │ 遥操作→示教→VLA→sim2real         │
        │ feature store/流处理/OLAP/编排     │   │ 关键瓶颈：冷启动悖论、采集成本    │
        │ = 三个领域共享的"看不见的杠杆"      │   │ 被迫走"合成优先 + 主动采集"路线  │
        └───────────────────────────────────┘   └─────────────────────────────────┘
```

**一句话定位**：
- **自动驾驶数据闭环**是"被量产车队红利推着走的成熟飞轮"，已迭代到第 3 代（生成式 + 世界模型 + 主动误差挖掘）。
- **数据基建栈**是横跨 AD 与具身的横向能力，决定了上面飞轮的"转速上限"和"是否转得对"。
- **具身智能数据闭环**是"没有车队红利的追赶者"，被迫在合成数据、遥操作硬件、real-to-sim-to-real 上做激进创新，反过来正在倒逼 AD 重新审视"真实数据为王"的默认假设。

**三者的 6 个交叉点**（教材后续展开）：
1. **跨本体/跨传感器 schema 归一化**：OXE（22 种机器人本体统一 schema）↔ AD 多车型/多传感器套件归一化（同构问题）。
2. **生成式补长尾**：Waabi/Huawei/GraspVLA 共同趋势 —— 把 corner case 从"等"变成"造"。
3. **世界模型**：Wayve GAIA-3 / DriveDreamer / OccWorld ↔ NVIDIA Cosmos / Waabi World —— 渲染层（像素扩散）vs 决策层（占据自回归）分工。
4. **端到端范式**：π0 flow matching + action chunk ↔ AD 端到端轨迹生成的连续化输出。
5. **real-to-sim-to-real 双向闭环**：MIT RialTo ↔ AD 用真实接管/事故数据回灌仿真。
6. **数据基建层**：lakehouse/版本化/血缘/feature store —— AD 与具身共用同一套选型逻辑。

---

## 2. Top 公司/产品做法对照表

### 2.1 自动驾驶（按飞轮类型分组）

| 公司 | 数据引擎核心 | 自动标注 | 长尾挖掘 | 仿真/世界模型 | 评测/场景库 | 关键数字 |
|---|---|---|---|---|---|---|
| **Tesla** | shadow mode + trigger classifier（Hydra 多头）；"用未来标注过去" | 4D vector-space NeRF 重建，多车多趟联合优化 | 触发器上传 engineering clip | 自研世界模拟器 | 自有车队回归 | 8相机×36fps；1000车1h≈10亿帧；标注 ROI 100x；去雷达项目1周内自动标1万clips |
| **Waymo** |车队 learning + data mining 找 tail | **Offboard 3D Auto-Labeling**（CVPR 2021，object-centric 分治，点云序列） | 误差驱动 + 多城市分布覆盖 | Waymax + Sim Agents Challenge（~57.5万场景）；World Model | Waymo Open Dataset | 单车日产 ~25TB；200 TFLOPS 边缘 |
| **Cruise** | **Continuous Learning Machine (CLM)**：预测的"真值"由未来感知自监督生成 | 自监督（未来=真值） | **误差驱动全量挖掘**（预测-未来现实分歧显著→自动入库） | GM/Cruise 全闭环框架 | disengagement 回归 | 旧金山市区挑战场景频率是郊区46倍；U-turn <0.1% |
| **Waabi** | 生成式 AI 闭环优先 | 真实 log 正则化 | **非对称自对弈**自动生成让 student 失败的场景 | **Waabi World** + UniSim + Copilot4D + RTR + TraVL（旗舰路线） | Mixed Reality Testing | Volvo VNL 卡车零样本迁移；2025/06 Mixed Reality |
| **Momenta** | **"一个飞轮+两条腿"**：Mpilot 量产喂数据 / MSD L4 吃红利 | 自动化覆盖>90% 长尾 | 自动发现问题/记录/标注/训练/验证五环 | 飞轮大模型/世界模型 | — | 目标千亿公里 + 数百万长尾 |
| **小鹏** | 300+ 触发器，影子模式+定向触发 | 全自动标注，效率 45000x | 算法-人分歧自动上传（无人工） | — | — | 上传 22x、带宽 15x、集群利用率>90%、云端 10 EFLOPS、训练 1M→200M Clips、自研图灵芯片 |
| **理想** | **87万车影子模式** + 云端世界模型 | 主流水线内嵌脱敏 | 影子模式覆盖 + OTA 回灌 | 云端世界模型（ICCV'25《世界模型：从数据闭环到训练闭环》） | — | 云端 4.5 EFLOPS；VLA 依赖"数百万辆车构建的数据闭环" |
| **华为 ADS 4.0** | **WEWA**（World Engine + World Action） | 全流程标注 | **扩散生成 1秒1000极端场景**（AI教AI） | World Engine（Diffusion 扩散） | **25万场景库 + 500+类功能场景 + 200+测评指标** | 云端 7.5 EFLOPS；日行 3500万公里；模型 5天迭代；6亿公里仿真 |
| **百度 Apollo** | **文心大模型数据智能搜索引擎** + 数据中台（PaaS化） | 自动化标注 + 仿真场景生成 | LLM 场景挖掘 | 仿真场景生成 | PB级场景库（真实+虚拟） | 千万公里路测→大模型挖掘→反哺训练 |
| **Pony.ai** | Robotaxi 车队里程驱动 | — | 真实运营数据场景挖掘 | — | — | 截至2024-04 累计 3200万+ km；7代系统 650m 检测 |

### 2.2 学术前沿（数据引擎当产品来设计）

| 项目 | 核心贡献 | 路线 | 引用/影响 |
|---|---|---|---|
| **AIDE** (NEC Lab, CVPR 2024) | 用 VLM + LLM 自动化数据引擎 pipeline（active learning + self/semi-supervised labeling + error mining） | foundation model 替代手写规则+人工标注的新基座 | 被引 57+；VLADR Workshop Oral |
| **Data-Centric Evolution in AD** 综述（arXiv 2401.12888） | 系统梳理闭环 pipeline taxonomy（MagLev/Tesla/Waymo/Auto4D/VMA） | 给团队做 internal taxonomy/onboarding 的最佳教材 | AD 数据闭环的"官方"心智模型 |
| **Uber Auto4D** | 4D 空间 3D + 时间 1D，object size branch + motion path branch | offboard 4D 精标 | — |
| **VMA** | 众包多行程聚合 LiDAR 标注 + 人工验证闭环 | 众包地图路线 | — |

### 2.3 具身智能（与 AD 同构或对照）

| 公司/项目 | 数据引擎核心 | 采集硬件 | 仿真/sim2real | 关键数字 |
|---|---|---|---|---|
| **Physical Intelligence π0/π0.5** | flow matching + action chunk=50（~1秒），异构任务共训 | — | — | 10000 小时灵巧操作数据，7 构型 68 任务 |
| **OpenVLA / Octo** | 开源 VLA 基线（Llama2+SigLIP+DINOv2 / diffusion head） | — | — | OpenVLA 7B；Octo 27M/93M；970k/800k OXE 轨迹 |
| **Open X-Embodiment** | **跨本体 schema 归一化范本**（1M+轨迹，22 本体，21 机构） | — | — | 527 技能/约 16 万任务 |
| **DROID** | in-the-wild 广覆盖 | 50 采集员跨三洲 | — | 76k 轨迹，350h，564 场景，86 任务 |
| **ALOHA 2 / UMI / Mobile ALOHA** | 采集硬件产品化、可分发 | $27k 双臂 / **$400 手持 GoPro 夹爪** | diffusion policy 部署真机 | UMI 采集成本降一个数量级 |
| **银河通用 GraspVLA** | **纯合成数据 10 亿帧预训练 + 少样本微调**，零样本 Sim2Real | — | 统一 CoT：自回归感知 + flow matching | 七大泛化（物体/场景/光照/背景/姿态/高度/语言） |
| **Skild AI / 1X NEO** | 互联网人视频 + 大规模仿真做预训练 | — | 看人学 | Skild 数小时内 60-80% 任务成功率 |
| **Figure Helix 02** | 用 1000 小时关节数据替换 109504 行 C++ | 人体 mocap/穿戴 | — | 模型替代手写控制代码的硬数据 |
| **Tesla Optimus** | 复制 AD 车队飞轮思路，但**没有车队红利** | 公开招 Data Collection Operator；投入>$200万 | — | "被动车队飞轮 → 主动人工采集"的退化 |
| **MIT RialTo** | real-to-sim-to-real **双向闭环**：失败样本回灌 sim | 相机扫环境数字孪生 | 按需 sim→real | — |

---

## 3. 数据基建栈选型总表

> 横向底座：六层独立关注点。共识 = "**OpenLineage 当标准 + DataHub/OpenMetadata 当平台**"，不要让一个工具扛所有职责。

| 能力层 | top 方案 | 选型维度 | AD 场景结合点 |
|---|---|---|---|
| **lakehouse 表格式** | Iceberg（多引擎中立、隐藏分区、社区最活）；Delta（Databricks 绑定）；Hudi MoR（高频 upsert/CDC，Uber/ByteDance/Robinhood 验证）；Paimon（流式湖仓，Flink 原生） | 按写入模式选：纯追加→Iceberg；高频 upsert/GDPR 删除→Hudi MoR；流批一体+Flink→Paimon。XTable 对冲格式锁死 | 感知日志追加→Iceberg；车队元数据/标注迭代/删除→Hudi MoR；国产 EV 生态（理想/蔚来）→Paimon+Doris/Hologres |
| **数据版本化** | lakeFS（服务端、PB 级 Git 语义、零拷贝 branch，2025-11 收购 DVC）；DVC（客户端、ML 数据集+代码，保持开源）；Pachyderm（pipeline-centric） | 版本化对象存储数据湖表→lakeFS；版本化 ML 数据集+模型+pipeline→DVC；需内建血缘+容器化→Pachyderm | 场景挖掘结果写带 tag 的 branch（`scene_v3_corner_case_2026Q2`）；训练 job 通过 snapshot_id pin 死数据；模型卡反查 snapshot |
| **数据血缘** | **OpenLineage**（标准/协议，非平台）；Marquez（参考后端）；DataHub（消费+列级+实时）；Atlas（Hadoop+Ranger） | **先标准化事件，再选消费者**。三大引擎（Airflow/Spark/dbt）插 listener 产生事件 | AD pipeline 跨"采集→清洗→标注→挖掘→训练→评测"多引擎，OpenLineage 是把五段血缘拼成图的事件标准 |
| **数据质量** | Great Expectations（Python+Data Docs）；Deequ（Spark+AWS，约束 ML 推荐器）；Soda Core（SQL+YAML）；dbt tests；Databricks DQX；Pandera | 按栈选：Spark 大规模→Deequ；Python notebook 探查→GX；SQL 仓库+CI→Soda。质量断言回写 DataHub/OpenMetadata | "脏一行毁一个 epoch"，质量门禁前置到 ingest；WAP（Write-Audit-Publish）模式把质量内建到表格式 |
| **元数据治理** | **OpenMetadata**（统一+内置质量+MLflow 集成+部署 2-4 周）；**DataHub**（实时事件+MCP Server 面向 AI Agent+Actions Framework，部署 4-8 周）；Atlas/Amundsen 边缘化 | AI-forward 团队优先 DataHub/OpenMetadata 二选一，避开 Atlas/Amundsen | 模型/feature/训练数据/评测集四类资产治理；DataHub MCP Server 让 AD Agent 编程化查"这版模型用了哪些数据/质量分/owner" |
| **feature store** | Feast（开源自管）；Tecton（托管，声明式 transformation）；Databricks/SageMaker；Hopsworks（point-in-time 一致性强） | 算法栈强+成本敏感→Feast 自建；要托管+流式+point-in-time 保证→Tecton | **point-in-time 正确性是时序 ML 底线**：AD label 都是未来帧，无此保证 AUC 虚高且无法复现。也是"离线 batch 标注特征"与"车端在线推理特征"对齐的唯一可信层 |
| **流处理** | **Apache Flink**（首选，stateful+exactly-once+watermark+流批一体）；Spark Structured Streaming；Kafka Streams；Flink Serverless | AD 几乎只能选 Flink：多传感器对齐/去重/窗口聚合刚需 stateful；watermark 处理车端乱序 | 理想 Hologres+Flink 万亿级车联网：150万 RPS，10秒增量 ETL 框架，端到端延迟<2s；冷热比 2:1→5:1，计算成本 -40% |
| **消息队列** | Kafka（默认）；Pulsar（多租户+分层存储）；AutoMQ | 高吞吐+多消费者+生态→Kafka；跨地域/冷数据 S3 分层→Pulsar | 车端回流入口几乎都用 Kafka；AD 同一路段数据要同时喂感知训练/地图更新/回放测试 |
| **OLAP** | Doris（多维 JOIN，国产生态）；StarRocks（联邦查询）；ClickHouse（单表/压缩/成本）；Druid（高并发摄入/时序看板） | 评测段多 JOIN→Doris/StarRocks；采集段监控→Druid；高基数日志聚合→ClickHouse | 评测段（模型版本×场景×指标）→Doris/StarRocks；车队覆盖率看板→Druid；国产 AD 厂商 Doris 实践最多 |
| **批+交互** | Spark（重型 ETL/特征工程/MLlib）；Trino（ad-hoc 交互 SQL，比 Spark 快 2-30x） | 必须两者都有且分工：Spark 写/Trino 读，湖仓 Iceberg 统一避免数据搬运 | Spark 跑车端海量数据格式转换/抽帧/标注预处理；Trino 做评测工程师秒级 SQL 探查 |
| **编排** | Airflow（大厂默认，task-centric）；**Argo Workflows**（K8s 原生，GPU 任务）；**Dagster/Flyte**（asset-centric，data-centric）；Prefect（AI/LLM 集成） | AD 闭环"数据集/模型版本/评测报告"是一等公民→Dagster/Flyte；K8s 平台+大量 GPU 任务→Argo（Kubeflow 底层就是它） | 推荐 Argo/Dagster 而非 Airflow；范式分水岭 = task-centric vs data-centric |
| **ML 训练编排** | Ray（分布式训练+Ray Serve+RL，步间延迟关键）；Kubeflow（端到端 MLOps+KServe，治理强）；Metaflow（devex 优先，治理弱）；KubeRay 同集群共存 | 端到端大模型/RL→Ray；标准化多模型生命周期+多团队→Kubeflow；最务实 = Kubeflow 管 DAG + Ray 管算力 | 把 K8s 做成"Invisible Infrastructure"（KubeRay/Kubeflow SDK 抽象），算法团队不碰 YAML |
| **VLA/端到端基座**（具身侧） | π0/π0.5（闭源旗舰，flow matching）；OpenVLA（7B 开源）；Octo（轻量）；RT-2（学术起点） | 高频连续控制→flow matching；快速原型→OpenVLA token 化 | AD 端到端轨迹生成倾向 flow matching 类连续表达 |
| **遥操作/采集硬件**（具身侧） | ALOHA 2（$27k 双臂）；**UMI（$400 手持）**；Mobile ALOHA；Tesla 式 SOP | 单次成本×可分发性×动作空间匹配×场景多样性 | AD 定向 corner case 采集借鉴 UMI 低成本可分发思路 |
| **仿真/世界模型引擎** | AD 侧：CARLA/Waymax/Waabi World/NVIDIA DRIVE Sim；具身侧：Isaac Lab 3.0（OpenUSD）/MuJoCo/**Genesis（43M FPS）**/Newton | 单机 FPS 吞吐×sim2real 保真×OpenUSD 资产复用×与真实数据闭环对接 | OpenUSD 正成为 AD+具身共享的 3D 场景资产交换标准 |

---

## 4. 具身 vs 自动驾驶 数据闭环：可迁移 / 不可迁移

### 4.1 AD → 具身（AD 成熟，可向具身输出）

| 维度 | AD 状态 | 具身借鉴点 |
|---|---|---|
| **冷启动** | 量产车队天然飞轮，已解决 | 具身没有车队红利，被迫用仿真/人工遥操作 bootstrap；AD 的成熟度领先 3-5 年 |
| **shadow mode** | Tesla/小鹏/理想成熟范式 | 具身没有"人开车 AI 后台跑"的天然对照，只能退化为人遥操作 |
| **offboard 精标** | Waymo/Tesla/Auto4D 成熟 | 具身的"未来=真值"在多关节机器人控制中不直接成立 |
| **scene library + eval harness** | 华为 25万场景+200指标 / 百度 PB级 | 具身还在早期，可借鉴 AD 的测评指标体系化思路 |

### 4.2 具身 → AD（具身激进，倒逼 AD 重审假设）

| 维度 | 具身激进做法 | AD 可借鉴/重审点 |
|---|---|---|
| **跨本体 schema** | OXE：22 种本体统一字段 | AD 多车型/多传感器/多区域车队的"数据格式归一化"是同构问题，OXE schema 设计可直接迁移 |
| **生成式优先** | GraspVLA 10亿帧合成预训练 + 少样本微调跑通 | 挑战 AD "真实路采为金标准"的默认假设；AD 应评估世界模型/仿真在端到端预训练阶段能替代多少真实里程 |
| **采集硬件产品化** | UMI($400) / ALOHA2($27k) / Tesla 采集员 SOP | AD 定向 corner case 采集需要"采集设备/标注工具/SOP 产品化"思维，而非定制项目 |
| **real-to-sim-to-real 双向闭环** | MIT RialTo：失败样本回灌 sim | AD 应把真实接管/事故数据系统化回灌到仿真与世界模型，从单向供给升级为闭环自优化 |
| **video-to-action 预训练** | Skild/1X NEO：互联网人视频→机器人动作 | 对应 AD 的"人类驾驶视频→自动驾驶策略"，难点（动作空间映射/本体差异/视频-动作对齐）同构 |
| **flow matching 连续输出** | π0 action chunk=50 高频连续控制 | AD 端到端轨迹/规划输出在离散化 vs 连续化间取舍，flow matching 值得评估 |
| **数据采集即产品功能** | 1X NEO/Redwood/Figure Helix 家用机器人持续采 | 提示 AD 重新审视车内数据采集的隐私/合规边界 |

### 4.3 不可迁移（业务本质差异）

- **安全等级**：AD 涉及人身安全，真实数据为王 + 强监管 + 强合规；具身因数据成本高被迫合成优先 —— 这是两大领域最大路线分歧点。
- **车队红利**：AD 的量产车队是"永不疲倦的差异采样器"；具身没有等价物，只能人工遥操作补位。
- **物理形态**：机器人关节数据 ≠ AD 轨迹数据，Figure "1000 小时替换 10万行 C++" 不可直接照搬到 AD 数据量结论。

---

## 5. 负责人能力框架

### 5.1 工程能力分级（私有 rubric：introduced / practiced / can-decide）

> "能决策+能说服"是 staff 级分水岭，恰好定义 PM→架构师/Function Owner 的跃迁。可借鉴 CircleCI E1-E6 / Sprad IC1-IC6 / FT Engineering Progression 的能力维度（技术深度/影响范围/主动性/领导力），不必拘泥行业命名。

| 能力维度 | Introduced（介绍基本概念） | Practiced（有机会强化） | Can-decide（能在选型评审拍板） |
|---|---|---|---|
| **分布式存储/复制/一致性** | DDIA 阅读完成 | 在生产环境跑过 Iceberg/Hudi time travel | 能在 Iceberg vs Delta vs Hudi vs Paimon 评审上拍板 |
| **数据库内部** | CMU 15-445 跟完 | 写过/调优过 OLAP 查询 | CMU 15-721 完成，能判断 ClickHouse vs Doris vs StarRocks 选型 |
| **流处理** | Flink 概念课 | 写过 stateful Flink job | 能判断 watermark/checkpoint/exactly-once 在 AD 多传感器对齐场景的取舍 |
| **数据版本化/血缘** | OpenLineage 文档 | 接入过 DataHub/OpenMetadata | 能定 catalog/治理/SLA 边界，决策 centralized→federated 演进时机 |
| **数据质量** | GX/Deequ 教程 | 落地过 WAP 模式 | 能定"质量分"作为血缘图一等属性的治理架构 |
| **feature store** | Feast/Tecton 概念 | 部署过在线+离线 materialization | 能定 feature 归属（按实体/按团队）、在线/离线一致性 SLA、point-in-time 正确性保证 |
| **ML 平台/部署** | FSDL/Kubeflow/Ray | 跑过端到端训练-评测-部署 | 能在 Kubeflow vs Ray vs Metaflow 选型评审上有判断力，Kubeflow 管 DAG + Ray 管算力分层 |
| **AD 数据闭环架构** | arXiv 2401.12888 综述 | 落地过 trigger classifier / shadow mode | 能定"数据闭环 vs 模型闭环"两回路组织划分；设计闭环北极星复合指标 |
| **仿真/世界模型** | CARLA/Bench2Drive 跑过 | 落地过 log replay + re-simulation 双层回归 | 能判断 GAIA-3 / DriveDreamer / OccWorld 路线分水岭，定"渲染层 vs 决策层"分工 |
| **数据团队组织** | dbt Labs 三范式 | 推动过 centralized→federated 演进 | 能定数据平台/ML 平台/数据产品三层分工，feature store 谈判边界 |
| **PM→架构师桥接** | Data PM 概念 | 落地过 Principal Technical PM 工作 | 能在跨团队谈判（feature store 归属、SLA、发现机制）上代表平台方拍板 |

### 5.2 Top 公司 JD 要点（自评基线）

| 公司 | 岗位 | 硬性门槛 | 技术栈倾向 | URL |
|---|---|---|---|---|
| **NVIDIA** | Autonomous Vehicle Dataloop Lead | CS/EE 学士 + 3年+ 软件/数据工程经验 | **C/C++/Python/Java + ML/DL + CV + 分布式计算 + Azure/AWS/GCP** | https://www.jobzmall.com/nvidia/job/autonomous-vehicle-dataloop-lead |
| **AT&T** | Principal Product Technology Manager (AI & Data Products) | — | 技术 PM 贴近架构/工程/产品 | （Monte Carlo 引用） |
| **LinkedIn** | Technical Product Owner - PIM & Data Architecture | 5+ 年经验 | 薪酬 12.6-18.9 万美元 | （LinkedIn JD） |

**NVIDIA 强调 C++ + 底层硬件结合**，与 Waymo/Zoox 偏 Python/算法的画像不同 —— 选型时先定位自己公司的技术栈倾向。

### 5.3 PM → Function Owner 跃迁路径

```
PM (4年数据闭环) → 技术型 Data PM → Principal Technical PM / Function Owner → Data Architect / Platform Tech Lead
                      ↑                              ↑                                  ↑
              补强：分布式系统理解        补强：架构级技术 fluency           补强：analytics 深度
                    feature store 谈判         跨团队组织设计                  闭环北极星指标体系
```

**关键补强项**：
1. 分布式系统理解（DDIA 奠基）
2. 架构级技术 fluency（CMU 15-721 OLAP/Lakehouse）
3. analytics 深度（feature store point-in-time、闭环复合指标）
4. 跨团队组织设计（centralized→federated、平台即产品）

---

## 6. 学习资源清单（分阶段）

### 6.1 地基阶段（introduced → practiced）

| 类型 | 资源 | 为什么读 | URL |
|---|---|---|---|
| 书 | **Designing Data-Intensive Applications**（Kleppmann，第二版） | 数据基建负责人的"宪法"。分布式存储/复制/一致性/流/批/OLAP 的心智模型全部在此，是所有架构讨论的共同语言。第二版覆盖 Lakehouse 新范式 | — |
| 课程 | **CMU 15-445/645**（Andy Pavlo） | "THE BEST 大学数据库课程"，数据库内部 | — |
| 论文/综述 | **Data-Centric Evolution in Autonomous Driving**（arXiv 2401.12888） | AD 数据闭环的学术总纲：7 组件架构、Tesla/NVIDIA/Waymo 数据引擎、auto-labeling 演进、dataset 规模对比。是建立全局认知的首选 | https://arxiv.org/html/2401.12888v1 |

### 6.2 进阶阶段（practiced → can-decide）

| 类型 | 资源 | 为什么读 | URL |
|---|---|---|---|
| 课程 | **CMU 15-721**（Andy Pavlo, Spring 2024） | 进阶 OLAP/Lakehouse/分布式查询处理/查询优化。**直接对应 AD 数仓选型**（ClickHouse vs Iceberg vs Delta）的底层判断力 | https://15721.courses.cs.cmu.edu/spring2024/schedule.html |
| 课程 | **Full Stack Deep Learning**（Pieter Abbeel） | ML 基础设施/部署/监控/Kubernetes 全生命周期，开源免费。补 ML 部署短板，理解 ML 平台团队的工作语言 | https://fullstackdeeplearning.com |
| 书 | **Designing Machine Learning Systems**（Chip Huyen，Stanford CS329S 教材） | Production ML 的"圣经"。AD 数据闭环本质是超大尺度 Production ML 系统 | — |
| 公司博客 | **CodeCompass Substack: Tesla Data Engine Trigger Classifiers** | Tesla 主动选数机制（trigger classifier + shadow mode）最技术化的拆解 | https://codecompass00.substack.com/p/tesla-data-engine-trigger-classifiers |
| 行业报告 | **State of Workflow Orchestration Ecosystem 2025**（pracdata.io） | 编排工具 2025 格局最权威横向对比：下载量/活跃度/任务vs数据范式分水岭/event-driven 趋势 | https://www.pracdata.io/p/state-of-workflow-orchestration-ecosystem-2025 |
| 战略分析 | **TheDataGuy: Open-Source Data Governance Frameworks**（2025-08） | OpenMetadata/DataHub/Atlas/Amundsen 四维度并列对比，含 MCP Server、MLflow 集成等 2025 最新能力 | https://thedataguy.pro/writing/2025/08/open-source-data-governance-frameworks/ |

### 6.3 AD 数据闭环范式（专项）

| 类型 | 资源 | 为什么读 | URL |
|---|---|---|---|
| 公司博客 | **Cruise: Continuous Learning Machine**（Sean Harris, 2020） | 数据闭环 PM 必读范式 —— 把"未来=真值"的自监督 auto-labeling + 误差驱动主动学习讲得最透彻 | https://medium.com/cruise/cruises-continuous-learning-machine-predicts-the-unpredictable-on-san-francisco-roads-30d60f4c691b |
| 深度解析 | **Tesla AI Day 2021 - Part 2**（Towards Data Science） | 4D vector-space 标注、NeRF 场景重建、shadow mode 触发回传的开山公开讲述，量化指标齐全（100x、10亿帧、1万clips/周） | https://towardsdatascience.com/tesla-ai-day-2021-review-part-2-training-data-how-does-a-car-learn-e8863ba3f5b0/ |
| 论文 | **Offboard 3D Object Detection from Point Cloud Sequences**（Qi et al., CVPR 2021） | Waymo auto-labeling 开山作，定义"offboard 精标"高价值数据资产类别 | https://openaccess.thecvf.com/content/CVPR2021/papers/Qi_Offboard_3D_Object_Detection_From_Point_Cloud_Sequences_CVPR_2021_paper.pdf |
| 论文 | **AIDE**（CVPR 2024） | VLM/LLM 自动化数据引擎 pipeline 的前沿，foundation model 成为新一代基座。可作数据闭环平台下一代架构的 RFP | https://openreview.net/forum?id=7piDzw05kh |
| 访谈 | **How Waymo Is Using ML**（Scale AI 对 Dmitri Dolgov） | Waymo co-CEO 亲述 data mining 找 tail、auto-labeling 替代人工、仿真补人类示例盲区 | https://learn.scale.com/public/blogs/how-ml-waymo-building-scalable-autonomous-driver-dmitri-dolgov |
| 公司官方 | **Momenta: 重新定义无人驾驶关键路径** | "一个飞轮+两条腿" + "闭环自动化五环（发现/记录/标注/训练/验证）" → 可直接落地为数据平台五个产品模块 | https://www.momenta.cn/article/54.html |
| 公司官方 | **小鹏1024科技日** + 东方财富研报 | 给出可量化数据基建北极星指标（触发器数、45000x 标注倍率、上传22x、带宽15x、10 EFLOPS、200M clips） | https://www.xiaopeng.com/news/company_news/4537.html ; https://pdf.dfcfw.com/pdf/H3_AP202408201639367464_1.pdf |
| 公司官方 | **华为乾崑智驾 ADS** | 扩散模型1秒生成1000极端场景、25万场景库+200测评指标、7.5 EFLOPS/5天迭代 —— "生成式造难例"与"scene library+eval harness"样板 | https://auto.huawei.com/cn/ads |

### 6.4 仿真/世界模型段（专项）

| 类型 | 资源 | URL |
|---|---|---|
| 公司博客 | **GAIA-3: Scaling World Models to Power Safety and Evaluation**（Wayve） | https://wayve.ai/thinking/gaia-3/ |
| 工程博客 | **Diagnosing the Long Tail**（Mobileye Meteor + Genario） | https://www.mobileye.com/blog/diagnosing-the-long-tail-how-mobileye-turns-edge-cases-into-targeted-training/ |
| 综述论文 | **Beyond Behavior Cloning: A Survey of Closed-Loop Training**（NVIDIA 2025） | https://research.nvidia.com/publication/2025-12_beyond-behavior-cloning-autonomous-driving-survey-closed-loop-training |
| 公司官方 | **How Waabi World works + Asymmetric Self-Play** | https://waabi.ai/insights/how-waabi-world-works ; https://waabi.ai/research/selfplay |
| 论文 | **Bench2Drive**（NeurIPS 2024） | https://arxiv.org/abs/2406.03877 |
| 论文 | **Raw2Drive**（NeurIPS 2025）+ **PlanT 2.0**（2025）—— 仿真内 RL 的对与反 | https://openreview.net/forum?id=CAz7UGRdLs |
| 工程博客 | **Accelerating AV Simulation with Neural Reconstruction and World Foundation Models**（NVIDIA） | https://developer.nvidia.com/blog/accelerating-av-simulation-with-neural-reconstruction-and-world-foundation-models/ |
| 标准文档 | **ASAM OpenSCENARIO 2.0** + Applied Intuition 解读 | https://www.asam.net/standards/detail/openscenario-dsl/ ; https://www.appliedintuition.com/blog/asam-openscenario-v2 |
| 论文+开源 | **DriveDreamer / OccWorld**（世界模型两大技术路线） | https://arxiv.org/abs/2309.09777 ; https://github.com/wzzheng/OccWorld |
| 工程博客 | **Closed-Loop Log Replay vs Re-Simulation**（Applied Intuition） | https://www.appliedintuition.com/blog/closed-loop-log-replay |
| 开源资源 | **Awesome-Data-Centric-Autonomous-Driving**（GitHub awesome-list） | https://github.com/LincanLi98/Awesome-Data-Centric-Autonomous-Driving |

### 6.5 数据基建选型（专项）

| 类型 | 资源 | URL |
|---|---|---|
| 深度对比 | **Onehouse: Iceberg vs Delta vs Hudi Feature Comparison**（2025-10） | https://www.onehouse.ai/blog/apache-hudi-vs-delta-lake-vs-apache-iceberg-lakehouse-feature-comparison |
| 工程文 | **InfoQ: Building Reproducible ML Systems with Apache Iceberg** | https://www.infoq.com/articles/reproducible-ml-iceberg/ |
| 公司博客 | **Apache Hudi MoR Comparison** + Uber/Robinhood/ByteDance 案例 | https://hudi.apache.org/blog/2025/07/21/mor-comparison/ |
| 公司博客 | **DVC Joins lakeFS — Your Questions Answered**（2025-11） | https://dvc.org/blog/dvc-joins-lakefs-your-questions-answered/ |
| 公司博客 | **DataHub: Open Source Data Lineage** | https://datahub.com/blog/open-source-data-lineage/ |
| 教程 | **OpenLineage + Airflow + Marquez**（Astronomer） | — |
| 选型指南 | **Tecton: Choosing Feature Store** + Databricks Point-in-Time 文档 | https://resources.tecton.ai/hubfs/Choosing-Feature-Solution-Feast-or-Tecton.pdf?hsLang=en |
| 公司讲话 | **Cruise @Scale: ML Infrastructure for AV**（YouTube） | — |

### 6.6 流处理/编排/OLAP（专项）

| 类型 | 资源 | URL |
|---|---|---|
| 工业实践 | **理想汽车 Hologres+Flink 万亿级车联网** | https://developer.aliyun.com/article/1686564 |
| 工业实践 | **蔚来 Flink 实时计算平台**（Flink Forward Asia 2021） | https://developer.aliyun.com/article/891962 |
| 官方文档 | **Flink Stateful Stream Processing**（checkpoint/watermark/exactly-once） | https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/stateful-stream-processing/ |
| 公司对比 | **Confluent: Kafka vs Pulsar** | https://www.confluent.io/compare/kafka-vs-pulsar/ |
| 技术对比 | **Onehouse: Spark vs ClickHouse vs Presto vs StarRocks vs Trino** | https://www.onehouse.ai/blog/apache-spark-vs-clickhouse-vs-presto-vs-starrocks-vs-trino-comparing-analytics-engines |
| 技术博客 | **ayedo.de: MLOps mit Kubeflow vs Ray** | https://ayedo.de/en/posts/cloud-native-ai-pipelines-mlops-mit-kubeflow-vs-ray/ |
| 工程博客 | **Tesla Data Engine**（taivo.ai） | https://taivo.ai/tesla-data-engine |
| 工业实践 | **阿里巴巴 EV Lakehouse**（Paimon+Doris+Hologres） | https://www.alibabacloud.com/blog/electric-vehicle-data-revolution-how-real-time-lakehouse-architectures-solve-automotive-big-data-challenges_602044 |

### 6.7 具身智能（专项）

| 类型 | 资源 | URL |
|---|---|---|
| 论文+项目 | **Open X-Embodiment**（CoRL 2023）—— 跨本体 schema 范本 | https://robotics-transformer-x.github.io/ ; https://arxiv.org/abs/2310.08864 |
| 论文 | **π0: VLA Flow Model**（Physical Intelligence） | https://arxiv.org/html/2410.24164v1 |
| 论文+数据集 | **DROID**（RSS 2024） | https://droid-dataset.github.io/ ; https://arxiv.org/abs/2403.12945 |
| 项目+论文 | **ALOHA 2** + **UMI** | https://aloha-2.github.io/ ; https://arxiv.org/abs/2405.02292 ; https://umi-gripper.github.io/ ; https://arxiv.org/abs/2402.10329 |
| 论文 | **RT-2** / **OpenVLA** / **Octo** | https://arxiv.org/abs/2307.15818 ; https://arxiv.org/abs/2406.09246 ; https://arxiv.org/abs/2405.12213 |
| 公司发布 | **银河通用 GraspVLA**（合成数据预训练路线） | https://hub.baai.ac.cn/view/42551 ; https://finance.sina.com.cn/tech/digi/2025-01-10/doc-ineenpiz5566961.shtml |
| 公司博客 | **Skild AI: Building the General-Purpose Robotic Brain** + NVIDIA 案例 | https://www.skild.ai/blogs/building-the-general-purpose-robotic-brain ; https://www.nvidia.com/en-us/case-studies/skild-ai/ |
| 公司博客 | **Figure Helix 02** | https://www.figure.ai/news/helix-02 |
| 行业分析 | **Roborax: Embodied AI Data Flywheel** | https://www.roborax.ai/embodied-ai-data-flywheel-physical-ai/ |
| 技术报道 | **MIT CSAIL RialTo** | https://news.mit.edu/2024/precision-home-robotics-real-sim-real-0731 |
| 开源+benchmark | **Genesis** + **NVIDIA Isaac Lab 3.0** | https://github.com/Genesis-Embodied-AI/genesis-world ; https://huggingface.co/blog/nvidia/state-of-simulation-for-physical-ai |

### 6.8 组织/PM 跃迁（专项）

| 类型 | 资源 | URL |
|---|---|---|
| 公司工程博客 | **Uber Engineering: Michelangelo / Palette Meta Store** | https://www.uber.com/us/en/blog/palette-meta-store-journey/ |
| 组织设计 | **dbt Labs: The Right Structure for a Scalable Data Team** | — |
| 行业新闻 | **InfoQ: Uber Hive Federation**（PB 级去中心化） | https://www.infoq.com/news/2026/04/uber-hive-decentralized-data/ |
| 平台工程 | **platformengineering.org: What is a Data Platform Engineer** | https://platformengineering.org/blog/what-is-a-data-platform-engineer |
| 公司博客 | **Monte Carlo: What good Data PMs do** | https://montecarlo.ai/blog-what-good-data-product-managers-do-and-why-you-probably-need-one |

---

## 7. 关键洞察与反直觉发现

### 7.1 闭环的"物理不变量"决定自动化天花板

**洞察**：Cruise CLM 的"未来=真值"是预测/规划类任务自监督的根基；Waymo Offboard 的"未来帧 + 跨帧聚合"是检测类任务精标的根基。**先找到你业务里那个"可自监督的不变量"，再谈自动化** —— 这是数据闭环 PM 最该吃透的范式。
> 来源：https://medium.com/cruise/cruises-continuous-learning-machine-predicts-the-unpredictable-on-san-francisco-roads-30d60f4c691b

### 7.2 触发器是车云契约层，本质都是"差异采样器"

**反直觉**：Cruise 用误差、Tesla 用 shadow 分歧、小鹏用 300+ 触发器 —— 看似不同，本质都是把"何时回传"从无穷多手写规则收敛到少量统一信号。**trigger classifier 是产品功能，值得作为独立子模块立项设计**，而非散落在各团队。
> 来源：https://codecompass00.substack.com/p/tesla-data-engine-trigger-classifiers

### 7.3 数据基建北极星不是"标注条数"而是复合指标

**反直觉**：单看数据量会误判飞轮健康度。真正反映飞轮的复合指标 = **触发器数 + 自动化覆盖率 + 误差驱动采样命中 + 上传倍数 + 训练带宽 + 集群利用率 + 标注倍率 + 训练 clip 规模 + OTA 回灌周期**。这套体系比"我们采了 X 亿公里"更接近本质。
> 来源：https://www.xiaopeng.com/news/company_news/4537.html

### 7.4 长尾靠"造"而非只靠"等"是范式切换

**洞察**：华为扩散模型 1秒1000场景、Waabi UniSim/Waabi World、Cruise 自动 error mining —— 共同趋势是**生成式 + 主动挖掘双轮驱动**。当真实采集边际成本高（卡车/长尾）时，"生成式补数据"是比"扩车队"更陡的 scaling 曲线。
> 来源：https://auto.huawei.com/cn/ads ; https://waabi.ai/research/unisim

### 7.5 合成数据正在从"补充"变成"预训练主力"

**反直觉**：AD 长期以真实路采为金标准，对合成数据保守；具身因数据成本极高被迫"合成优先"，GraspVLA 用 10 亿帧合成数据预训练跑通零样本 Sim2Real。**AD 需重新评估：自己的世界模型/仿真在"合成数据预训练 + 少量真机微调"范式下能替代多少真实里程** —— 这是当前两大领域最大的路线分歧点。
> 来源：https://hub.baai.ac.cn/view/42551

### 7.6 飞轮的商业架构比技术架构更关键

**洞察**：Momenta 的"Mpilot 量产喂数据 / MSD L4 吃红利"、理想的"87万车影子模式 + OTA 回灌"，都是**用低成本量产场景养高价值算法资产的双产品线互馈**。做数据闭环 PM 不能只看技术，要先看商业架构是否支撑飞轮转动。
> 来源：https://www.momenta.cn/article/54.html

### 7.7 Feature store 是数据平台与 ML 平台的"谈判地"

**反直觉**：负责人绕不开的跨团队接口。归属（基础设施 vs 特征定义）、在线/离线一致性 SLA、发现机制三件事必须**提前写死**，否则反复内耗。Uber Palette 按实体组织、Pinterest 两工程师逐步标准化 —— 都可迁移。
> 来源：https://www.uber.com/us/en/blog/palette-meta-store-journey/

### 7.8 脱敏/合规要前置嵌入主流水线

**反直觉**：规模化下唯一可行的合规姿势是**前置嵌入**而非事后补丁。理想把脱敏放在"采集→脱敏→训练→OTA"链路里 —— 量产车企做数据闭环时这是必须照搬的设计。
> 来源：https://pdf.dfcfw.com/pdf/H3_AP202408201639367464_1.pdf

### 7.9 scene library + eval harness 是被低估的高杠杆子系统

**洞察**：华为 25万场景 + 200测评指标、百度 PB 级场景库 —— 它们决定**"飞轮转得对不对"**，而不只是"转得快不快"。建议把"场景库 + 测评指标体系"作为独立产品线立项，而非附属工具。配合 ASAM OpenSCENARIO 2.0 做可互换防工具锁定。
> 来源：https://auto.huawei.com/cn/ads ; https://www.asam.net/standards/detail/openscenario-dsl/

### 7.10 real-to-sim-to-real 双向闭环优于单向 sim-to-real

**反直觉**：AD 已解决冷启动（量产车队），但 real-to-sim（用真实数据反哺仿真/世界模型）仍在早期。**MIT RialTo 的"按需数字孪生 + 失败样本回灌"是高度可迁移的方法论** —— AD 应把真实接管/事故数据系统化回灌到仿真与世界模型，从单向供给升级为闭环自优化。
> 来源：https://news.mit.edu/2024/precision-home-robotics-real-sim-real-0731

### 7.11 point-in-time 正确性是时序 ML 的底线

**反直觉**：feature store 的核心价值不是缓存，而是"训练时只拿事件时间之前的特征值"。**AD 感知/预测模型的 label 都是未来帧**，没有这个保证，训练集会"偷看未来"导致 AUC 虚高且无法复现。选型时这是不可妥协的硬指标。
> 来源：https://resources.tecton.ai/hubfs/Choosing-Feature-Solution-Feast-or-Tecton.pdf?hsLang=en

### 7.12 工业实践与学术研究存在脱节

**反直觉**：NVIDIA《Beyond Behavior Cloning》综述指出当前学术研究与工业实践存在脱节。**选论文时要按"动作生成×环境响应×训练目标"三轴评估落地成本**，而非只看 SOTA 分数 —— 这是规划仿真段技术栈和判断"哪些论文可落地"的元地图。
> 来源：https://research.nvidia.com/publication/2025-12_beyond-behavior-cloning-autonomous-driving-survey-closed-loop-training

---

## 附录：6 个 facet 的置信度

| Facet | 置信度 |
|---|---|
| 自动驾驶数据闭环 / 数据引擎 | 0.88 |
| 仿真 / 场景库 / 长尾挖掘 / sim2real / 世界模型 | 0.86 |
| 数据基建栈（lakehouse/版本化/血缘/质量/元数据/feature store） | 0.88 |
| 流处理 / OLAP / 编排 / MLOps | 0.86 |
| 具身智能 / 机器人数据基建 | 0.88 |
| 负责人能力框架 / 职业阶梯 / 学习资源 | 0.86 |

**整体置信度**：0.87。主要不确定性集中在：(1) 各公司公开数字的真实性（部分为厂商宣传）；(2) 具身智能数据闭环的成熟度因领域新而样本少；(3) 选型维度的"AD 场景结合点"是基于公开资料的推理而非一线验证。
