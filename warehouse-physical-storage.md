# 物理存储三件套：分区 / 排序键 / TTL

> 第一遍：因果骨架深化与纠错。
> 仓库：`/home/cheche/Workspaces/Learning/HilPerformancePlatform`
> 数据栈：ClickHouse 25.3 + ReplicatedMergeTree + Distributed，1 分片 2 副本。

---

## 0. 一句话结论

物理存储三件套是 ClickHouse 在「写多查少、海量明细」场景下做时间空间换时间的核心杠杆：
**分区**让你整块跳过/整块删除、**排序键（稀疏索引）**让你只读命中行、**TTL** 让过期数据自动消失。

```
        JRC 车辆数据
            │ Flink 实时写入
            ▼
┌──────────────────────────────────────────────────────────┐
│  local 表 = ReplicatedMergeTree                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ PARTITION BY toYYYYMMDD(msg_receive_time)   ← 分区 │  │
│  │ ORDER BY (msg_receive_time, vin, ...)       ← 排序 │  │
│  │ TTL msg_receive_time + INTERVAL 7 DAY       ← 保留 │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
            │ 副本异步复制（2 replica）
            ▼
┌──────────────────────────────────────────────────────────┐
│  Distributed 虚表 = 查询入口                              │
│  分片键：perf 表用 rand() / fragments 表用 cityHash64(vin)│
└──────────────────────────────────────────────────────────┘
            │
            ▼
        业务查询
```

三件套的位置都在 local 表的 DDL 头部，三行相邻：
`warehouse/ddl/perf_stats_ods_ddl_v1.sql:24-27`。

---

## 1. 三件套因果骨架

### 1.1 分区（PARTITION BY）：物理独立 part，整块跳过/整块删

- **作用**：同一分区的数据物理上落在一组独立 part 文件里，与其它分区互不干扰。
  分区剪枝（partition pruning）靠 **MinMax 索引**——每个分区记下分区键的最小/最大值，
  查询时先用 WHERE 条件把"逻辑上不可能命中"的整块分区直接跳过，
  再进入存活的 part 做 granule 级读取。
- **失败层**：当 WHERE 里**不含分区键表达式所依赖的列**时，分区剪枝失效，
  优化器只能扫描所有分区（本仓库 TTL=7 天，分区数被天然限制在 ~7–8 个，所以"所有分区"是个有限小数）。
- **整块删**：`DROP PARTITION` 不重写 part，只把分区标记为 inactive 并在约 10 分钟内物理删除——
  相对 mutation 它是轻量操作，但**不是"瞬间完成"**：客户端返回快，物理删除是延迟的，
  行被标记 inactive 后即从查询结果消失。
- **代码证据**：
  - `warehouse/ddl/perf_stats_ods_ddl_v1.sql:24` `PARTITION BY toYYYYMMDD(msg_receive_time)`（ODS 三表统一）
  - `warehouse/ddl/perf_stats_ods_ddl_v1.sql:65,91` 同样分区键（`ods_control_cmd_local` / `ods_sys_perf_local`）
  - `warehouse/ddl/perf_stats_dwd_dws_design_v1.sql:41,74,129,167,238,272` DWD/DWS 改用预计算的 `dt` 列做分区键

> **纠错锚点**：`DROP PARTITION` 是"轻量但异步"，不是"原子/瞬间"。官方文档原文：
> *"tags the partition as inactive and deletes data completely, approximately in 10 minutes."*
> 想要"零重写地丢老数据"的标准答案不是手动 DROP，而是 **TTL 到期自动 drop**（见 1.3）。

---

### 1.2 排序键（ORDER BY）：稀疏主索引，只认前缀

- **作用**：数据在每个 part 内**物理上按 ORDER BY 列排序**。
  每 `index_granularity`（默认 8192）行记一条 mark，组成驻内存的扁平数组 `primary.idx`——
  这就是"稀疏主索引"。它的查询语义是**前缀匹配**：
  当 WHERE 命中 ORDER BY 的**第一列**时，CK 对索引 mark 做二分查找，跳过绝大多数 granule。
- **失败层（关键纠错点）**：当 WHERE 跳过第一列、只过滤第二列（如只给 `vin`）时，
  CK **不是"完全用不上索引"**，而是退化成 **generic exclusion search**——
  仍然在碰主索引，但效率极差，**当且仅当前导列基数低**时才有点用。
  本仓库 ORDER BY 第一列是 `msg_receive_time`（秒级 DateTime，超高基数），
  所以 vin-only 查询在实践中 ≈ 全 part 扫描。结论对，但"用不上索引"的说法太绝对。
