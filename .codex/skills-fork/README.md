# Dev Workflow v0.4

面向强模型、对非专业用户友好的单入口开发工作流。运行实现位于本目录；taskmaster、todo-list-csv 和其他上游材料只是隔离参考源，不是运行依赖。

- 设计理念：[Skill 融合工作流设计](../docs/skill-fusion-design.md)
- 目录边界：[`.codex/README.md`](../README.md)

## 快速使用

只调用一次：

```text
/dev-workflow
```

随后直接描述目标。入口在同一编排 frame 内完成：

```text
意图判断
  -> 建立可恢复 WorkflowState
  -> 按 decision node owner 调度设计
  -> 必要时做 UI/逻辑 prototype
  -> Transient / Durable / Epic
  -> TDD、执行、调试
  -> 新鲜验证
```

阶段切换不需要再次输入 slash command。

## 运行结构

```text
.codex/skills-fork/
├── .codex-plugin/plugin.json
├── README.md
└── dev-workflow/                 # plugin skills root
    ├── dev-workflow/             # 唯一可发现入口
    │   ├── SKILL.md
    │   └── agents/openai.yaml
    ├── stages/                   # supporting docs，不参与 discovery
    │   ├── design-assist/STAGE.md
    │   ├── grilling/STAGE.md
    │   ├── prototype/{STAGE,UI,LOGIC}.md
    │   ├── writing-plans/STAGE.md
    │   ├── executing-plans/STAGE.md
    │   ├── test-driven-development/STAGE.md
    │   ├── systematic-debugging/STAGE.md
    │   └── verification-before-completion/STAGE.md
    └── <9 个 legacy reference>/ # 原文保留，不可发现
        └── REFERENCE.md
```

### 宿主调用模型

- `dev-workflow/SKILL.md` 是 plugin 唯一的 `SKILL.md`，Claude 设置 `disable-model-invocation: true`，Codex 设置 `allow_implicit_invocation: false`。
- 用户只在开始时调用一次 `/dev-workflow`；explicit-only 是为了避免普通问答被重工作流自动接管。
- 不增加浅层 auto-router：model-invoked router 无法调用 user-only 主入口，只会制造第二套编排。
- 入口保持为整个会话的 orchestration frame，按需读取一个 `STAGE.md`；stage 返回 delta，不 invoke 自己或 sibling。
- `stages/` 和 `REFERENCE.md` 都不参与 skill discovery。

## WorkflowState

所有路由和恢复共用一个状态对象，核心字段包括：

```text
goal / intent / phase / constraints
decision_mode / owners{domain -> model|user} / ui_mode
decisions / assumptions
open_nodes{id, domain, owner, question, depends_on, status} / fog
prototype{parent_id, status, artifact, result}
current_node / awaiting
return_point{phase, node, evidence_ref}
consensus / readiness
task_root / next / checkpoint
```

规则：

1. `open_nodes` 包含所有未决设计，不只包含重大问题。
2. id 创建后不复用；依赖满足的节点才进入 ready frontier。
3. stage 只返回当前 node 的 delta；只有入口能计算全局 readiness。
4. 每次等待用户、owner 切换、prototype detour 和阶段返回前先更新 state。
5. 局部领域完成不代表全局设计完成。
6. detour 前写 `return_point`；恢复原节点并成功合并后才清空。

### 状态投影

- 问答、研究、评审和无需设计的 Transient 只使用 `goal / intent / phase / readiness / next`。
- Design/Discovery、prototype、跨会话和 detour 才使用完整 WorkflowState。
- Minimal 工作升级时再补全字段，简单任务不填写空的 owner/UI/decision graph。

## 意图路由

| 输入 | 路由 |
|---|---|
| 继续/恢复、给出 checkpoint，或存在活动 tracker | 先按 config/marker 解析根，再按 DESIGN → Design Drift → Durable/Epic tracker 优先序恢复 |
| 问答、研究、解释、评审且不改文件 | 直接处理，`stop`，不建 tracker |
| Bug、测试/构建失败、异常 | debugging stage，保存 `return_point` |
| 完整可执行计划 | 复用现有真值，进入 execution |
| 部分计划 | 检查设计、边界、依赖和验证缺口 |
| 明确局部改动 | Transient execution |
| 新想法或行为变化 | design frontier |
| 大而模糊、无法一会话看清 | discovery loop |

计划只有在目标/边界/完成标准明确、`open_nodes` 和 `fog` 为空、共识门满足、依赖和验证足够时才可执行。

## Decision owner

路由单位是当前 decision node，不是用户画像。

