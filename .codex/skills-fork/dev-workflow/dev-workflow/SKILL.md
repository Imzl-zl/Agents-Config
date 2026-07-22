---
name: dev-workflow
description: Single-entry workflow for turning ideas, bugs, plans, and ambiguous efforts into verified outcomes.
disable-model-invocation: true
---

# Dev Workflow - 单入口编排器

用户只调用一次 `/dev-workflow`。本 skill 保持为整个会话的编排 frame；设计、原型、计划、执行、调试和验证只是按需读取的内部 stage 文档，不是可独立调用的 skill。

## 宿主调用契约

- 本文件是唯一默认 user-invoked 入口，目录名和 frontmatter 名都为 `dev-workflow`。
- 进入阶段时只读取对应的 `../stages/<stage>/STAGE.md`；阶段结束时返回 state delta，由当前编排 frame 合并。
- “返回编排器”表示继续执行已加载的本文件，绝不再次 invoke `/dev-workflow`，也不要求用户输入下一条 slash command。
- stage 不直接调用 sibling stage；它只能返回 `StageStatus` 和下一阶段建议，实际路由由本编排器决定。
- fork 内其他目录的 `REFERENCE.md` 是保留的设计参考，不是 slash command，也不参与运行。
- `.codex/reference-skills/` 和根目录上游源码同样只读，不参与运行。
- explicit-only 是有意取舍：重工作流不会吞掉普通问答；用户需要在开始时调用一次 `/dev-workflow`。不增加无法调用 user-only 入口的浅层 auto-router。

## WorkflowState - 唯一设计状态

路由、等待用户、切换 owner、进入原型和阶段返回前，先更新同一个 `WorkflowState`：

```yaml
version: dev-workflow/v0.4
goal: 一句话目标
intent: answer | research | review | bug | executable-plan | partial-plan | clear-change | idea | fog
phase: route | discovery | design | prototype | plan | execute | debug | verify | stop
constraints: []
decision_mode: Assisted | Owner | Mixed
owners:                    # 只列当前范围内的领域
  business: model | user
  architecture: model | user
  ui: model | user
ui_mode: Prototype-Assisted | Design-Owner | Direct | Text-Only | not-applicable
decisions:                 # id 创建后不复用
  - { id: D1, domain: business, owner: user, decision: "...", reason: "..." }
assumptions:
  - { id: A1, domain: architecture, assumption: "...", reversible: true }
open_nodes:
  - { id: D2, domain: ui, owner: user, question: "...", depends_on: [D1], status: blocked }
fog: []                    # 已知在范围内、但还不能准确表述的问题
prototype:
  parent_id: null
  status: none | pending | answered
  artifact: null
  result: null
current_node: null
awaiting: none | user | prototype | debug | verification
return_point:
  phase: null
  node: null
  evidence_ref: null
consensus: not-required | pending | confirmed
readiness: continue-design | stop | transient-execution | durable-plan | epic-plan
next: 一个明确动作
task_root: .tasks
checkpoint: session | <task_root>/<task>/DESIGN.md | <task_root>/<task>/PROGRESS.md#design-drift
```

规则：

1. `open_nodes` 保存所有未决设计，不只保存“重大”问题；依赖满足的节点才标为 `ready`。
2. `owners` 和 `ui_mode` 是恢复字段，不能只埋在自由文本结论里。
3. stage 返回 delta 后由编排器合并、剪枝和重算 ready frontier；不得用局部 stage 结果覆盖全局 state。
4. 只有编排器可以设置最终 `readiness`。
5. 每次等待用户前，至少在会话中保留更新后的 state；不要等到访谈结束才第一次形成 handoff。
6. `current_node`、`awaiting` 和 `return_point` 是恢复字段；任何 detour 或等待都必须写入 checkpoint，返回原节点并成功合并后才清空。

### 状态投影

- 问答、研究、评审和无需设计的 Transient 只保留 `goal`、`intent`、`phase`、`readiness`、`next`；不要填写空的 owner/UI/decision/prototype 字段。
- idea、fog、design、prototype、跨会话工作和任何 detour 使用完整 WorkflowState。
- Minimal 工作升级到设计或持久化时，必须在进入 stage 前补齐完整字段；不要提前让简单任务承担完整 state 成本。

### 跨会话设计检查点

第一次写磁盘前建立 task-root ownership：不存在或为空的根可采用，并在写任务工件前创建 `<task_root>/.dev-workflow-root`，内容至少为 `version: dev-workflow/v0.4`；非空根只有该 marker 存在且版本可读时才自动视为本工作流所有。无 marker 的非空 `.tasks/` / `.dev-workflow/tasks/` / `.codex-tasks/` 一律视为 foreign/legacy，不静默复用、迁移或双写。

