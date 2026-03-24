# Agents Config

一套用于 AI 编程助手（Claude Code / Codex）的项目协作与任务管理配置集合。

---

## 核心组件

### 1. Init Project — 项目协作体系初始化

**主要介绍**

这是本仓库的核心命令，用于在项目中快速建立标准化的协作文件体系。

#### 功能

一键初始化项目的协作基础设施，包括：

| 文件 | 用途 |
|------|------|
| `CLAUDE.md` / `AGENTS.md` | 项目唯一 canonical 规则文件 |
| `tools.md` | 稳定协作知识、常用命令、模式、坑点 |
| `memory.md` | 当前状态快照 + 最近活跃窗口 |
| `memory/archive/YYYY-MM.md` | 按月追加的重要协作历史 |

#### 核心特性

- **两种模式自动识别**：全新初始化 / 已有规则归一
- **自动备份**：重写前自动备份到 `init-project.backup/[timestamp]/`
- **智能提取**：从现有规则、文档、Git 历史、代码结构中自动提取关键信息
- **无参数设计**：用户只需执行一个命令，不需要选择模板类型

#### 使用方式

**Claude Code**：
```
/init-project
```

**Codex**：
```
/prompts:init-project
```

#### 提供版本

| 文件 | 主规则文件 | 适用工具 |
|------|-----------|---------|
| `.claude/commands/init-project.md` | `CLAUDE.md` | Claude Code |
| `.codex/prompts/init-project.md` | `AGENTS.md` | Codex |

---

## 辅助组件

### 2. Taskmaster — 统一任务执行协议

**位置**：`.codex/skills/taskmaster/`

用于多步骤任务的跟踪和自主执行，支持三种任务形态：

| 形态 | 适用场景 | 真值文件 |
|------|---------|---------|
| **Single Task** | 单一交付物 | `TODO.csv` / `SPEC.md + TODO.csv + PROGRESS.md` |
| **Epic Task** | 多交付物、有依赖链 | `EPIC.md + SUBTASKS.csv + PROGRESS.md` |
| **Batch Task** | 同构行级批量工作 | `workers-input.csv + workers-output.csv` |

**核心原则**：
- 磁盘上的真值文件优于内存
- 无显式验证不得标记完成
- 支持冷启动恢复

**触发词**：`跟踪任务`、`做个计划`、`从零开始`、`长时任务`、`自主执行` 等

---

### 3. Todo List CSV — 轻量任务跟踪

**位置**：`.codex/skills/todo-list-csv/`

更简单的任务跟踪方式，在项目根目录创建 CSV 文件跟踪进度：

```csv
id,item,status,done_at,notes
1,复现问题,IN_PROGRESS,,
2,加回归测试,TODO,,
3,修复实现,TODO,,
4,运行测试/构建,TODO,,
```

**特点**：
- 与 `update_plan` 双轨同步
- 全部完成后自动删除 CSV
- 提供自动化脚本 `scripts/todo_csv.py`

**触发条件**：需要修改项目且任务有多个可独立验收步骤

---

### 4. AGENTS.md — 全局 Agent 规则

**位置**：`.codex/AGENTS.md`

定义 AI 助手的全局行为规范，包括：

| 类别 | 内容 |
|------|-----|
| 语言 | 默认中文回复 |
| 响应风格 | 不主动提议后续任务 |
| Debug-First | 拒绝静默降级，让失败暴露 |
| 工程质量 | SOLID、DRY、代码度量硬限制 |
| 安全基线 | 不硬编码密钥、参数化查询、输入校验 |
| 技能路由 | 自动匹配并加载相关 Skill |

**代码度量硬限制**：
- 函数长度：50 行
- 文件大小：300 行
- 嵌套深度：3 层
- 参数数量：3 个
- 圈复杂度：10

---

### 5. Config — 环境配置

提供 Windows 和 WSL 两套配置：

#### Windows 配置

**位置**：`.codex/config(Win).toml`