- **另一个常见混淆（必须分开）**：
  - **分区剪枝**由 `PARTITION BY` 表达式驱动（这里依赖 `msg_receive_time`）；
  - **稀疏索引跳读**由 `ORDER BY` 前缀驱动。
  这是**两个独立层**。vin-only 查询同时触发两个失败：分区剪枝失效（因为分区表达式不含 vin）
  **且** 稀疏索引退化为 generic exclusion。不能把锅全甩给排序键。
- **代码证据**：
  - `warehouse/ddl/perf_stats_ods_ddl_v1.sql:24-27`
    ```sql
    PARTITION BY toYYYYMMDD(msg_receive_time)
    ORDER BY (msg_receive_time, vin, seq_num, dag_timestamp, module_name)
    TTL msg_receive_time + INTERVAL 7 DAY
    SETTINGS index_granularity = 8192;
    ```

---

### 1.3 TTL：后台 merge 自动删过期数据（不是逐行即时删）

- **作用**：表级 TTL `... + INTERVAL N DAY` 让 ClickHouse 在**后台 merge** 时自动删除过期行，
  无需人工 DROP、无需 cron。这是"零重写地丢老数据"的官方姿势。
- **失败层 / 风险**：
  1. TTL **不是逐行物理删**，而是 **part 粒度、惰性**：只在 TTL merge（一种特殊后台 merge，
     由 `merge_with_ttl_timeout` 默认 14400s/4h 节流）跑的时候才动手。
     所以一行数据满 7 天后可能还**短暂可见几小时**，直到下一次 TTL merge——
     TTL 是保留期的**上界**，不是精确时刻。
  2. TTL merge 一旦执行，**数据真的没了**，同表不可恢复。
- **两种删除路径**（这是深挖点，见第 4 节）：
  - 整 part 全过期 → 直接 drop 整个 part（便宜；建议开 `ttl_only_drop_parts=1`）。
  - 部分 row 过期 → 读 part、过滤过期行、写新 part（仍是 part 级，不是行级 reaper）。
- **代码证据**：
  - `warehouse/ddl/perf_stats_ods_ddl_v1.sql:26,67,93` 三张 ODS 表 `TTL msg_receive_time + INTERVAL 7 DAY`
  - `warehouse/ddl/perf_stats_dwd_dws_design_v1.sql:43,76,131,169` 四张 DWD 分钟表 `TTL dt + INTERVAL 30 DAY`
  - `warehouse/ddl/perf_stats_dwd_dws_design_v1.sql:240,274` 两张 DWS 日表 `TTL dt + INTERVAL 180 DAY`

> **关键协同**：本仓库刻意让 **PARTITION BY 表达式与 TTL 时间字段对齐**
> （`toYYYYMMDD(msg_receive_time)` ↔ `msg_receive_time + INTERVAL 7 DAY`）。
> 这样一天的数据天然聚在一个分区，整 part 过期时能整块 drop，几乎零开销。
> 这是"分区 + TTL"组合的标准最佳实践。

---

## 2. Q1–Q3 逐题评估（原答 → 肯定 → 精确纠正 → 正确因果）

### Q1：partition / orderby / ttl 各是什么？

**学员原答**：partition=按时间分区让数据好查；orderby=按条件排序先按什么后按什么；ttl=看保存多久。

**对的肯定**：方向正确——三件套的字面理解没跑偏。

**精确纠正**：
- partition 不是"让数据好查"这么软，它的硬机制是 **物理独立 part + MinMax 索引整块跳过 + 整块 DROP**。
- orderby 不是普通排序，它是**稀疏索引（index_granularity=8192）**，只认**前缀**；跳过第一列就大幅退化。
- ttl 不是"看保存多久"的被动元数据，它是**后台 merge 自动删过期数据**的主动机制，且是 part 粒度惰性执行。

**正确因果链**：
分区决定**哪些 part 整块不读**；排序键决定**存活 part 里哪些 granule 不读**；
TTL 决定**过期 part 何时被后台 merge 干掉**。三者叠加，才能在海量明细上做到"只读命中行 + 自动瘦身"。

---

### Q2：查询 B（带 `msg_receive_time` 时间窗 + vin）为什么比查询 A（只 vin）快？

**学员原答**：B 更快因为条件多 + 限定了接收时间区域，"分片查询 + 时序存储"，不限定就遍历所有 / 磁盘爆。

**对的肯定**：直觉对——限时间区域确实让 B 快得多。

**精确纠正（真正的因果）**：
1. **日期命中 `PARTITION BY toYYYYMMDD(msg_receive_time)`** → 触发**分区剪枝**，
   大部分天数的分区被整块跳过。
2. **`msg_receive_time` 是 `ORDER BY` 第一列** → 触发**稀疏索引二分查找**，
   在存活 part 里跳过绝大多数 granule，只读命中行。
3. **"分片查询"在本仓库根本没起作用**：DDL 头部明确写着 `1分片2副本`
   （`perf_stats_ods_ddl_v1.sql:4`，`perf_stats_dwd_dws_design_v1.sql:7`），
   只有 1 个分片，Distributed 表把所有行都送到那一个分片，没有跨分片并行可拿。