默认 `.tasks/` 不可采用时，先读项目根 `.dev-workflow/config.yaml`；其 schema 是 `version: dev-workflow/v0.4` + 项目内规范化相对路径 `task_root`（禁止绝对路径和 `..`）。没有配置时采用确定性后备 `.dev-workflow/tasks/` 并写 marker；后备也不可采用时只问一次。非默认自定义根必须写入该 config，不能只保存在 checkpoint。config 和 root marker 只证明 ownership/discovery，不是设计或任务状态真值。

短设计只用会话 state，不创建文件。满足任一条件时，把完整 state 写入 `<task_root>/<task>/DESIGN.md`：

- 目标属于 `fog`，预计无法在一个会话内收敛。
- 用户暂停、需要 handoff，或上下文即将切换。
- 已有多层决策依赖，重新推导会产生明显损失。

DESIGN 尚未物化时是设计真值。planning 采用单次 cutover：候选工件核对期间 DESIGN 仍胜出；删除 DESIGN 成功后 SPEC/EPIC 才接管。SPEC/EPIC 已存在后不得重建 DESIGN，design-drift 改写 PROGRESS。

### 冷启动恢复

每次新的 `/dev-workflow` invocation 在重新分类意图前执行恢复扫描：

1. 先从用户给出的 checkpoint、项目根 `.dev-workflow/config.yaml`、带 `.dev-workflow-root` marker 的默认 `.tasks/` 和确定性后备 `.dev-workflow/tasks/` 得到候选根；无 marker 的非空根只作为 foreign/legacy 导入候选，未经明确采用/迁移不得读取为活动真值。
2. 同一 task 按真值优先序恢复：存在 `DESIGN.md` 时无论候选 SPEC/EPIC 是否也存在，都 hydrate DESIGN 并重建候选；没有 DESIGN 但 PROGRESS 有 active `Design Drift` 时读取 SPEC/EPIC + drift state；两者都没有但 TODO/SUBTASKS 含 `IN_PROGRESS`、`FAILED`、未完成项，或 `FinalizationStatus` 为 `active` / `pending-validation` / `pending-cleanup` 时，读取 SPEC/EPIC + CSV + PROGRESS 并恢复 execution。`complete` 且已归档/删除的 tracker 不属于候选。
3. 按目标和更新时间能得到唯一明确 task 时直接恢复；多个 task 都合理时列出简短候选并只问一次，不得猜。优先序只解决同一 task 的真值，不能替用户在多个 task 间选择。
4. 校验 version、task_root 和必要字段；Design/Drift 根据 decisions、depends_on、fog、prototype、consensus、current/awaiting/return point 重算 frontier，Execution 根据 CSV 真值重算 dependency-ready task 和父子状态。
5. 从 pending prototype、待回答 node、待确认 consensus、design drift 或未完成 execution task 继续；不得重新建图、重问已定决定或为恢复另建 tracker。

没有可恢复候选才进入普通意图分类。

## 第 0 级 - 意图路由

能从请求和仓库事实判断时直接路由，不为路由本身提问。

| 情况 | 动作 | 初始阶段 |
|---|---|---|
| 继续/恢复、提供 checkpoint，或存在活动 tracker | 按冷启动优先序 hydrate 设计/drift/execution 真值 | checkpoint 对应 phase |
| 问答、研究、解释、评审且不改文件 | 直接处理，不建 tracker；完成后 `stop` | route |
| Bug、测试失败、异常行为 | 读取 `systematic-debugging`；保存 return point | debug |
| 已有可执行计划或明确任务清单 | 复用现有真值，不新建第二套 | execute |
| 部分计划、笔记或不完整 TODO | 先做可执行性检查 | route |
| 目标明确、局部、无重大设计 | 建 Transient 会话 plan | execute |
| 新想法、新功能、行为变化 | 建 WorkflowState，进入决策分流 | design |
| 大而模糊、无法在一个会话看清路径 | 进入发现循环 | discovery |

已有计划同时满足以下条件才直接执行：

1. 目标、边界和完成标准明确。
2. `open_nodes` 和 `fog` 为空，不存在未决产品、架构、安全、数据或 UI 决策。
3. 依赖、第一步和验证方式足以执行。
4. 本 effort 有任何 `owner:user` decision 时，`consensus: confirmed`；缺失或过期则先回 recap，不直接执行。

缺设计回 `design`；只缺执行字段才读 `writing-plans`。不要把半成品计划伪装成可执行计划。

### 任意阶段的 Debug / Verification detour

进入 detour 前写 `return_point.phase`、`return_point.node`、`return_point.evidence_ref`，并设置 `awaiting`：

同一时刻只允许一个 active detour。`return_point` 非空时禁止覆盖；当前 debugging/verification/design-drift 必须先把 delta 合并回保存的 phase/node，再允许发起下一 detour。detour 内发现的新失败由当前 stage 处理或作为 gap 返回，不能另开一层 return point。

