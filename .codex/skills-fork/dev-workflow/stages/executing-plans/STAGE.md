
# Executing Plans — 单一真值执行阶段

这是 `/dev-workflow` 的 supporting stage，不是独立 skill。它接收已准备好的 Transient、Durable 或 Epic 工作，执行后返回 task delta；不创建第二套 tracker，也不调用其他 skill。

所有 `.tasks/...` 路径按 `WorkflowState.task_root` 解析；恢复和 shape 升级必须沿用同一个根。

磁盘 shape 还必须验证 `<task_root>/.dev-workflow-root`；marker 缺失或版本不匹配时返回 blocked，不猜目录归属。

## 加载形态

### Transient

- 读取会话 plan、设计 handoff 和验收标准。
- 不创建 CSV、SPEC、PROGRESS 或项目根 TODO。
- 如果任务增长到需要恢复或风险显著上升，原地升级为 Durable。

### Durable

按顺序读取：

1. `.tasks/<task>/SPEC.md`
2. `.tasks/<task>/TODO.csv`
3. `.tasks/<task>/PROGRESS.md` 的恢复区

`TODO.csv` 是状态真值；会话 plan 是镜像。

### Epic

读取 `EPIC.md`、`SUBTASKS.csv`、父 `PROGRESS.md`，再读取当前 dependency-ready 子任务目录。父 CSV 是子任务状态真值，子目录 CSV 是子任务内部状态真值。
父子状态重算规则：子任务只由子目录的 TODO.csv 决定；父项为 `FAILED` 当任一 child 为 `FAILED`，为 `IN_PROGRESS` 当没有失败但仍有未完成 child，为 `DONE` 仅当所有 child 都 DONE 且父级 final validation 通过。恢复时若父 CSV 与 child CSV 不一致，按 child 真值重算父项，不反向覆盖 child。

## 开工前检查

1. 目标、boundary、依赖和 validation 是否足以开始。
2. 文件、符号和接口是否仍与仓库一致。
3. `sync_targets` 是否覆盖公开接口、调用方、类型、schema、迁移、配置、测试和文档。
4. WorkflowState 的 `open_nodes` 与 `fog` 是否为空；若有 user-owned decisions，`consensus` 是否已 confirmed。

事实过时但不改变设计时，就地更新 tracker 并记录原因。发现设计缺口时返回 `StageStatus: design-drift` 和原 task return point，由当前编排 frame 恢复 design；不让用户手动重新路由。

## 状态机

```text
TODO → IN_PROGRESS → DONE
                  ↘ FAILED → IN_PROGRESS
```

- 任务开始时同步磁盘 CSV 与会话 plan。
- `DONE` 必须有当前 `validation` 的新鲜证据。
- 失败时保留 `IN_PROGRESS` 或标记 `FAILED`，在 notes 记录错误、已验证假设和下一策略。
- 串行 Durable 同一时刻最多一个 `IN_PROGRESS`。
- Epic 只有在写范围互不冲突、依赖已完成时才能并行多个子任务。

## 每个任务的执行循环

1. 选择第一个依赖已满足的 `TODO` 或可重试 `FAILED`。
2. 标记 `IN_PROGRESS`，同步磁盘真值和会话 plan。
3. 永久生产功能、bugfix、refactor 或行为变化在写实现代码前读取 `../test-driven-development/STAGE.md`，按 Red-Green-Refactor 执行；throwaway prototype 不加载。
4. 按 boundary 实现，不做与目标无关的清理。
5. 对每个 `sync_target` 使用符号 references、schema/route/config 搜索重新发现真实影响面；更新所有必要调用方。
6. 运行该行 `validation`，阅读完整结果。
7. 通过：写 `DONE`、`completed_at` 和关键 notes；更新 `PROGRESS.md` 恢复区。
   所有任务尚未结束时保持 `FinalizationStatus: active`。
8. 未通过：返回 `StageStatus: debug-required`、`ReturnPhase: execute`、当前 task id 和证据；编排 frame 读取 debugging stage，修复后恢复同一 task。
9. 继续下一 dependency-ready 项。

测试失败、依赖缺失或工具报错不是自动询问用户的理由。先调查并改变策略；只有工具和仓库都无法提供、且会实质改变设计的信息才升级给用户。

## Plan 漂移

