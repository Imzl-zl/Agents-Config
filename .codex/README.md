# `.codex` 目录边界

本目录把运行实现、安装技能、参考快照和设计文档物理分开，避免参考源被误认为运行依赖。

```text
.codex/
├── skills-fork/                  # Zhanggui v0.4 plugin
│   ├── .codex-plugin/
│   ├── zhanggui/             # plugin skills root
│   │   ├── zhanggui/         # 唯一可发现 SKILL + Codex policy
│   │   ├── stages/               # supporting STAGE.md，不可发现
│   │   └── <9 legacy references> # REFERENCE.md，不可发现
│   └── README.md
├── skills/                       # 真正安装/可发现的独立技能
│   ├── .system/
│   └── init-project/
├── reference-skills/             # 只读快照，不参与 discovery
│   ├── taskmaster/
│   ├── todo-list-csv/
│   └── README.md
├── docs/skill-fusion-design.md   # 权威设计
└── README.md                     # 本文件
```

仓库根目录另有两个上游参考源：

- `superpowers/`：原版 Superpowers；Codex plugin marker 已改为 `plugin.json.disabled`。
- `skills/`：其他上游 skill 设计参考。

## 运行规则

- 日常融合流程只调用 `/zhanggui`；这是 plugin 唯一可发现 skill。
- `stages/` 是 supporting docs，九个 legacy 目录是 REFERENCE，均不参与 discovery。
- explicit-only 防止普通问答自动进入重工作流；阶段切换仍由同一 frame 完成。
- 短任务使用 minimal 会话状态；跨会话 pre-plan 设计使用 DESIGN，物化后由 SPEC/EPIC 接管，design-drift 写 PROGRESS。
- `reference-skills/`、根目录 `superpowers/` 和根目录 `skills/` 不是运行依赖，只用于阅读、比较和追溯。
- 融合改动写入 `skills-fork/`，不得反向修改参考快照。
- `.codex/skills/` 只放确实需要独立发现的已安装技能。

## 文档职责

- [`skills-fork/README.md`](skills-fork/README.md)：如何使用当前实现。
- [`docs/skill-fusion-design.md`](docs/skill-fusion-design.md)：为什么这样设计、哪些方案被否决。
- [`reference-skills/README.md`](reference-skills/README.md)：参考快照来源和只读约束。