4. **A 慢的真正原因**：只用 `vin`（ORDER BY 第二列），跳过了第一列 →
   稀疏索引退化为 generic exclusion（前导列 `msg_receive_time` 高基数，所以 ≈ 全 part 扫）；
   同时分区剪枝也失效（分区表达式不含 vin）→ 所有分区都要进。

**正确因果链**：
B 快 = **分区剪枝（层 1）+ 稀疏索引前缀跳读（层 2）** 双双命中；
A 慢 = 两层**同时失效**，不是"分片没用上"，更不是"磁盘爆"。

---

### Q3：去掉分区查询会怎样？`retention_guard` 文件是干嘛的？

**学员原答**：去掉分区查询都要分区级扫描；retention_guard 没找到，猜防磁盘爆。

**精确纠正（三个坏结果）**：
1. **磁盘爆到只读**：没有分区约束 + 没有 TTL 自动清理，明细数据无限堆积，磁盘写满后 CK 进入只读，
   写入直接失败。
2. **查询跨 3 个月全扫超时**：分区剪枝失效后，跨大时间范围查询只能扫所有 part，超时是常态。
3. **想删老数据只能 `ALTER TABLE ... DELETE`（mutation）**：这是**重量级异步操作**，
   会**重写整个受影响 part**，官方明确说"a heavy operation not designed for frequent use"。
   这正是 TTL 存在的理由——用后台 merge 零成本删过期数据，而不是临时 mutation。

**`retention_guard` 的真实身份（证据）**：
- 文件：`warehouse/ddl/onboard_prod_shadow_ods_retention_guard_20260715.sql`
- **它不保护生产 ODS 表**，只改三张独立的 **shadow 表**（`ods_sys_perf_prod_shadow_local` 等，
  ZK 路径与生产表完全不同）。
- 干的事：`ALTER TABLE ... MODIFY TTL msg_receive_time + INTERVAL 14 DAY`
  （7d → 14d），把 shadow 表保留期延长到 14 天，**只是元数据修改，不移动/删除任何 part**。
- 目的（注释原文）：*"Temporary data-preservation guard while semantic percentile history is rebuilt."*
  在一次 prod-shadow/canary 测试期间，让 shadow 证据多活几天，便于重建百分位历史。
- 回滚文件：`onboard_prod_shadow_ods_retention_guard_rollback_20260715.sql`，
  把 TTL 从 14d 改回 7d，注释明确警告：如果有 >7 天的保留分区还在，回滚会让它下次 merge 时被删。
  **回滚是条件性元数据操作，不是数据恢复**。

**正确因果链**：
三件套任一层失效都会引发具体可观测的故障（磁盘只读 / 查询超时 / mutation 风暴）；
`retention_guard` 是一次性的 shadow 表 TTL 延长脚本，与生产 ODS、与 S3 canary（`onboard_s3_canary_*`）
都是**独立工件**，不要混为一谈。

---

## 4. TTL 风险深挖：删了之后要用怎么办？（学员敏锐追问）

这是整课最值得展开的点。TTL merge 跑完，数据**同表不可恢复**。本仓库的缓释手段是**分层保留**：

| 层 | 表 | TTL | 粒度 | 证据 |
|----|----|-----|------|------|
| ODS 原始明细 | `ods_*_local`（3 张） | **7 天** | 单条报文 | `perf_stats_ods_ddl_v1.sql:26,67,93` |
| DWD 分钟聚合 | `dwd_perf_*_minute_local`（6 张） | **30 天** | 车辆×分钟 | `perf_stats_dwd_dws_design_v1.sql:43,76,131,169` |
| DWS 日聚合 | `dws_perf_*_daily_local`（4 张） | **180 天** | 车辆×天 | `perf_stats_dwd_dws_design_v1.sql:240,274` |

**设计意图（刻意权衡）**：原始明细存储成本高、基数大 → 早删；
日级汇总体积小、价值高 → 长留 180 天。
"想看 3 个月前某辆车的报文级细节"本来就不在能力范围内，但"3 个月前车辆的日级性能趋势"完全可查。

**进一步的冷存储路径（事实存在）**：
- `warehouse/ddl/onboard_prod_shadow_ods_hot_cold_candidate_20260715.sql:8`
  `MODIFY TTL msg_receive_time + INTERVAL 2 DAY TO VOLUME 's3'`
  → 用 `TTL ... TO VOLUME/DISK` 把冷数据**搬迁**到 S3 而不是删除，这是 TTL 的另一种语义。
- `warehouse/ddl/onboard_s3_canary_create_20260715.sql`
  + `onboard_s3_canary_move_20260715.sql`
  → 独立的 storage_policy='hot_cold' 金丝雀表，注释明确"does not reference any production ODS table"，
  用来验证 S3 卷可用后再推广。