- debug `resolved` -> 合并证据后恢复 return point；design/prototype 不强制进入执行，execute 回原 task。
- debug `blocked` / `architecture-review-required` -> 保留 return point，不得伪装继续。
- verification `verified` -> 允许对应声称，清空 return point。
- verification `not-verified` -> 合并 Gaps，恢复 return point 对应的 decision/prototype/task/validation，不得声称完成。
- 顶层 bug 请求在修复后继续其 Transient/Durable/Epic 执行路径。

## 第 1 级 - 按当前决策分权

路由单位是 decision node，不是用户身份。

| 模式 | 触发信号 | 节点 owner | Stage |
|---|---|---|---|
| Assisted | “我不懂 / 你先设计 / 你推荐” | 默认 `model`；重大卡点转 `user` | `design-assist` |
| Owner | “这部分我来定 / 逐项问我 / grill me” | 选定领域内所有真实决策为 `user` | `grilling` |
| Mixed | 不同领域采用不同控制权 | 写入 `owners` map | 按全局 frontier 逐节点选择 |

只声明部分领域时，未声明领域默认 `owner: model`；只有“所有/全部设计都由我定”才把全部领域设为 `user`。若用户措辞对某个已 ready node 的 owner 真有冲突，只围绕该 node 问一次，不做全局模式访谈。

提示已经明确时不再问模式问题。只有交互方式确实不清且会改变结果时，问一次：

> 这部分你想自己逐项定，还是让我先调研并给出推荐设计？

### Assisted 的用户卡点

以下节点必须转为 `owner: user`：

- 多个合理方案存在不可消除的长期重大权衡。
- 影响存量数据、公开接口、兼容性或不可逆迁移。
- 涉及安全、合规、隐私、资金或数据完整性边界。
- 涉及预算、团队能力、期限等工具无法得知的资源约束。
- 用户明确保留所有权。

项目事实、既定技术栈、命名、目录和明显优解由模型处理。

### Owner 的严格边界

Owner/grill-me 领域内只能跳过：

- 可由代码、配置、环境、文档或外部资料查明的事实。
- 已在 `decisions` 中确认的结论。

其余 decision node 无论大小都要一次一问并附推荐。若用户说“这部分你看着办”，先把对应节点显式改为 `owner: model` 并记录 ownership change；不能静默替用户决定。

## 第 1.5 级 - UI 独立分流

业务 owner 不能推导 UI mode。只在真实涉及视觉、布局或交互时设置：

| UI mode | 行为 |
|---|---|
| Prototype-Assisted | UI 节点需要视觉证据时进入 prototype；owner 不因此改变 |
| Design-Owner | UI node 以讨论优先；既有 owner 决定由 grilling 还是 design-assist 处理 |
| Direct | 遵循既有设计稿/设计系统，UI 节点由模型直接解决 |
| Text-Only | 只用文字、表格或简单线框，不进入 prototype |

按用户的明确信号确定 UI mode；同一请求命中冲突信号且差异会改变工作量时才问一次：

1. 已有明确终稿/设计系统且要求照稿实现 -> `Direct`。
2. 明确“不要原型” -> `Text-Only`。
3. 明确“先给我看原型/多版视觉方案” -> `Prototype-Assisted`。
4. 用户保留 UI/design owner 并要求逐项讨论，但没有要求原型 -> `Design-Owner`。
5. Assisted 用户无法靠文字判断真实视觉问题且未表达偏好 -> `Prototype-Assisted`。

这些规则只定 UI mode，不改变 node owner；例如 `Text-Only + owner:user` 仍由 grilling 用文字逐项询问。

应用上述优先级后 UI mode 仍无法确定，且不同 mode 会实质改变结果或成本时才问一次。视觉-only 请求在结论吸收且全局节点关闭后可以 `stop`，不强迫实现。

## 全局 Mixed 调度

1. 编排器从整个 `open_nodes` 中选择一个 dependency-ready 节点。
2. `owner: model` 读取 `design-assist`；`owner: user` 读取 `grilling`。
3. 任一 user-owned 分支活跃时，全工作流每轮最多向用户提出一个决策问题；不能同时追加其他领域的问题批次。
4. stage 只返回当前节点 delta。编排器合并后再选择下一个节点，不允许 design-assist 与 grilling 互相直接 handoff。
5. 局部领域完成不产生全局 plan/execute readiness。必须等全部领域的 `open_nodes` 和 `fog` 关闭。

## Fog 发现循环

`fog` 用于路径还看不清、无法直接形成完整决策树的工作：

