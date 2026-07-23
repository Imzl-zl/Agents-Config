# Using Git Worktrees — 隔离工作区

这是 `/zhanggui` 的 supporting stage，不是独立 skill。执行开始前需要隔离时读取：高风险改动、并行子任务需要互不干扰的写范围、当前工作区有未提交改动，或用户要求隔离。

**核心原则：先检测已有隔离，再用宿主原生工具，最后才手动 git worktree。绝不和宿主对着干。**

## Step 0: 检测已有隔离

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
git rev-parse --show-superproject-working-tree 2>/dev/null   # 有输出 = 在 submodule 里，按普通仓库处理
```

- `GIT_DIR != GIT_COMMON` 且不是 submodule：**已经在隔离工作区里**，跳到 Step 2，不要再套一层。
- 普通仓库：若用户未在指令中声明过 worktree 偏好，先征求一次同意：
  > 要不要建一个隔离 worktree？可以保护你当前分支不被改动。
  已有声明的偏好直接执行不再问；用户拒绝则原地工作，跳到 Step 2。

## Step 1: 创建隔离工作区

### 1a. 宿主原生工具优先

已有 `EnterWorktree`、`WorktreeCreate`、`/worktree` 之类的原生工具时**必须用它**，然后跳到 Step 2。原生工具自动处理目录、分支和清理；绕过它手动 `git worktree add` 会制造宿主看不见的幽灵状态。

### 1b. Git 手动后备（仅当无原生工具）

目录优先级：用户声明的偏好 > 已存在的 `.worktrees/`（胜过 `worktrees/`）> 默认 `.worktrees/`。

创建前必须验证目录被 git 忽略：

```bash
git check-ignore -q .worktrees 2>/dev/null || { echo ".worktrees" >> .gitignore && git add .gitignore; }
git worktree add ".worktrees/$BRANCH_NAME" -b "$BRANCH_NAME"
cd ".worktrees/$BRANCH_NAME"
```

沙箱拒绝创建时：告知用户，原地工作并继续 Step 2。

## Step 2: 项目初始化

按项目类型自动检测并安装依赖（package.json → `npm install`；Cargo.toml → `cargo build`；requirements.txt/pyproject.toml → pip/poetry；go.mod → `go mod download`）。

## Step 3: 验证干净基线

跑项目测试套件。**失败**：报告失败项，问是继续还是先调查——分不清新 bug 和存量问题的基线没有意义。**通过**：报告就绪。

```text
Worktree 就绪：<full-path>
基线测试通过（<N> 个，0 失败）
可以开始 <task>
```

## Red Flags

- 已检测到隔离还再建一层。
- 有原生 worktree 工具却手写 `git worktree add`（第一大错误）。
- 项目本地目录未验证 ignore 就创建。
- 跳过基线测试，或基线失败不问就继续。

## 输出 delta

```text
Workspace: 路径与分支
Baseline: 测试命令与结果
StageStatus: isolated | in-place | blocked
```

收尾时的 worktree 清理规则见 `../finishing-a-development-branch/STAGE.md`；本 stage 不设置全局 readiness。