| 模式 | 规则 |
|---|---|
| `Assisted` | 默认 owner=model；模型查事实并解决明显优解，重大卡点转 user |
| `Owner` | 用户选定领域的真实决策全部 owner=user；一次一问并附推荐 |
| `Mixed` | 各领域写入 owner map；入口从同一全局 frontier 逐节点调度 |

用户只声明部分领域时，未声明领域默认 owner=model；只有用户明确说“全部设计都由我定”才把所有领域设为 user。ready node 的 owner 信号冲突时只问该节点一次。

### Owner / grill-me 保真

Owner 领域只能跳过：

- 代码、配置、环境、文档和外部资料能回答的事实。
- 已确认的决定。

其余真实 decision node 无论大小都由用户决定。用户说“这部分你看着办”时，必须先显式把对应 node 改成 owner=model；模型不能静默替用户决定。

任何 user-owned 分支活跃时，全工作流一轮最多问一个问题。所有节点关闭后还必须完成一次完整 recap，并记录 `consensus: confirmed`，才能进入计划或执行。

## UI 独立分流

| UI mode | 行为 |
|---|---|
| `Prototype-Assisted` | 对需要视觉证据的 UI node 做 prototype，owner 不随之改变 |
| `Design-Owner` | UI node 以讨论优先；既有 owner 决定由 grilling 还是 design-assist 处理 |
| `Direct` | 按已有稿件或设计系统直接解决 UI nodes |
| `Text-Only` | 只用文字、表格或线框，不做 prototype |

确定性优先级：已有终稿且照稿实现 -> `Direct`；明确不要原型 -> `Text-Only`；明确先看原型/多版视觉方案 -> `Prototype-Assisted`；用户保留 UI owner 并要求逐项讨论但未要求原型 -> `Design-Owner`；Assisted 用户无法靠文字判断真实视觉问题 -> `Prototype-Assisted`。冲突信号会改变成本时才问一次。UI mode 不改变 node owner。

业务 owner 不能推导 UI mode。视觉-only 请求在 parent node 和全局 frontier 都关闭后可以结束，不强迫实现。

## Prototype detour

prototype 必须带：

```text
ParentDecisionId
Question
SuccessSignal
```

返回：

```text
ParentDecisionId
Artifact
PrototypeStatus: pending | answered
Result / Rejected
StageStatus: awaiting-user | node-resolved | blocked
```

入口把 delta 合并回 parent node；其他领域的 decisions、owners、open_nodes、fog 和 ui_mode 不丢失。prototype 不设置全局 readiness。

## Fog discovery

1. 先确认 destination。
2. breadth-first 扫描产品、架构、数据、安全、UI、资源和验证面。
3. 能准确表述的形成带依赖的 node；暂时说不清的留在 `fog`。
4. 按 ready frontier 逐节点研究、grilling 或 prototype。
5. 每个答案都更新 decisions、pruned branches、nodes 和 fog。
6. destination/成功标准、scope/non-goals/约束明确，`fog` 与 nodes 清空，验证目标已知后才毕业。

发现循环只产生决定，不提前写生产实现。

## 设计恢复

- 短设计使用会话 WorkflowState，不创建文件；跨会话、暂停或复杂依赖才写 `<task_root>/<task>/DESIGN.md`。
- 冷启动先解析用户 checkpoint、项目根 `.dev-workflow/config.yaml`，以及带 `.dev-workflow-root` marker 的 `.tasks/` / 确定性后备 `.dev-workflow/tasks/`；无 marker 的非空根和 `.codex-tasks/` 未经明确采用/迁移不作为活动真值。
- 同一 task 的恢复优先序是 DESIGN（即使候选 SPEC 已存在）→ active Design Drift → CSV 非终态或 `FinalizationStatus: active | pending-validation | pending-cleanup` 的 Durable/Epic tracker。
- DESIGN 是 pre-plan 真值。候选工件核对期间不得 yield/执行；全部最终工件提交并复读后，以删除 DESIGN 作为最后一次真值写操作。
- DESIGN 删除成功后 SPEC/EPIC 接管设计真值、CSV 接管状态真值；之后不再重建 DESIGN。design-drift 的未决 state 写入 PROGRESS，普通执行恢复读取 SPEC/EPIC + CSV + PROGRESS。
- 多个 task 都可能匹配时只问一次选择，不用同一 task 的优先序替用户猜任务。

这保证每一阶段只有一个真值，同时覆盖设计恢复和普通执行恢复。

## Readiness 硬门

`consensus: confirmed` 后只要新增/重开 node 或 fog，立即重置为 `pending`（仍有 user-owned decisions）或 `not-required`（没有）；重新 recap 前不能 planning/execute。

