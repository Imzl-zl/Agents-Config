# Reference Skill Snapshots

本目录保存融合改造前的参考快照。它们用于对照和追溯，不参与运行。

## 内容

| 目录 | 来源用途 | 运行状态 |
|---|---|---|
| `taskmaster/` | Compact/Full/Epic/Batch、恢复和 CSV 真值协议参考 | 不可发现，不调用 |
| `todo-list-csv/` | 项目根 CSV 与会话 plan 同步脚本参考 | 不可发现，不调用 |

其他上游参考源位于仓库根目录：

- `superpowers/`
- `skills/`

## 只读约束

1. 参考快照保持原始内容，不为适配 zhanggui 修改 frontmatter、正文、脚本或模板。
2. 需要复用理念时，先形成明确设计结论，再在仓库根 `zhanggui/` 中重新实现。
3. 不把本目录加入 `.codex-plugin` 的 `skills` 路径。
4. 不把参考目录移动回 `.codex/skills/`；该目录只存真正安装的技能。
5. 参考文件中的 `.codex-tasks`、项目根 CSV 等路径只描述旧协议，不代表当前 zhanggui 的运行行为。

当前融合实现与操作说明见 [`../../zhanggui/README.md`](../../zhanggui/README.md)。
