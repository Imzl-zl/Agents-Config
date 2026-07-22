# Grilling - 用户主导的单节点访谈

这是 `/dev-workflow` 的 supporting stage，不是独立 skill。它只处理编排器当前选中的 `owner: user` decision node。事实由模型查，决策由用户定。

## 硬纪律

1. **依赖先行**：只处理 `depends_on` 已关闭的节点。
2. **全局一次一问**：一次只问当前一个 decision node，等待回答后再选择下一题；其他领域不能在同一轮追加问题批次。
3. **事实自查**：代码、配置、环境、文档和外部资料能回答的事实不问用户。
4. **每问附推荐**：给推荐和理由，但不替用户选择。
5. **明确答案不仪式追问**：只有含糊、冲突或改变约束时才追问理由。
6. **共享理解硬门**：没有用户明确确认完整决策 recap，不能进入计划或生产实现。

## 输入

从 WorkflowState 读取完整 goal/constraints、已定 decisions、全局 owners/ui_mode、当前 user-owned node、依赖、剩余 frontier 和 `consensus`。

不要重问：

- 可查明的事实。
- 已在 `decisions` 中确认的结论。

除此之外，当前 Owner 领域内的真实 decision node 无论大小都由用户决定。不要把项目惯例或纯机械步骤膨胀成设计问题；但只要确有多个选择且仍是 user-owned，就必须问，不能以“低风险”为由静默默认。

## 单节点流程

1. 查明当前 node 所依赖的事实，确认问题仍然成立。
2. 提一个开放问题，附推荐、理由和必要权衡。
3. 在等待用户前更新当前 state：`StageStatus: awaiting-user`、当前 node id 和问题。
4. 收到答案后记录决定、owner=user 和理由；用答案剪枝、新增或重排后续节点。
5. 答案含糊或与约束冲突时，只围绕同一 node 追问一次；清楚答案直接关闭 node。
6. 用户说“这部分你看着办”时，不回答该问题；返回明确的 `OwnershipChanges`，由编排器改为 model 后重新调度。
7. 返回 delta；不调用 design-assist/prototype，也不设置全局 `Readiness`。

## UI 与 Prototype

| UI mode | 当前 UI node 的行为 |
|---|---|
| `Prototype-Assisted` | 返回 `PrototypeRequest`，用户通过视觉证据做决定 |
| `Design-Owner` | 保持一次一问；纸上不能可靠判断时返回 `PrototypeRequest` |
| `Direct` | 只有已有明确稿件/规范时可把 node 作为事实关闭；否则仍由用户决定 |
| `Text-Only` | 仅用文字、表格或简单线框，不请求 prototype |

prototype 不夺走 owner。它回答后仍由用户确认 parent decision，除非用户明确把该 node 转给 model。

## 跨领域和节奏

- 编排器维护唯一全局 frontier；本 stage 不和其他 stage 并行向用户提问。
- 用户只转交某个领域时，仅修改该领域剩余 node 的 owner。
- 局部领域结束不代表全局设计结束。
- 每次用户回答后都更新 WorkflowState；长访谈或暂停时同步 `DESIGN.md`，不能等整个 grilling 结束才记录。

## 共识确认

当编排器确认 `open_nodes` 和 `fog` 都为空且本轮存在 user-owned decisions：

1. 复述完整 decisions、重要 assumptions、rejected branches 和边界。
2. 设置 `consensus: pending`。
3. 只问一次是否已达到 shared understanding。
4. 用户明确确认后返回 `ConsensusChange: pending -> confirmed`。

Open nodes 为空但 consensus 未确认时，仍是 `Readiness: continue-design`。

## 输出 delta

```text
Question: 当前 node 的单个问题、推荐和权衡（等待时）
Resolved: Dn -> 结论、owner=user、理由（回答后）
NewNodes: id、domain、owner、question、depends_on
Pruned: 被答案取消的 node 及原因
OwnershipChanges: 显式 owner 变化
PrototypeRequest: none | ParentDecisionId + Question + SuccessSignal
ConsensusChange: none | pending | confirmed
StageStatus: awaiting-user | node-resolved | ownership-changed | needs-prototype | consensus-pending | consensus-confirmed
Next: 对编排器的一个建议动作
```

任何局部 delta 都不能直接宣布 `transient-execution`、`durable-plan` 或 `epic-plan`。
