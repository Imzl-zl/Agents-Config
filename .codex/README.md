# `.codex` 目录边界

本目录只放 Codex 的 agent 配置与参考快照。zhanggui skill 项目已独立至仓库根 [`zhanggui/`](../zhanggui/README.md)，不在本目录内。

```text
.codex/
├── AGENTS.md                     # Codex 全局规则
├── config(Win).toml / config(WSL).toml
├── prompts/init-project.md
├── skills/                       # 真正安装/可发现的独立技能
│   ├── .system/
│   └── init-project/
├── reference-skills/             # 只读快照，不参与 discovery
│   ├── taskmaster/
│   ├── todo-list-csv/
│   └── README.md
└── README.md                     # 本文件
```

仓库根目录相关内容：

- `zhanggui/`：单入口工作流 skill 项目（plugin manifest + 自包含可移植 skill + 设计文档）。
- `superpowers/`：原版 Superpowers 克隆（gitignored）；Codex plugin marker 已改为 `plugin.json.disabled`。
- `skills/`：其他上游 skill 设计参考（gitignored）。

## 运行规则

- 日常融合流程只调用 `/zhanggui`；入口与 stages 位于 `zhanggui/skills/zhanggui/`。
- explicit-only 防止普通问答自动进入重工作流；阶段切换仍由同一 frame 完成。
- 短任务使用 minimal 会话状态；跨会话 pre-plan 设计使用 DESIGN，物化后由 SPEC/EPIC 接管，design-drift 写 PROGRESS。
- `reference-skills/`、根目录 `superpowers/` 和根目录 `skills/` 不是运行依赖，只用于阅读、比较和追溯。
- 融合改动写入仓库根 `zhanggui/`，不得反向修改参考快照。
- `.codex/skills/` 只放确实需要独立发现的已安装技能。

## 文档职责

- [`../zhanggui/README.md`](../zhanggui/README.md)：如何使用当前实现。
- [`../zhanggui/docs/skill-fusion-design.md`](../zhanggui/docs/skill-fusion-design.md)：为什么这样设计、哪些方案被否决。
- [`reference-skills/README.md`](reference-skills/README.md)：参考快照来源和只读约束。
