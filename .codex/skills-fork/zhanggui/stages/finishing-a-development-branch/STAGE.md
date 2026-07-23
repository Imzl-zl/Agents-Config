# Finishing a Development Branch — 分支收尾

这是 `/zhanggui` 的 supporting stage，不是独立 skill。verification 已 `verified` 且工作发生在独立分支/worktree 上，或用户要求收尾时读取。收尾方式是用户决定：本 stage 呈现结构化选项并执行选择，符合"未被要求不自行 commit/push/PR"的全局规则。

## 输入

```text
Verified: verification stage 的新鲜结论（必须为 verified）
Branch / Workspace: 当前分支与工作区状态
```

verification 不是 `verified` 时返回 `StageStatus: blocked`，先回验证/修复，不带着失败的检查进入收尾。

## Step 1: 检测环境

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

| 状态 | 菜单 | 清理 |
|------|------|------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无 worktree 需要清理 |
| `GIT_DIR != GIT_COMMON`，具名分支 | 标准 4 选项 | 按来源判定（Step 4） |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 精简 3 选项（无本地合并） | 不清理（外部管理） |

## Step 2: 确定 base 分支

```bash
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

不确定时问一句："这个分支是从 main 切出来的，对吗？"

## Step 3: 呈现选项（一次一问，不加解释）

普通仓库与具名分支 worktree：

```text
实现完成。接下来怎么处理？

1. 本地合并回 <base-branch>
2. Push 并创建 Pull Request
3. 保留分支（稍后自行处理）
4. 丢弃这些工作

选哪个？
```

Detached HEAD：

```text
实现完成。当前是 detached HEAD（外部管理的工作区）。

1. Push 为新分支并创建 Pull Request
2. 保持现状（稍后自行处理）
3. 丢弃这些工作

选哪个？
```

## Step 4: 执行选择

**选项 1（本地合并）**：先 `cd` 到主仓库根，`git checkout <base>` → `git pull` → `git merge <feature>` → 在合并结果上重跑测试确认；确认成功后才清理 worktree、`git branch -d <feature>`。

**选项 2（Push + PR）**：`git push -u origin <feature>`。**不清理 worktree**——用户需要它迭代 PR 反馈。

**选项 3（保留）**：报告"保留分支 <name>，worktree 在 <path>"。不清理。

**选项 4（丢弃）**：先确认——列出将永久删除的分支、提交和 worktree 路径，要求用户输入 `discard` 原文确认；确认后清理 worktree、`git branch -D <feature>`。

### Worktree 清理的来源判定（仅选项 1 和 4）

- 路径在 `.worktrees/` 或 `worktrees/` 下：本工作流创建的，负责清理——`cd` 到主仓库根后 `git worktree remove <path>` + `git worktree prune`。
- 其他路径：宿主环境所有，**不删**；有原生退出工具就用，没有就原地保留。

## 常见错误

| 错误 | 后果 → 纠正 |
|------|------|
| 未验证就给选项 | 合并坏代码 → 必须先有 verified 结论 |
| 开放式提问"接下来干嘛" | 含糊 → 呈现固定 4/3 选项 |
| 选项 2 清理 worktree | 用户没法迭代 PR → 只有 1 和 4 清理 |
| 先删分支再删 worktree | `branch -d` 失败 → 顺序：合并→删 worktree→删分支 |
| 在 worktree 内部执行删除 | 静默失败 → 先 `cd` 主仓库根 |
| 丢弃不要求确认 | 误删工作 → 必须输入 `discard` 原文 |

## 输出 delta

```text
Choice: 用户选择的选项
Actions: 实际执行的命令与结果
Cleanup: worktree/branch 清理状态
StageStatus: finished | kept | blocked
```

本 stage 不设置全局 readiness；`finished`/`kept` 后由编排器结束 effort 或继续剩余工作。
