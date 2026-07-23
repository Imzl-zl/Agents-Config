# Zhanggui（掌柜）v0.4

面向强模型、对非专业用户友好的单入口开发工作流。运行实现位于本目录；taskmaster、todo-list-csv 和其他上游材料只是隔离参考源，不是运行依赖。

> 本 README 只是导览，不参与运行，也不复述规则细节——避免出现第三份会漂移的"真值"。运行契约以 [`zhanggui/zhanggui/SKILL.md`](zhanggui/zhanggui/SKILL.md) 与各 `STAGE.md` 为准；完整设计论证见 [Skill 融合工作流设计](../docs/skill-fusion-design.md)。

- 设计理念：[Skill 融合工作流设计](../docs/skill-fusion-design.md)
- 目录边界：[`.codex/README.md`](../README.md)

## 快速使用

只调用一次：

```text
/zhanggui
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
└── zhanggui/                 # plugin skills root
    ├── zhanggui/             # 唯一可发现入口
    │   ├── SKILL.md
    │   ├── RECOVERY.md           # task-root 与冷启动细则，按需读取
    │   └── agents/openai.yaml
    ├── stages/                   # supporting docs，不参与 discovery
    │   ├── design-assist/STAGE.md
    │   ├── grilling/STAGE.md
    │   ├── prototype/{STAGE,UI,LOGIC}.md
    │   ├── writing-plans/STAGE.md
    │   ├── executing-plans/STAGE.md
    │   ├── test-driven-development/STAGE.md
    │   ├── systematic-debugging/STAGE.md
    │   ├── verification-before-completion/STAGE.md
    │   ├── code-review/{STAGE,code-reviewer}.md
    │   ├── finishing-a-development-branch/STAGE.md
    │   ├── using-git-worktrees/STAGE.md
    │   └── dispatching-parallel-agents/STAGE.md
    └── <9 个 legacy reference>/ # legacy 参考（含适配注记），不可发现
        └── REFERENCE.md
```

### 宿主调用模型

- `zhanggui/SKILL.md` 是 plugin 唯一的 `SKILL.md`，Claude 设置 `disable-model-invocation: true`，Codex 设置 `allow_implicit_invocation: false`。
- 用户只在开始时调用一次 `/zhanggui`；explicit-only 是为了避免普通问答被重工作流自动接管。
- 不增加浅层 auto-router：model-invoked router 无法调用 user-only 主入口，只会制造第二套编排。
- 入口保持为整个会话的 orchestration frame，按需读取一个 `STAGE.md`；stage 返回 delta，不 invoke 自己或 sibling。
- `stages/`、`RECOVERY.md` 和 `REFERENCE.md` 都不参与 skill discovery。

## 核心概念地图

规则原文只在右列出处维护，此表只是索引：

| 概念 | 一句话 | 规则出处 |
|---|---|---|
| WorkflowState | 全会话唯一状态对象；简单任务只用最小投影 | SKILL.md「WorkflowState」 |
| 意图路由 | 恢复/问答/bug/计划/局部改动/想法/fog 的入口分流 | SKILL.md「第 0 级」 |
| Decision owner | 以决策节点为单位分权：Assisted / Owner / Mixed | SKILL.md「第 1 级」 |
| UI mode | Prototype-Assisted / Design-Owner / Direct / Text-Only，与业务 owner 正交 | SKILL.md「第 1.5 级」 |
| Prototype | 用可运行证据回答一个决策节点，delta 合并回 parent | `stages/prototype/` |
| Fog discovery | 大而模糊工作的发现循环，只产生决定不产生实现 | SKILL.md「Fog 发现循环」 |
| 共识与 readiness | user-owned 决策必须 recap 确认；五级 readiness 硬门 | SKILL.md「共识与 Readiness 硬门」 |
| Shape 与真值 | Transient 会话 plan / Durable TODO.csv / Epic 父子 CSV，单一真值 | `stages/writing-plans/`、`stages/executing-plans/` |
| 恢复 | 压缩 state 块 + 机械触发的 DESIGN checkpoint + 冷启动流程 | SKILL.md「跨会话设计检查点」、`RECOVERY.md` |
| 质量与收尾 | code review 双轴（质量+忠实）、分支收尾选项、worktree 隔离、并行派发——按需加载，不用不占上下文 | `stages/code-review/` 等 4 个 stage |

与模型内置 plan 的关系：模型 plan 保持短（目标与即时进度），磁盘 tracker 只补执行细节（boundary、related_files、sync_targets、depends_on、validation），会话 plan 是磁盘 CSV 的镜像投影。细节见 `stages/writing-plans/STAGE.md`「与模型 plan 的互补」。

## Runtime 与参考资料

### 唯一可发现入口

- `zhanggui`

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

这些目录保留 `REFERENCE.md` 供设计比较；其中 brainstorming、using-superpowers、subagent-driven-development、dispatching-parallel-agents 含少量 zhanggui 适配注记，其余接近上游原文。每份文件在正文前有 inert marker，因此历史 frontmatter 不是文件首部，也不会被 skill loader 解析。对应 `agents/openai.yaml` 已移除，所以它们不是 slash command。

其中 4 份能力已提升为按需加载的运行 stage：requesting/receiving-code-review 融合为 `stages/code-review/`；finishing-a-development-branch、using-git-worktrees、dispatching-parallel-agents 各有对应 stage。REFERENCE 原文继续留存对比。subagent-driven-development 保持 reference——整体移植会创建第二套执行真值（违反设计文档演进规则 3），其"每任务审查 + 派发"理念已并入 code-review 与 dispatching 两个 stage。brainstorming、using-superpowers 被入口的设计路由取代。`brainstorming/scripts/` 的 server 方案只属于 legacy 参考——运行时 UI 原型使用 `stages/prototype/UI.md` 的无 server `?variant=` 方案。根目录 `superpowers/`、`skills/` 以及 `.codex/reference-skills/` 同样只读，不参与运行。

### 任务目录与迁移

v0.4 默认根仍为 `.tasks/`。不存在/空目录可采用并创建 `.zhanggui-root`；非空目录只有 marker 版本匹配才自动视为本工作流所有。默认根不可用时，项目根 `.zhanggui/config.yaml` 可用 `version` + 项目内相对 `task_root` 声明自定义根；否则使用确定性后备 `.zhanggui/tasks/`。完整细则见入口旁 [`RECOVERY.md`](zhanggui/zhanggui/RECOVERY.md)。本设计仓库没有真实 `.codex-tasks/` 任务，因此不做虚构迁移；已有项目只能显式采用、一次性导入或保留隔离，禁止静默合并和双写。

### 文档维护

生产代码的文件行数可作为拆分信号；Skill、STAGE、设计文档和参考资料不设 300 行硬上限。必要时按职责做 progressive disclosure，但不得为满足数字删除约束、示例或可读空白。

恢复上游 Superpowers plugin 前必须先停用 zhanggui；不要同时启用两个强入口。

## 验收场景

23 条权威验收场景见 [设计文档 §12](../docs/skill-fusion-design.md)。建议的冒烟子集（先跑这 5 条）：Assisted 冷启动、Mixed 一次一问、prototype detour 合并、design-drift、冷启动恢复。
