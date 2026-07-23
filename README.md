[![Linux.do 社区](https://cdn3.linux.do/original/4X/d/6/5/d65def8cc0c413f318bee2bcd1c774bc4ad109a8.png)](https://linux.do/ "访问 Linux.do 社区")

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

## Zhanggui 与参考源

### 2. Zhanggui — 当前融合运行实现

**位置**：`zhanggui/`（仓库根独立项目）

日常只需调用一次：

```text
/zhanggui
```

v0.4 只暴露一个 `/zhanggui` 入口，按 decision owner、UI mode 和全局 frontier 读取不可发现 stages。上游能力已吸收为 12 个按需 stage；短任务使用 minimal state，跨会话设计在物化前使用 DESIGN，物化后由 SPEC/EPIC 接管真值。

详细说明：

- [使用说明](zhanggui/README.md)
- [设计理念](zhanggui/docs/skill-fusion-design.md)
- [目录边界](.codex/README.md)

---

### 3. Reference Skills — 改造前参考快照

**位置**：`.codex/reference-skills/`

| 快照 | 参考内容 | 运行状态 |
|---|---|---|
| `taskmaster/` | Compact/Full/Epic/Batch、恢复和 CSV 真值协议 | 不可发现，不调用 |
| `todo-list-csv/` | 项目根 CSV、update_plan 同步与辅助脚本 | 不可发现，不调用 |

这些文件保持原始语义，只用于阅读、比较和追溯。它们不属于 `.codex/skills/`，不在 zhanggui plugin 的 `skills` 路径中，也不是 `/zhanggui` 的依赖。

其他上游参考源：

- `superpowers/`：原版 Superpowers，Codex plugin marker 已禁用。
- `skills/`：其他上游 skill 设计参考。

参考源规则见 [`.codex/reference-skills/README.md`](.codex/reference-skills/README.md)。

---


### 4. AGENTS.md — 全局 Agent 规则

**位置**：`.codex/AGENTS.md`

定义 AI 助手的全局行为规范，包括：

| 类别 | 内容 |
|------|-----|
| 语言 | 默认中文回复 |
| 响应风格 | 不主动提议后续任务 |
| Debug-First | 拒绝静默降级，让失败暴露 |
| 工程质量 | SOLID、DRY、生产代码度量与文档完整性 |
| 安全基线 | 不硬编码密钥、参数化查询、输入校验 |
| 技能路由 | 自动匹配相关能力；重型 `/zhanggui` 显式启动 |

**生产代码度量约束**：
- 函数长度：50 行
- 生产代码文件：300 行作为拆分信号
- 嵌套深度：3 层
- 参数数量：3 个
- 圈复杂度：10
- Skill、设计文档和参考资料不设硬行数上限；按职责拆分，但不为行数删除必要内容

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
- **Codex 用户**：按用途复制 `.codex/` 中的运行配置；参考快照可选保留

### 2. 复制配置

```text
# Claude Code
~/.claude/commands/init-project.md

# Codex 运行部分
~/.codex/
├── prompts/init-project.md
├── skills/zhanggui/             # 掌柜 skill（复制自仓库 zhanggui/skills/zhanggui/）
├── skills/init-project/         # 独立安装技能
├── AGENTS.md
└── config.toml                  # 重命名自 config(Win).toml 或 config(WSL).toml

# 可选参考资料，不参与运行
~/.codex/reference-skills/
```

### 3. 在项目中初始化

进入项目目录，运行初始化命令：

```
/init-project    # Claude Code
/prompts:init-project    # Codex
```

---

## 文件结构

```text
Agents Config/
├── README.md
├── superpowers/                      # 上游参考源；Codex plugin marker disabled
├── skills/                           # 上游参考源
├── zhanggui/                         # 掌柜 skill 项目（plugin + 可移植 skill + 设计文档）
│   ├── .codex-plugin/plugin.json
│   ├── README.md
│   ├── docs/                         # skill-fusion-design.md + legacy 测试材料
│   └── skills/zhanggui/              # 自包含 skill（SKILL.md + RECOVERY.md + stages/）
├── .claude/
│   └── commands/init-project.md
└── .codex/
    ├── README.md                     # 配置/参考边界
    ├── prompts/init-project.md
    ├── skills/                       # 真正安装/可发现的独立技能
    │   ├── .system/
    │   └── init-project/
    ├── reference-skills/             # 不可发现的原始参考快照
    │   ├── taskmaster/
    │   ├── todo-list-csv/
    │   └── README.md
    ├── AGENTS.md
    ├── config(Win).toml
    └── config(WSL).toml
```

---

## 设计理念

1. **单一入口**：只发现 `/zhanggui`，stage 不进入 discovery
2. **按决策分权**：业务、技术和 UI node 可分别设置 owner
3. **按需状态**：Transient 使用 minimal state，复杂设计才启用完整 WorkflowState
4. **单一真值切换**：pre-plan DESIGN 经验证 cutover 到 SPEC/EPIC，执行状态由对应 CSV 负责
5. **参考源只读**：上游仓库以 gitignored 本地克隆保留（superpowers/、skills/），fork 期改造副本存于 git 历史，不参与运行
6. **文档完整优先**：文档不设 300 行硬限制，必要时按职责拆分
7. **Debug-First**：失败保存 return point，从根因修复并用新鲜证据验证

---

## License

MIT