入口按顺序计算：

1. `open_nodes`/`fog` 非空，或本轮存在 user-owned decisions 且 `consensus != confirmed` -> `continue-design`
2. 只要求研究/评审/设计且交付满足 -> `stop`
3. 单会话低风险实现 -> `transient-execution`
4. 需要恢复、高风险或依赖较多 -> `durable-plan`
5. 多 deliverable 或依赖链 -> `epic-plan`

内部 stage 无权绕过此门。

## 计划与执行真值

| Shape | 状态真值 |
|---|---|
| Transient | 会话 plan；不创建执行文件 |
| Durable | `.tasks/<task>/TODO.csv` |
| Epic | 父 `SUBTASKS.csv` + 当前 child `TODO.csv` |

`PROGRESS.md` 固定写 `FinalizationStatus: active | pending-validation | pending-cleanup | complete`。它只表示整体恢复阶段；CSV 仍是任务状态真值。`complete` 且已归档/删除的 tracker 不参与冷启动。

统一 Durable CSV：

```csv
id,plan_ref,goal,boundary,related_files,sync_targets,depends_on,status,validation,completed_at,notes
```

`sync_targets` 支持 `symbol | type | schema | route | config | migration | test | doc`。计划记录已知影响面；执行必须重新做 references/search。

永久生产功能、bugfix、refactor 或行为变化在写实现前读取 TDD stage。throwaway prototype 不读取。debug/verification detour 都先保存 `return_point`，失败时回原 decision/prototype/task/validation。design-drift 必须重开受影响 node、重新共识和 planning，不能只改 SPEC 后继续；完成声称需要 verification 的新鲜证据。

同一时刻只允许一个 active detour；`return_point` 非空时不得覆盖。当前 debug/verification/design-drift 先合并并恢复原节点，下一 detour 才能开始。

`Batch` 不再是独立 shape，也不依赖 `spawn_agents_on_csv`。同质批量工作按风险和恢复价值选择 Durable/Epic，再把独立且写范围不冲突的行作为执行并行策略；数量大不等于必须 Epic。

## Runtime 与参考资料

### 唯一可发现入口

- `dev-workflow`

### 9 份 legacy reference

- `brainstorming`
- `using-superpowers`
- `subagent-driven-development`
- `finishing-a-development-branch`
- `requesting-code-review`
- `receiving-code-review`
- `dispatching-parallel-agents`
- `using-git-worktrees`
- `writing-skills`

这些目录保留完整 `REFERENCE.md` 供设计比较；每份文件在原文前有 inert marker，因此历史 frontmatter 不是文件首部，也不会被 skill loader 解析。对应 `agents/openai.yaml` 已移除，所以它们不是 slash command。根目录 `superpowers/`、`skills/` 以及 `.codex/reference-skills/` 同样只读，不参与运行。

### 任务目录与迁移

v0.4 默认根仍为 `.tasks/`。不存在/空目录可采用并创建 `.dev-workflow-root`；非空目录只有 marker 版本匹配才自动视为本工作流所有。默认根不可用时，项目根 `.dev-workflow/config.yaml` 可用 `version` + 项目内相对 `task_root` 声明自定义根；否则使用确定性后备 `.dev-workflow/tasks/`。本设计仓库没有真实 `.codex-tasks/` 任务，因此不做虚构迁移；已有项目只能显式采用、一次性导入或保留隔离，禁止静默合并和双写。

### 文档维护

生产代码的文件行数可作为拆分信号；Skill、STAGE、设计文档和参考资料不设 300 行硬上限。必要时按职责做 progressive disclosure，但不得为满足数字删除约束、示例或可读空白。

恢复上游 Superpowers plugin 前必须先停用 dev-workflow；不要同时启用两个强入口。

## 验收场景

- “我什么都不懂，你先设计” -> Assisted；模型查事实，只把重大卡点转 user。
- “业务我定，UI 你先做三版” -> business Owner + UI Prototype-Assisted；全局仍一次一问。
- “业务你定，UI 我定” -> per-domain Mixed；局部完成不提前执行。
- “不要原型” -> Text-Only。
- 只比较布局 -> parent node 关闭且无其他节点时 stop。
- 长 grilling / fog effort -> DESIGN checkpoint 可恢复依赖、owner、UI mode 和 consensus。
- 设计中的 bug -> debug 后回原 decision node。
- 部分计划 -> readiness check，不伪装成可执行。
- 短改动 -> Transient，无持久 CSV。
- 高风险/跨会话 -> Durable；多 deliverable -> Epic。