特点：
- 使用 PowerShell 原生通知
- `elevated` 沙箱模式
- 通过 `cmd /c npx` 启动 MCP 服务

#### WSL 配置

**位置**：`.codex/config(WSL).toml`

特点：
- 更宽松的权限策略
- 原生 Linux 命令
- 支持 SSE 远程 MCP 服务

---

#### 必须配置项

使用前需要修改以下配置：

##### 1. 模型提供者（必须）

```toml
[model_providers.Custom]
base_url = "http://your url/v1"  # ← 改成你的 API 地址
name = "Custom"
requires_openai_auth = true
wire_api = "responses"
```

| 配置项 | 说明 |
|-------|------|
| `base_url` | 你的 OpenAI 兼容 API 地址，如 `https://api.openai.com/v1` |
| `requires_openai_auth` | 设为 `true` 时，API Key 通过环境变量 `OPENAI_API_KEY` 或客户端认证提供 |

##### 2. MCP 服务（可选）

**Exa 搜索服务**：

```toml
[mcp_servers.exa]
env = {EXA_API_KEY = 'Your key'}  # ← 改成你的 Exa API Key
```

**Augment 上下文引擎**：

```toml
[mcp_servers.augment-context-engine]
env = {AUGMENT_API_TOKEN = "Your key"}  # ← 改成你的 Token
# AUGMENT_API_URL 可保持默认或改成你的中转地址
```

#### 配置速查表

| 服务 | 配置项 | 是否必须 | 获取方式 |
|-----|-------|---------|---------|
| Custom 模型 | `base_url` | **必须** | 你的 API 服务地址 |
| Custom 模型 | API Key | **必须** | 环境变量 `OPENAI_API_KEY` |
| Exa | `EXA_API_KEY` | 可选 | https://exa.ai |
| Augment | `AUGMENT_API_TOKEN` | 可选 | https://augmentcode.com/ 或者站内公益地址 https://acemcp.heroman.wtf/relay/|

> **提示**：如果不需要某个 MCP 服务，直接删除对应的 `[mcp_servers.xxx]` 段即可。

---

## 快速开始

### 1. 选择你的工具

- **Claude Code 用户**：使用 `.claude/commands/init-project.md`
- **Codex 用户**：使用 `.codex/` 下的全部文件

### 2. 复制配置

将对应文件复制到你的项目或全局配置目录：

```
# Claude Code
~/.claude/commands/init-project.md

# Codex
~/.codex/
├── prompts/init-project.md
├── skills/taskmaster/
├── skills/todo-list-csv/
├── AGENTS.md
└── config.toml  # 重命名自 config(Win).toml 或 config(WSL).toml
```

### 3. 在项目中初始化

进入项目目录，运行初始化命令：

```
/init-project    # Claude Code
/prompts:init-project    # Codex
```

---

## 文件结构

```
Agents Config/
├── README.md
├── .claude/
│   └── commands/
│       └── init-project.md      # Claude Code 版本初始化命令
└── .codex/
    ├── prompts/
    │   └── init-project.md      # Codex 版本初始化命令
    ├── skills/
    │   ├── taskmaster/          # 多步骤任务执行协议
    │   │   ├── SKILL.md
    │   │   └── assets/          # 模板文件
    │   └── todo-list-csv/       # 轻量任务跟踪
    │       ├── SKILL.md
    │       └── scripts/
    ├── AGENTS.md                # 全局 Agent 规则
    ├── config(Win).toml         # Windows 配置
    └── config(WSL).toml         # WSL 配置
```

---

## 设计理念

1. **真值在磁盘**：所有状态以文件形式持久化，支持冷启动恢复
2. **最小骨架**：规则文件保持短、小、硬，不堆砌阶段性历史
3. **Debug-First**：让失败暴露而非隐藏，从根因解决问题
4. **无参数设计**：用户只需一个入口，复杂分支内部处理

---

## License

MIT
