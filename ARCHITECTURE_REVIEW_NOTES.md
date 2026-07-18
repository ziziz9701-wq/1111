# Yuanpo 记忆系统架构评审（独立文档）

这份文件是对 `MEMORY_SYSTEM_DESIGN.zh-CN (1).md` 的架构补齐建议，不修改原文档内容。  
目标是把“已经写好的设计理念”补齐为更可执行、可验证、可控的架构约束。

## 先说结论

当前版本已经把“原料/正本/投影”、“来源优先级”、“多表面共享正本”讲得很清楚。  
下一步最关键的是：把同构数据对象的**语义类型系统**补齐，防止后续实现时出现“概念看起来像对了，系统却互相打架”。

## 需要补齐的架构缺口（按优先级）

### P0（必须先补）

1. 建立统一对象元模型（claim schema）

- 建议给所有长期对象加一套公共字段：  
`kind`（event / preference / relationship / subjectivity / state / handoff）、  
`claim_type`（observed / reported / inferred / authored / agreed / imported）、  
`truth_scope`（user / ai / relationship / external_fact / mixed）、  
`verification`（user_direct / segment_verified / tool_verified / unverified）、  
`valid_time`（发生时间）、`transaction_time`（写入时间）、  
`state`（active / deprecated / superseded / revoked / frozen）、  
`visibility`（normal / protected / restricted）。
- 当前版本有很多字段，但目前是按对象各自“自己定规则”，缺统一契约。  
没有统一契约时，不同模块会用各自标准解释同一记忆，后续最容易引发冲突判断错位。

2. 显式拆分“检索 -> 授权 -> 应用”流程

- `检索`：只负责找到候选与证据链。  
- `授权`：基于当前状态、敏感分区、用户意图判断是否可用于本轮。  
- `应用`：决定是否写入 prompt、更新 state、更新 response guidance。
- 现在设计把候选分数和可读性写在同一条线上，容易出现“能召回≠能用”的边界问题。

3. 冲突关系图（除了版本链）

- 除了 `supersede/revoke` 之外，再补关系边：  
`supports / contradicts / narrows / qualifies / derives_from / coexists_with`。
- 这可帮助评估“旧认知与新认知”是替代、补充还是并存，避免所有差异都落到最后写入时间上。

### P1（应尽快补）

4. 认知状态与人格状态分离到更细层

- 当前版本把 `values`、`drives`、`response tendency` 混在一起讨论，但容易被实现混淆。  
- 建议分离：
  - `persona_identity`（身份与价值定义）
  - `personality_style`（风格与表达倾向）
  - `transient_emotional_shape`（当前状态残影）
  - `long_term_affective_anchor`（长期影响）
- 防止一次高情绪事件覆盖长期人格基准。

5. 防自污染（anti self-confirmation）机制

- 引入两段式更新门槛：  
  - 本轮生成的候选必须能回溯到 user 源证据；  
  - 当缺口来自前轮 AI 回答时，只能写入“候选”，不能写正本。
- 额外加“反例要求”：高置信候选要触发一次外部交叉验证（历史片段/工具证据/时间窗口一致性），减少系统自举式强化。

6. 时态模型与衰减策略

- 增加 `effective_until / grace / decay`；  
- 明确“尚未确认”的记忆是否参与召回；  
- 提供“过期但未撤销”的处理状态（可检索但默认降权），避免误报。

### P2（迭代中补）

7. 来源与代理权限矩阵写成规则表

- 每种对象定义“谁可 propose / who can approve / who can author / who can apply”。  
- 让前端、Agent、后台候选在同一条规则上运行，减少“技术表面代写主观内容”。

8. 指标治理内建化

- 建议把以下指标列为必看监控：  
  - abstention_rate（该不该回答的判断率）  
  - evidence_grounded_recall（是否能回到源证据）  
  - conflict_resolution_success_rate（冲突关系解决成功率）  
  - protected_block_rate（敏感访问是否被正确拒绝）  
  - stale_projection_ratio（陈旧投影被召回比例）

## 一个可以直接落地的最小补丁（不改现有文档）

- 新建 schema registry（只读）  
  - `canonical_object_type`  
  - `allowed_actions_by_object`  
  - `authoring_authority_by_actor`
- 把当前 `memory/state/handoff` 全量都映射到该 registry 上。
- 在检索层新增 `authorize` 步骤和 `apply` 步骤；若失败，保留 `defer` 原因。
- 在版本账本加 `conflict_links`（数组）字段和 `resolution_strategy`。

## 上文问题与本补充映射

以下是你上一次反馈中核心问题在本补充中的对应位置，方便汇报与验收：

- “每种对象补齐”  
  - 对应：`P0-1 建立统一对象元模型（claim schema）`
- “分层不是为了多，而是职责单一”已写但要落成可执行边界  
  - 对应：`P0-1` + `最小补丁中的 schema registry`
- “候选要有来源、版本、可追溯”  
  - 对应：`最小补丁`、`P0-2`、`字段级建议（可在交付说明扩展）`
- “防止自我污染与循环确认”  
  - 对应：`P0-3 冲突关系图`、`P1-5 防自污染机制`
- “检索精度和 top1/top2 决策”  
  - 对应：`P0-2`（授权前后分离）与 `最小补丁`（defer 原因记录）
- “敏感上下文不能关键词开关”  
  - 对应：`P0-2` 的授权层 + `P2-7` 权限矩阵 + 指标中的 `protected_block_rate`
- “跨模型共享需权限与一致性”  
  - 对应：`P1-7 来源与代理权限矩阵` + `最小补丁中的 authoring_authority_by_actor`
- “长期情绪/当前状态混淆”  
  - 对应：`P1-4 认知状态与人格状态分离到更细层`
- “维护成本与评测闭环”  
  - 对应：`P2-8 指标治理内建化`

## 交付说明

这份文件与原始 `.md` 平行，便于你做架构评审和实现拆分。  
如果你要，我下一步可以直接再加一版“字段级补齐清单（含 JSON Schema 雏形）”，并按你当前实现的对象名（如 `memory`、`anchor`、`segment`、`handoff`）对齐字段映射。 