# Claude Code 使用指南

## 目录

1. [概述](#1-概述)
2. [核心概念](#2-核心概念)
3. [配置体系](#3-配置体系)
4. [MCP 工具接入](#4-mcp-工具接入)
5. [Context 加载机制](#5-context-加载机制)
6. [Hooks 机制](#6-hooks-机制)
7. [Plugins 插件系统](#7-plugins-插件系统)
8. [Keybindings 快捷键](#8-keybindings-快捷键)
9. [环境变量](#9-环境变量)
10. [Permissions 权限管理](#10-permissions-权限管理)
11. [提升使用体验](#11-提升使用体验)
12. [常用命令](#12-常用命令)
13. [最佳实践](#13-最佳实践)

---

## 1. 概述

Claude Code 是 Anthropic 官方 CLI 工具，提供 AI 辅助编程能力。支持 Linux、macOS、Windows，通过终端与 Claude 对话来完成代码编写、调试、重构等任务。

### 核心能力

| 能力 | 说明 |
|------|------|
| 代码生成 | 根据自然语言描述生成代码 |
| 代码编辑 | 读取、修改、调试现有代码 |
| 文件操作 | 创建、删除、移动文件 |
| Bash 命令 | 执行 shell 命令 |
| 多模态 | 支持读取图片、PDF 等 |
| 工具调用 | 集成 MCP 工具和自定义工具 |

---

## 2. 核心概念

### 2.1 与 Claude API 的区别

| 特性 | Claude Code | Claude API |
|------|-------------|------------|
| 交互方式 | 终端对话式 | HTTP 请求 |
| 会话管理 | 自动维护 | 手动管理 |
| 工具调用 | 内置 + MCP | 需自行实现 |
| 权限控制 | 自动询问 | 自行处理 |
| 项目感知 | 自动加载上下文 | 需自行传入 |

### 2.2 核心实体

- **Session**: 对话会话，存储在 `~/.claude/sessions/` 目录
- **Project**: 项目目录，对应一个会话的上下文
- **Context**: 上下文集合，包括配置、文件、对话历史
- **Tool**: Claude Code 可调用的工具（Read/Edit/Bash 等）
- **MCP Server**: Model Context Protocol 服务器，扩展工具能力
- **Skill**: 技能模块，提供特定领域的指令模板
- **Hook**: 钩子，在特定事件触发时执行自定义脚本
- **Plugin**: 插件，扩展 Claude Code 功能的软件包

### 2.3 工作流程

```
用户输入
    │
    ▼
┌─────────────┐
│   Parser    │  解析输入，识别命令/技能
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Memory    │  加载项目上下文、CLAUDE.md
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Rules     │  应用项目规则
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   LLM       │  Claude API 调用
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Tool Call  │  执行工具（Read/Edit/Bash/MCP）
└──────┬──────┘
       │
       ▼
  返回结果
```

---

## 3. 配置体系

### 3.1 配置优先级

Claude Code 配置采用**层叠覆盖**机制，优先级从高到低：

```
项目级 .claude/settings.json    ← 最高优先级
用户级 ~/.claude/settings.json
全局级 /etc/claude/settings.json
```

### 3.2 主配置文件结构

```json
{
  // 环境变量配置
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.anthropic.com",
    "ANTHROPIC_MODEL": "claude-opus-4-7",
    "API_TIMEOUT_MS": "300000"
  },

  // 主题配置
  "theme": "dark",

  // 已启用的插件
  "enabledPlugins": {
    "plugin-name": true
  },

  // 插件市场
  "extraKnownMarketplaces": {
    "plugin-name": {
      "source": {
        "source": "github",
        "repo": "owner/repo"
      }
    }
  },

  // MCP 服务器配置
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem"],
      "env": {
        "KEY": "value"
      }
    }
  },

  // Hooks 配置
  "hooks": {
    "preCommand": ["echo 'Before: $COMMAND'"],
    "postCommand": ["echo 'After: $COMMAND'"]
  },

  // 允许的命令（无需每次询问）
  "allow": [
    "npm:*",
    "git:*"
  ],

  // 权限配置
  "permissions": {
    "defaultAllow": "ask",
    "bash": "allow"
  },

  // 状态栏配置
  "statusLine": {
    "type": "command",
    "command": "bash -c 'your status command'"
  }
}
```

### 3.3 配置文件位置

```
~/.claude/
├── settings.json           # 主配置
├── settings.local.json     # 本地覆盖配置
├── keybindings.json        # 快捷键配置
├── permissions.json        # 权限配置
├── allowed_commands.json   # 允许的命令列表
├── history.jsonl           # 对话历史
├── sessions/               # 会话目录
│   └── session-id/
├── plugins/                # 插件目录
│   └── cache/
├── cache/                  # 缓存目录
├── debug/                  # 调试日志
└── plans/                  # 计划缓存

project/
├── .claude/
│   ├── settings.json       # 项目级配置
│   ├── mcp.json           # MCP 配置
│   └── CLAUDE.md          # 项目文档
└── CLAUDE.md              # 项目根文档
```

### 3.4 settings.local.json

用于覆盖全局配置，本地专用：

```json
{
  "env": {
    "DEBUG": "true"
  }
}
```

---

## 4. MCP 工具接入

### 4.1 MCP 概述

MCP (Model Context Protocol) 是一种标准协议，用于扩展 Claude 的工具能力。通过 MCP，可以将外部工具和数据源接入 Claude Code。

### 4.2 MCP 服务器配置

#### 方式一：通过 mcp.json（项目级）

在项目 `.claude/mcp.json` 中配置：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./workspace"]
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "xoxb-..."
      }
    }
  }
}
```

#### 方式二：通过 settings.json（全局）

在 `~/.claude/settings.json` 中配置：

```json
{
  "mcpServers": {
    "server-name": {
      "command": "path/to/command",
      "args": ["arg1", "arg2"],
      "env": {
        "ENV_VAR": "value"
      },
      "disabled": false
    }
  }
}
```

### 4.3 常用 MCP 服务器

| 服务器 | 命令 | 用途 |
|--------|------|------|
| filesystem | npx -y @modelcontextprotocol/server-filesystem | 文件系统操作 |
| github | npx -y @modelcontextprotocol/server-github | GitHub API 操作 |
| slack | npx -y @modelcontextprotocol/server-slack | Slack 消息发送 |
| sqlite | npx -y @modelcontextprotocol/server-sqlite | SQLite 数据库 |
| fetch | npx -y @modelcontextprotocol/server-fetch | HTTP 请求 |

### 4.4 自定义 MCP 服务器

```json
{
  "mcpServers": {
    "my-custom-server": {
      "command": "node",
      "args": ["/path/to/server.js"],
      "env": {
        "MY_API_KEY": "secret"
      }
    }
  }
}
```

MCP 服务器需要实现 JSON-RPC 协议，提供 `tools` 列表。

### 4.5 MCP 调试

```bash
# 查看 MCP 连接状态
# 在 Claude Code 中使用 /mcp 命令查看
```

---

## 5. Context 加载机制

### 5.1 Context 组成

Claude Code 的 Context 由以下部分组成：

```
Context
├── 项目配置 (.claude/)
│   ├── settings.json
│   ├── mcp.json
│   └── CLAUDE.md
├── 项目根文档 (向上查找)
│   ├── ./CLAUDE.md
│   ├── ../CLAUDE.md
│   └── ... (直到根目录)
├── 对话历史
├── 已加载的文件内容
├── 工具执行结果
└── Memory 记录
```

### 5.2 CLAUDE.md 加载机制

```
工作目录: /home/user/project/src
    │
    ▼
检查 ./CLAUDE.md        → 不存在
检查 ../CLAUDE.md        → 不存在
检查 ../../CLAUDE.md     → 不存在
检查 /home/user/CLAUDE.md → 不存在
检查 /home/CLAUDE.md     → 不存在
检查 /CLAUDE.md          → 不存在
    │
    ▼
最终: /home/user/project/CLAUDE.md (如果存在) 被加载
      或 /home/user/CLAUDE.md (如果存在) 被加载
```

**注意**：项目目录下的 `.claude/CLAUDE.md` 优先级高于向上查找找到的 CLAUDE.md。

### 5.3 CLAUDE.md 结构示例

```markdown
# Project Name

## Overview
项目描述和目的

## Tech Stack
- Framework: React 18
- Build: Vite
- Styling: Tailwind CSS

## Key Commands
- `npm run dev` - 启动开发服务器
- `npm run build` - 构建生产版本

## Project Structure
src/
├── components/
├── pages/
└── utils/

## Code Conventions
- 使用 TypeScript
- 组件使用 PascalCase
- 工具函数使用 camelCase

## Important Notes
- 遵循特定的代码规范
- 部署流程说明
```

### 5.4 上下文限制

Claude Code 会根据上下文窗口大小自动管理和截断内容。可以通过 `/compact` 命令手动压缩历史。

---

## 6. Hooks 机制

### 6.1 Hook 类型

| Hook | 触发时机 | 可用变量 |
|------|----------|----------|
| preCommand | 命令执行前 | `$COMMAND` |
| postCommand | 命令执行后 | `$COMMAND`, `$EXIT_CODE` |
| preToolUse | 工具使用前 | `$TOOL_NAME` |
| postToolUse | 工具使用后 | `$TOOL_NAME`, `$ERROR` |
| onError | 错误发生时 | `$ERROR` |
| onExit | 退出时 | - |

### 6.2 Hooks 配置示例

```json
{
  "hooks": {
    "preCommand": [
      "echo 'Running: $COMMAND'",
      "git diff --quiet && echo 'Working tree clean' || echo 'Uncommitted changes!'"
    ],
    "postCommand": [
      "echo 'Completed: $COMMAND (exit $EXIT_CODE)'"
    ]
  }
}
```

### 6.3 应用场景

**自动检查 git 状态：**
```json
{
  "hooks": {
    "preCommand": [
      "git diff --quiet && echo 'clean' || echo 'dirty'"
    ]
  }
}
```

**记录命令执行日志：**
```json
{
  "hooks": {
    "postCommand": [
      "echo '[$(date)] $COMMAND -> $EXIT_CODE' >> ~/.claude/command.log"
    ]
  }
}
```

**安全警告：**
```json
{
  "hooks": {
    "preCommand": [
      "if echo '$COMMAND' | grep -qE '(rm -rf|sudo rm)'; then echo 'WARNING: Dangerous command!'; fi"
    ]
  }
}
```

---

## 7. Plugins 插件系统

### 7.1 插件配置

```json
{
  "extraKnownMarketplaces": {
    "plugin-name": {
      "source": {
        "source": "github",
        "repo": "owner/repo"
      }
    }
  },
  "enabledPlugins": {
    "plugin-name@version": true
  }
}
```

### 7.2 常用插件

| 插件 | 说明 |
|------|------|
| claude-hud | HUD 状态栏显示 |
| claude-code-guide | Claude Code 使用指南（内置） |

### 7.3 HUD 插件

用户当前启用了 `claude-hud` 插件，显示动态状态栏。

---

## 8. Keybindings 快捷键

### 8.1 默认快捷键

| 快捷键 | 功能 |
|--------|------|
| Ctrl+C | 取消当前操作 |
| Ctrl+L | 清屏 |
| Ctrl+S | 提交消息 |
| Ctrl+Z | 撤销 |
| Ctrl+Shift+P | 新建会话 |

### 8.2 自定义快捷键配置

文件位置：`~/.claude/keybindings.json`

```json
{
  "keybindings": [
    {
      "key": "ctrl+s",
      "action": "submit",
      "description": "提交消息"
    },
    {
      "key": "ctrl+c",
      "action": "cancel",
      "description": "取消操作"
    },
    {
      "key": "ctrl+l",
      "action": "clear",
      "description": "清屏"
    },
    {
      "key": "alt+1",
      "action": "newSession",
      "description": "新会话"
    }
  ]
}
```

---

## 9. 环境变量

### 9.1 常用环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| ANTHROPIC_API_KEY | API 密钥 | - |
| ANTHROPIC_BASE_URL | API 地址 | https://api.anthropic.com |
| ANTHROPIC_MODEL | 默认模型 | claude-opus-4-7 |
| API_TIMEOUT_MS | 请求超时(ms) | 300000 |
| CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC | 禁用非必要流量 | 0 |
| CLAUDE_CONFIG_DIR | 配置目录 | ~/.claude |

### 9.2 用户当前配置

```json
{
  "ANTHROPIC_BASE_URL": "https://api.minimaxi.com/anthropic",
  "ANTHROPIC_MODEL": "MiniMax-M2.7",
  "API_TIMEOUT_MS": "3000000"
}
```

### 9.3 模型选择

通过环境变量或会话内命令切换模型：

```bash
# 使用 Opus
/claude opus

# 使用 Sonnet
/claude sonnet

# 使用 Haiku
/claude haiku
```

---

## 10. Permissions 权限管理

### 10.1 权限类型

| 权限 | 说明 |
|------|------|
| bash | 执行 Bash 命令 |
| read | 读取文件 |
| write | 写入文件 |
| tools | 使用工具 |
| mcp | 调用 MCP 工具 |

### 10.2 权限配置

```json
{
  "permissions": {
    "defaultAllow": "ask",
    "bash": "allow",
    "mcp": "ask"
  },
  "allow": [
    "npm:*",
    "git:*",
    "bash:*"
  ]
}
```

### 10.3 allow 规则

使用通配符匹配命令：

| 规则 | 匹配示例 |
|------|----------|
| `npm:*` | npm install, npm test |
| `git:*` | git commit, git push |
| `bash:ls` | 仅 ls 命令 |
| `bash:rm -i*` | 带 -i 参数的 rm |

### 10.4 权限提升

某些命令（如 `sudo rm`）会触发额外的权限确认。

---

## 11. 提升使用体验

### 11.1 创建项目 CLAUDE.md

为每个项目创建 `CLAUDE.md`，帮助 Claude 理解项目上下文：

```markdown
# My Project

## Tech Stack
- Node.js 18+
- Express
- PostgreSQL

## Commands
- `npm run dev` - 开发
- `npm test` - 测试

## Architecture
src/
├── routes/
├── models/
└── middleware/
```

### 11.2 配置常用命令白名单

```json
{
  "allow": [
    "npm:*",
    "git:*",
    "docker:*",
    "kubectl:*",
    "python:*"
  ]
}
```

### 11.3 配置 MCP 服务器

根据项目需求添加 MCP 服务器：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_..."
      }
    }
  }
}
```

### 11.4 配置 Hooks

```json
{
  "hooks": {
    "preCommand": [
      "git diff --quiet && echo 'clean' || echo 'Working tree has changes'"
    ]
  }
}
```

### 11.5 使用 Skills

Skills 提供特定领域的指令模板：

```bash
# 使用安全审查 skill
/security-review

# 使用代码审查 skill
/review
```

### 11.6 使用 Plan Mode

对于复杂任务，使用 Plan Mode 进行设计：

```bash
# 进入计划模式
/plan

# 或描述任务后 Claude 会建议进入计划模式
```

---

## 12. 常用命令

### 12.1 启动命令

```bash
# 启动 Claude Code
claude

# 指定项目目录
claude /path/to/project

# 指定模型
claude --model opus

# 查看帮助
claude --help
```

### 12.2 会话命令

| 命令 | 功能 |
|------|------|
| `/new` | 新建会话 |
| `/session` | 切换会话 |
| `/sessions` | 列出所有会话 |
| `/compact` | 压缩对话历史 |
| `/clear` | 清屏 |
| `/exit` | 退出 |

### 12.3 配置命令

| 命令 | 功能 |
|------|------|
| `/config` | 查看/修改配置 |
| `/keybindings` | 快捷键设置 |
| `/permissions` | 权限管理 |
| `/mcp` | MCP 服务器管理 |

### 12.4 项目命令

| 命令 | 功能 |
|------|------|
| `/init` | 初始化项目文档 |
| `/memory` | 查看/管理记忆 |
| `/plan` | 进入计划模式 |

### 12.5 技能命令

| 命令 | 功能 |
|------|------|
| `/security-review` | 安全审查 |
| `/review` | PR 审查 |
| `/claude-api` | Claude API 相关 |

---

## 13. 最佳实践

### 13.1 项目设置

1. **创建项目 CLAUDE.md**：描述项目结构、技术栈、常用命令
2. **配置权限白名单**：避免频繁确认常用命令
3. **配置 Hooks**：自动检查 git 状态等

### 13.2 写作建议

1. **描述清晰**：明确说明要实现的功能和约束
2. **提供上下文**：给出相关代码和文件路径
3. **迭代改进**：先让 Claude 生成，再提出修改意见

### 13.3 代码审查

1. **使用 /review**：对 PR 进行审查
2. **使用 /security-review**：进行安全审查
3. **提供具体反馈**：指出具体文件和行号

### 13.4 调试问题

1. **提供完整错误信息**：包括错误日志和堆栈
2. **说明复现步骤**：如何一步步复现问题
3. **限制范围**：指定具体文件或模块

### 13.5 性能优化

1. **定期 `/compact`**：压缩对话历史，保持响应速度
2. **使用合适模型**：简单任务用 Haiku，复杂任务用 Opus
3. **明确范围**：避免让 Claude 处理无关的大文件

### 13.6 安全注意事项

1. **不提交密钥**：API Key 等敏感信息使用环境变量
2. **检查 Hooks**：确保不会意外泄露敏感信息
3. **验证生成的命令**：特别是删除类操作

---

## 附录 A：文件位置汇总

| 文件/目录 | 位置 | 说明 |
|-----------|------|------|
| 全局配置 | `~/.claude/settings.json` | 用户级配置 |
| 项目配置 | `./.claude/settings.json` | 项目级配置 |
| MCP 配置 | `./.claude/mcp.json` | MCP 服务器 |
| 项目文档 | `./CLAUDE.md` | 项目上下文 |
| 快捷键 | `~/.claude/keybindings.json` | 自定义快捷键 |
| 会话 | `~/.claude/sessions/` | 存储会话 |
| 历史 | `~/.claude/history.jsonl` | 对话历史 |
| 日志 | `~/.claude/debug/` | 调试日志 |

## 附录 B：配置示例

### 完整 settings.json 示例

```json
{
  "env": {
    "ANTHROPIC_API_KEY": "sk-...",
    "ANTHROPIC_BASE_URL": "https://api.anthropic.com",
    "ANTHROPIC_MODEL": "claude-opus-4-7",
    "API_TIMEOUT_MS": "300000"
  },
  "theme": "dark",
  "enabledPlugins": {
    "claude-hud": true
  },
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./workspace"]
    }
  },
  "hooks": {
    "preCommand": [
      "git diff --quiet && echo 'clean' || echo 'dirty'"
    ]
  },
  "allow": [
    "npm:*",
    "git:*",
    "docker:*"
  ],
  "permissions": {
    "defaultAllow": "ask",
    "bash": "allow"
  }
}
```

---

## 参考链接

- [Claude Code 官方文档](https://docs.anthropic.com/claude-code)
- [Anthropic API 文档](https://docs.anthropic.com/)
- [MCP GitHub](https://github.com/modelcontextprotocol)