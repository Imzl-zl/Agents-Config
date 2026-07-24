# Code Review — 双向代码审查

这是 `/zhanggui` 的 supporting stage，不是独立 skill。它有两个方向：**请求审查**（把改动交给独立审查视角检查）和**接收审查**（处理外部反馈）。本 stage 不改变 WorkflowState 的 owner、design decisions 或全局 readiness。

## 触发条件

**必跑**（由执行 stage 的完成门在 verification 之前调用）：

- 涉及安全、迁移、公开接口、存量数据，或跨多文件的改动。
- Epic child 完成、合并或交付之前。
- 用户明确要求审查。

**可跳过**（在 notes 记录跳过理由）：

- 单文件小改、纯文档改动、throwaway prototype。

**Ad-hoc**：意图路由的"评审且不改文件"请求直接使用本检查单，报告后 `stop`，不建 tracker。

## 输入

```text
Scope: BASE_SHA..HEAD_SHA 或明确 diff 范围
Contract: SPEC/decisions/task goal+boundary（忠实轴的对照物）
Trigger: completion-gate | per-task | ad-hoc | pre-merge
ReturnPhase / ReturnNode: 从执行调用时必填
```

本 stage 由执行循环或意图路由**同步调用**，不占用 detour 单槽：不写 `WorkflowState.return_point`，`awaiting` 保持不变；`ReturnPhase` / `ReturnNode` 只是原样回传的定位字段。

## 请求审查

1. 确定 git 范围：`BASE_SHA` / `HEAD_SHA`（或当前工作区 diff）。
2. 宿主有 subagent 能力时，按 [code-reviewer.md](code-reviewer.md) 模板派一个通用子代理。占位符映射：`DESCRIPTION` = 本次改动摘要；`PLAN_OR_REQUIREMENTS` = SPEC/decisions/task 的 goal+boundary；`BASE/HEAD_SHA` = 范围。审查者只拿到模板内容，不继承会话历史——这让它审查工作产物而不是你的思路。
3. 无 subagent 能力时自审：先完整读 diff，再按模板 "What to Check" 两轴逐项过：
   - **忠实轴**：实现是否符合计划/决策；偏离是正当改进还是问题；计划功能是否齐全。
   - **质量轴**：正确性、错误处理、边界情况、安全、测试验证真实行为而非 mock、生产就绪（迁移/兼容/文档）。
4. 审查是只读的：不改工作区、index、HEAD 或分支状态。

## 处理审查结果

- **Critical**（bug、安全、数据风险）：立即修复——返回执行循环新增或重开任务行，修复后重新运行该行 validation。
- **Important**（架构问题、缺功能、错误处理差、测试缺口）：进入 verification 前修复。
- **Minor**（风格、优化、文档润色）：记入 notes，不阻塞。
- 审查者认为**计划本身**有问题：按 design-drift 规则返回，不在审查里改设计。
- 审查者**错了**：用技术理由反驳（引用代码、测试、构建目标），不盲从，也不争论有效的技术反馈。

## 接收外部审查反馈

```text
1. READ：完整读完再反应
2. UNDERSTAND：用自己的话复述需求（或提问）
3. VERIFY：对照代码库现实核实
4. EVALUATE：对本代码库是否技术正确
5. RESPOND：技术性回应或有据反驳
6. IMPLEMENT：一次一项，逐项测试
```

规则：

- 任何一项不清楚：**全部暂停**，先澄清。条目之间可能相关，部分理解 = 错误实现。
- 实施顺序：阻塞项（崩溃/安全）→ 简单修复 → 复杂修复；逐项验证，确认无回归。
- 反驳时机：建议破坏现有功能、审查者缺完整上下文、违反 YAGNI（grep 证明无调用方就提议删除而不是"实现完整"）、对本技术栈不正确、与既有 `owner:user` 决策冲突（冲突时先回 recap，不私改决策）。
- **禁止表演性同意**："You're absolutely right!"、"Great point!"、任何感谢套话。正确的反馈直接修，用代码说话："已修，改动在 X。"
- 反驳错了就事实性更正："核实过你是对的——X 确实如此，正在修。"不长篇道歉，不辩解。

## 输出 delta

```text
Strengths: 做得好的具体点（帮助校准其余反馈的可信度）
Findings: [{severity: critical|important|minor, file:line, what, why, fix}]
ReturnPhase / ReturnNode: 原样返回
StageStatus: review-passed | fixes-required | blocked
```

`review-passed` 不等于 verification：完成声称仍必须走 verification stage 的新鲜证据。本 stage 不设置全局 readiness。