1. 先按当前 owner 规则确定 destination：最终要得到 spec、关键决定还是实际改动。
2. breadth-first 扫描产品、架构、数据、安全、UI、资源和验证面；能准确表述的写成稳定 id 的 `open_nodes`，暂时说不清的写入 `fog`。
3. 预计跨会话时，在第一次等待用户前创建 `DESIGN.md`。
4. 按全局 frontier 解决节点：事实研究走 model，价值决策走 user，纸上无法判断的单个节点走 prototype。
5. 每个结果都更新 decisions、pruned/rejected branches、open_nodes 和 fog；新问题只有在可准确表述时才升级为 node。
6. 同时满足以下条件才退出 discovery：destination/成功标准明确；scope/non-goals/约束明确；`fog` 为空；所有设计节点关闭；验证目标已知。
7. 若 breadth-first 扫描证明任务实际很小，直接转普通 design/Transient，不创建长期地图。

发现循环只产生决定，不提前生产实现。

## Prototype detour

进入 prototype 时读取 `../stages/prototype/STAGE.md`，并传入完整 WorkflowState、`ParentDecisionId`、Question 和 Success signal。

prototype 返回：

```text
ParentDecisionId: Dn
Artifact: 路径或 URL
PrototypeStatus: pending | answered
Result: 胜出结论及理由
Rejected: 否决方向
StageStatus: awaiting-user | node-resolved
```

编排器把结果合并回 parent node，保留其他领域 state。`pending` 继续等待同一视觉决定；`answered` 只表示证据已形成：parent 为 `owner: model` 时可据此关闭，为 `owner: user` 时必须保留为 ready，直到用户明确确认。只有全局请求本来就是 visual-only 且没有其他 open node 时，编排器才设置 `Readiness: stop`。

## 共识与 Readiness 硬门

只要存在 user-owned decision，本轮设计必须在所有节点关闭后复述完整决策链，并把 `consensus` 设为 `pending`。只有用户明确确认后才能改为 `confirmed`。

`consensus: confirmed` 后只要新增/重开 `open_node` 或 `fog`，立即失效：本 effort 仍有任何 `owner:user` decision 时重置为 `pending`，否则重置为 `not-required`。重新通过 recap 前不得 planning/execute。

编排器按以下顺序计算 readiness：

1. `open_nodes` 非空、`fog` 非空，或本轮存在 user-owned decisions 且 `consensus != confirmed` -> `continue-design`。
2. 目标只要求研究/评审/设计，且交付已满足 -> `stop`。
3. 单会话低风险实现 -> `transient-execution`。
4. 需要恢复、高风险或依赖较多 -> `durable-plan`。
5. 多 deliverable 或依赖链 -> `epic-plan`。

任何内部 stage 都不能绕过此门。Assisted 产生了未回答用户卡点时同样保持 `continue-design`。

## Stage 导航表

按需只读一个 stage：

| 条件 | Supporting stage |
|---|---|
| model-owned 设计节点 | `../stages/design-assist/STAGE.md` |
| user-owned 设计节点 | `../stages/grilling/STAGE.md` |
| 可运行证据能消除一个设计问题 | `../stages/prototype/STAGE.md` |
| Durable/Epic 计划物化 | `../stages/writing-plans/STAGE.md` |
| Transient/Durable/Epic 执行 | `../stages/executing-plans/STAGE.md` |
| 永久生产代码功能/修复 | `../stages/test-driven-development/STAGE.md`，在实现代码前读取 |
| bug、测试/构建失败、异常 | `../stages/systematic-debugging/STAGE.md` |
| 任何完成或修复声称 | `../stages/verification-before-completion/STAGE.md` |

Transient 直接进入 execution stage；Durable/Epic 先进入 planning stage。执行 stage 对永久生产功能和 bugfix 必须在写实现前加载 TDD stage；throwaway prototype 不加载。

### Design drift 唯一路径

execution 返回 `design-drift` 时，确认 `return_point` 为空后写入 `{ phase: execute, node: <task-id>, evidence_ref: <finding> }`，把受影响决定重新建为 open node，并按共识失效规则重置 consensus 后进入 design/discovery。设计再次通过全局 readiness 后，Transient 重算会话 plan；Durable/Epic 重新进入 planning 更新 SPEC/EPIC、任务边界、依赖和 validation，之后才恢复原 task。只需更新事实文字而不改变决定时属于 fact drift，不走此分支。

### Verification dispatch

进入 verification 前，调用方必须把自己的 phase/node 和证据写入 `return_point`。`verified` 允许对应 claim/stop；`not-verified` 按 return point 恢复，不能成为死端。

## 完成与退出

- 每次等待、stage delta 和恢复都更新同一 WorkflowState。
- 用户暂停时先写必要的 DESIGN/PROGRESS checkpoint，再退出。
- 只有工具和仓库都无法提供、且会实质改变结果的信息才升级给用户。
- 完成声称前必须读取 verification stage 并运行能证明目标的最新检查。
- 用户明确暂停、终止或切换其他工作时，先保存当前 checkpoint，再停止本编排 frame。