- **事实漂移**：路径、调用方或命令变化，但目标/设计不变 → 更新 tracker，继续。
- **任务分解漂移**：一个任务需要拆分或新增依赖 → 保留原 id，在 notes 记录拆分，追加新行，不重排旧 id。
- **设计漂移**：接口、数据、安全或产品规则需要重新决定 -> 返回 `StageStatus: design-drift`，并把 ReturnPhase、task id、finding evidence、受影响 open nodes/owners/consensus 写入当前 `PROGRESS.md` 的 `Design Drift` 区。SPEC/EPIC 继续作为稳定设计真值，禁止重建 DESIGN。设计重新收敛后 Durable/Epic 重新读 writing-plans 原地更新 SPEC/EPIC、boundary、depends_on 和 validation，清除 drift 区，再恢复同一 task。只改事实文字、不改变设计的情况属于 fact drift。

不要继续执行已经失真的计划。

## Shape 升级

### Transient → Durable

当出现跨会话风险、任务范围增长、重复失败或需要持久化决策时：

1. 由当前 handoff 生成 `SPEC.md`。
2. 将剩余会话 plan 转成统一 `TODO.csv`。
3. 创建最小 `PROGRESS.md` 恢复区并写 `FinalizationStatus: active`。
4. 磁盘 CSV 成为真值，重新同步会话 plan。

### Durable → Epic

当一个 TODO 开始承载多个独立 deliverable 或依赖链时：

1. 创建父 `EPIC.md`、`SUBTASKS.csv`、`PROGRESS.md`，父 PROGRESS 写 `FinalizationStatus: active`。
2. 将现有 Durable 目录完整移动到 `.tasks/<epic>/tasks/<child>/`，保留 TODO 行状态、`plan_ref`、SPEC、PROGRESS 和 raw 资料。
3. 在父 `SUBTASKS.csv` 写入该 child 的准确相对 `task_dir`，并把原目标、依赖和当前状态复制为父协调行。
4. 重写 child `PROGRESS.md` 的 Truth、相关 raw/SPEC 相对路径和恢复提示，使冷启动路径全部指向新位置。
5. 将剩余 deliverable 建成独立 child，并声明 `depends_on`；新 child 从 `TODO` 开始，其 PROGRESS 写 `FinalizationStatus: active`。
6. 按父子状态重算规则初始化父 `SUBTASKS.csv` 与父 `PROGRESS.md`。
7. 父 `SUBTASKS.csv` 成为协调真值，child TODO.csv 仍是各 child 内部真值。

## 恢复输出

跨会话恢复时先报告：

```text
Task: 一句话目标
Shape: transient | durable | epic
FinalizationStatus: none (Transient) | active | pending-validation | pending-cleanup | complete (normally not resumed)
Progress: X/Y
Current: 当前行或子任务
Truth: Transient 写 session plan；Durable 写 TODO.csv；Epic 同时写父 SUBTASKS.csv 和当前 child TODO.csv
Parent status: Epic 时报告按 child 真值重算后的父状态及任何分歧
Latest validation: 命令和结果
Next: 一个明确动作
```

## 整体完成

所有任务完成后：

1. Durable/Epic 把 `FinalizationStatus` 设为 `pending-validation`；Transient 不创建该字段。随后根据会话验收或 SPEC/EPIC final validation 运行整体 smoke、相关测试、lint 或 build，只运行能证明本次声称的完整检查。
2. 返回 `StageStatus: verification-required` 和完整证据；编排 frame 读取 `../verification-before-completion/STAGE.md` 核对需求与证据。
3. `not-verified` 记录 gaps 并回原 task/debug，Durable/Epic 保持 `pending-validation`；`verified` 恢复 execution 后，Durable/Epic 改为 `pending-cleanup`，Transient 继续完成会话交付。
4. 未被用户要求时不自行 commit、push 或创建 PR；按当前仓库规则交付改动和验证结果。
5. 项目已有 memory/archive 约定时，将关键决策、根因和坑点写入该位置；没有时不新建一套记忆目录。`SPEC.md` / `EPIC.md` 是否保留遵循项目文档约定；清理临时 raw 和无价值恢复信息。
6. Transient 确认会话 plan 全部 completed 后结束。Durable/Epic 还要确认 CSV 没有未完成/失败行且 cleanup 已结束，设置 `FinalizationStatus: complete`；只有此后才可删除临时 `TODO.csv` / `SUBTASKS.csv`。`complete` 的 PROGRESS 可按项目归档约定保留或删除。