**恢复路径**：TTL 删掉的数据**无法从同表找回**。唯一可靠兜底是把数据**另存一份到更长寿命的表/系统**
（冷存储卷、对象存储、数仓），或者**到期前**用 `ALTER TABLE ... MODIFY TTL` 延长保留期——
这正是 `retention_guard` 那个脚本在做的事（虽然它只动 shadow 表）。

---

## 5. 仓库真相：1 分片 2 副本，Distributed 已就位待扩

- **拓扑事实**：`perf_stats_dwd_dws_design_v1.sql:7`、`scripts/ck_create_tables.sql:2`、
  `.joycode/rules/project-overview.md:42` 一致写明 **1 分片 2 副本**。
- **关键提醒**：集群拓扑（`remote_servers/<shard>/<replica>` 的 XML）**不在本仓库任何文件里**——
  仓库里所有 `.xml` 都是 `pom.xml` 或 logback 配置。拓扑只存在于 ClickHouse 服务端外部配置
  （`config.xml` / `metrika.xml`）。仓库只能从注释和 `{shard}/{replica}` 路径宏**推断**拓扑。
- **副本 vs 分片的硬语义**：
  - **副本**存**相同**数据，提供 HA/容错，**不增加存储容量、不增加写吞吐**。
  - **分片**存**不同**数据，提供水平扩展，**才增加容量和写能力**。
  - 当前 1 分片 → 写入和磁盘上限就是**单节点上限**，这是物理天花板。
- **Distributed 已就位**：所有 local 表外都套了 Distributed 虚表（`perf_stats_ods_ddl_v1.sql:31,72,98`），
  分片键也写了（perf 表用 `rand()`，fragments 表用 `cityHash64(vin)`）。
  想扩容，只需在服务端 `remote_servers` 加第二个分片，业务侧 DDL **几乎不用动**——
  这是为未来水平扩展预留的接口。

---

## 6. 下一课预告：`rand()` vs `cityHash64(vin)` 分片键代价

> 对照表：`warehouse/ddl/onboard_full_fleet_percentile_current_day_fragments_v3.sql:60-65`
> 用 `cityHash64(vin)`；`perf_stats_ods_ddl_v1.sql:31,72,98` 用 `rand()`。

**先抛三个易错点（下节课深挖）**：

1. **"co-locate vin 自动避免跨分片聚合"是错的**。
   ClickHouse Distributed 查询**默认永远走两阶段聚合**：每个分片算 partial 状态 →
   发给 initiator → initiator 跑 final merge。所以 `cityHash64(vin)` **本身**并不省聚合。
   真正能跳过 final merge 的条件是：`distributed_group_by_no_merge=2`
   或 `optimize_distributed_group_by_sharding_key` **且** `GROUP BY` 键 == 分片表达式。

2. **当前 1 分片下，rand() vs cityHash64 没有任何聚合代价差异**——根本不存在跨分片。
   这个选择是**为未来多分片集群做的前瞻设计**，不是当下的性能权衡。

3. **`cityHash64(vin)` 现在就能拿到的好处**是：
   - `optimize_skip_unused_shards`：单 vin 点查只命中一个分片；
   - 本地 `JOIN/IN` by vin（免 `GLOBAL JOIN`，CK 文档明确把它列为"按 key 分片"的首要收益）。

---

## 7. 掌握度标注

| 知识点 | 当前 | 下一课目标 |
|--------|------|-----------|
| 分区剪枝 + 整块 DROP 机制 | **practiced** | can-decide（能选分区键） |
| 稀疏索引前缀匹配 + generic exclusion | **practiced** | can-decide（能排 ORDER BY） |
| TTL 后台 merge 删除 + 分层缓释 | **practiced** | can-decide（能定 TTL 分层） |
| 1 分片 2 副本的物理天花板 | introduced | practiced |
| `rand()` vs `cityHash64(vin)` 分片键代价 | introduced | **can-decide**（下一课主攻） |

> **本轮纠错清单（务必内化）**：
> 1. `DROP PARTITION` 是"轻量但异步"，不是"瞬间/原子"。
> 2. 跳过 ORDER BY 第一列不是"用不上索引"，是退化成 generic exclusion；前导列高基数时 ≈ 全扫。
> 3. 分区剪枝（PARTITION BY + MinMax）和稀疏索引跳读（ORDER BY 前缀）是**两个独立层**，别混。
> 4. TTL 是 part 粒度、惰性、后台 merge 触发；同表删了不可恢复，靠分层 + 冷存储兜底。
> 5. `retention_guard` 只动 shadow 表的 TTL 元数据，与生产 ODS、与 S3 canary 都无关。
> 6. 当前 1 分片，"分片查询"在 Q2 里没起任何作用。
