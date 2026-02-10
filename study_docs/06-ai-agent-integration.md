# 06 - AI 代理集成机制

## 📋 概述

Spec Kit 支持 18+ 种 AI 编码助手，包括 Claude Code、Gemini CLI、GitHub Copilot、Cursor 等。所有代理通过统一的接口集成，使用相同的命令模板和工作流。

### 支持的代理

| 代理 | CLI 工具 | 目录 | 类型 |
|------|-----------|------|------|
| **Claude Code** | `claude` | `.claude/commands/` | CLI |
| **Gemini CLI** | `gemini` | `.gemini/commands/` | CLI |
| **GitHub Copilot** | N/A | `.github/agents/` | IDE |
| **Cursor** | `cursor-agent` | `.cursor/commands/` | CLI |
| **Windsurf** | N/A | `.windsurf/workflows/` | IDE |
| **Amazon Q** | `q` | `.amazonq/prompts/` | CLI |
| **Amp** | `amp` | `.agents/commands/` | CLI |
| **...** | ... | ... | ... |

---

## 🏗️ 集成架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    用户界面层                              │
│              (VS Code, Cursor IDE, 等)                     │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   AI 代理适配层                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │  Claude  │  │  Gemini  │  │  Copilot │            │
│  │  适配器  │  │  适配器  │  │  适配器  │            │
│  └──────────┘  └──────────┘  └──────────┘            │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   统一命令接口                               │
│              (Markdown 模板 + 参数占位符)                    │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    自动化脚本层                               │
│              (Bash / PowerShell 脚本)                       │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   文件系统层                                 │
│              (创建规范、计划、任务文档)                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 核心配置：AGENT_CONFIG

这是 AI 代理集成的**单真源**，定义在 `src/specify_cli/__init__.py` 中：

```python
AGENT_CONFIG = {
    "copilot": {
        "name": "GitHub Copilot",
        "folder": ".github/",
        "install_url": None,  # IDE-based, no CLI tool
        "requires_cli": False,
    },
    "claude": {
        "name": "Claude Code",
        "folder": ".claude/",
        "install_url": "https://docs.anthropic.com/en/docs/claude-code/setup",
        "requires_cli": True,
    },
    "gemini": {
        "name": "Gemini CLI",
        "folder": ".gemini/",
        "install_url": "https://github.com/google/gemini-cli",
        "requires_cli": True,
    },
    "cursor-agent": {
        "name": "Cursor",
        "folder": ".cursor/",
        "install_url": "https://cursor.sh/docs/installation",
        "requires_cli": True,
    },
    # ... 其他代理
}
```

### 字段说明

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `name` | string | 代理的显示名称 | "Claude Code" |
| `folder` | string | 命令文件所在目录 | ".claude/" |
| `install_url` | string | None 或安装文档 URL | "https://..." 或 None |
| `requires_cli` | bool | 是否需要 CLI 工具检查 | true/false |

---

## 📝 命令格式差异

### Markdown 格式（最常用）

**使用代理**：Claude, Cursor, Copilot, Windsurf, Amp, SHAI, IBM Bob

```markdown
---
description: Create or update feature specification
scripts:
  sh: scripts/bash/create-new-feature.sh --json "{ARGS}"
  ps: scripts/powershell/create-new-feature.ps1 -Json "{ARGS}"
---

## User Input

{ARGS}

## Guidelines

...
```

**特点**：
- 自然语言提示
- `{ARGS}` 占位符
- 支持 Markdown 格式
- 易于人类阅读

### TOML 格式

**使用代理**：Gemini, Qwen

```toml
description = "Create or update feature specification"

[scripts]
sh = "scripts/bash/create-new-feature.sh --json '{{args}}'"
ps = "scripts/powershell/create-new-feature.ps1 -Json '{{args}}'"

[prompt]
## User Input

{{args}}

## Guidelines

...
```

**特点**：
- 结构化配置
- `{{args}}` 占位符
- 支持 TOML 语法
- 适合配置驱动

### JSON 格式

**使用代理**：opencode, CodeBuddy, Qoder

```json
{
  "description": "Create or update feature specification",
  "scripts": {
    "sh": "scripts/bash/create-new-feature.sh --json \"$ARGUMENTS\"",
    "ps": "scripts/powershell/create-new-feature.ps1 -Json \"$ARGUMENTS\""
  },
  "prompt": "## User Input\n\n$ARGUMENTS\n\n## Guidelines\n\n..."
}
```

**特点**：
- 标准化格式
- `$ARGUMENTS` 占位符
- 支持复杂结构
- 易于程序解析

---

## 🎯 代理特定实现

### 1. Claude Code 集成

#### 目录结构

```
.claude/
└── commands/
    └── specify-rules.md
```

#### 命令文件格式

```markdown
---
description: Create or update feature specification
scripts:
  sh: scripts/bash/create-new-feature.sh --json "{ARGS}"
  ps: scripts/powershell/create-new-feature.ps1 -Json "{ARGS}"
---

# Feature Specification

## User Input

{ARGS}

## Guidelines

...
```

#### 上下文文件（specify-rules.md）

```markdown
# Specify Context for Claude

## Project Constitution

<!-- SPECIFY_START -->
## Article I: Library-First Principle
<!-- SPECIFY_END -->

## Current Technologies

<!-- SPECIFY_START -->
- Vite
- SQLite
- Preact Signals
<!-- SPECIFY_END -->

## Current Feature

The current feature is: [CURRENT_FEATURE]
```

**标记作用**：
- `<!-- SPECIFY_START -->` - 新内容开始标记
- `<!-- SPECIFY_END -->` - 新内容结束标记
- 标记之间的内容会被 `update-agent-context.sh` 保留

#### 特殊处理

```python
# 处理 Claude CLI 的 migrate-installer 问题
CLAUDE_LOCAL_PATH = Path.home() / ".claude" / "local" / "claude"

if tool == "claude":
    if CLAUDE_LOCAL_PATH.exists() and CLAUDE_LOCAL_PATH.is_file():
        return True
```

### 2. Gemini CLI 集成

#### 目录结构

```
.gemini/
└── commands/
    └── specify.toml
```

#### 命令文件格式（TOML）

```toml
description = "Create or update feature specification"

[scripts]
sh = "scripts/bash/create-new-feature.sh --json '{{args}}'"
ps = "scripts/powershell/create-new-feature.ps1 -Json '{{args}}'"

[prompt]
# Feature Specification

## User Input

{{args}}

## Guidelines

...
```

#### TOML 优势

1. **简洁**：比 YAML 或 JSON 更简洁
2. **类型安全**：明确的类型定义
3. **注释支持**：原生支持注释
4. **无引号地狱**：字符串不需要引号（大部分情况）

### 3. GitHub Copilot 集成

#### 目录结构

```
.github/
└── agents/
    └── specify.md
```

#### Chat Mode 格式

```markdown
---
description: Create or update feature specification
mode: speckit.specify
---

# Feature Specification

## User Input

{ARGS}

## Guidelines

...
```

**特殊字段**：
- `mode`: 指定 Copilot Chat 的命令模式

#### 无 CLI 依赖

```python
"copilot": {
    "name": "GitHub Copilot",
    "folder": ".github/",
    "install_url": None,  # IDE-based
    "requires_cli": False,  # 不检查 CLI
}
```

### 4. Cursor 集成

#### 目录结构

```
.cursor/
└── commands/
    └── specify.md
```

#### 命令文件格式

```markdown
---
description: Create or update feature specification
scripts:
  sh: scripts/bash/create-new-feature.sh --json "{ARGS}"
  ps: scripts/powershell/create-new-feature.ps1 -Json "{ARGS}"
---

# Feature Specification

## User Input

{ARGS}

## Guidelines

...
```

---

## 🔌 代理检测与选择

### 自动检测

`update-agent-context.sh` 脚本根据目录结构检测代理：

```bash
detect_agent_type() {
    if [ -f ".claude/commands/specify-rules.md" ]; then
        echo "claude"
    elif [ -f ".gemini/commands/specify.toml" ]; then
        echo "gemini"
    elif [ -f ".github/agents/specify.md" ]; then
        echo "copilot"
    elif [ -f ".cursor/commands/specify.md" ]; then
        echo "cursor-agent"
    elif [ -f ".windsurf/workflows/specify.md" ]; then
        echo "windsurf"
    elif [ -f ".amazonq/prompts/specify.md" ]; then
        echo "q"
    else
        echo ""
    fi
}
```

### CLI 交互式选择

```python
# 在 CLI 中选择 AI 代理
ai_choices = {key: config["name"] for key, config in AGENT_CONFIG.items()}
selected_ai = select_with_arrows(ai_choices, "Choose your AI assistant:", "copilot")
```

### 用户指定

```bash
# 通过参数指定
specify init my-project --ai claude
specify init my-project --ai gemini
```

---

## 🎨 命令生成流程

### 模板包生成

Spec Kit 在 GitHub Release 中为每个代理生成预配置的模板包：

```
spec-kit-template-{agent}-{script_type}.zip
```

**包内容**：

```
spec-kit-template-claude-sh/
├── .claude/
│   └── commands/
│       ├── specify.md
│       ├── plan.md
│       ├── tasks.md
│       ├── implement.md
│       └── ...
├── .specify/
│   ├── memory/
│   │   └── constitution.md
│   ├── scripts/
│   │   ├── check-prerequisites.sh
│   │   ├── common.sh
│   │   ├── create-new-feature.sh
│   │   ├── setup-plan.sh
│   │   └── update-agent-context.sh
│   └── templates/
│       ├── plan-template.md
│       ├── spec-template.md
│       └── tasks-template.md
├── .vscode/
│   └── settings.json
└── README.md
```

### 生成脚本

`.github/workflows/scripts/create-release-packages.sh` 负责生成所有代理的模板包：

```bash
# 定义所有代理
ALL_AGENTS=(claude gemini copilot cursor-agent qwen windsurf)

# 为每个代理生成包
for agent in "${ALL_AGENTS[@]}"; do
    for script_type in "sh" "ps"; do
        # 生成对应格式的命令文件
        case $agent in
            claude|cursor-agent|copilot|windsurf)
                generate_commands "$agent" "md" "\$ARGUMENTS" "$base_dir/.claude/commands"
                ;;
            gemini|qwen)
                generate_commands "$agent" "toml" "{{args}}" "$base_dir/.gemini/commands"
                ;;
        esac
        
        # 创建 ZIP 包
        cd "$base_dir"
        zip -r "$output_dir/spec-kit-template-${agent}-${script_type}-${VERSION}.zip" .
    done
done
```

---

## 🔄 上下文更新机制

### 自动更新流程

```
1. 代理运行 /speckit.plan 命令
   │
2. 脚本执行：update-agent-context.sh __AGENT__
   │
3. 检测当前代理类型
   │
4. 定位对应的上下文文件
   │
5. 在 <!-- SPECIFY_START --> 和 <!-- SPECIFY_END --> 之间插入新技术
   │
6. AI 代理重新加载上下文
```

### 标记保护机制

```bash
update_agent_file() {
    local agent_file="$1"
    
    # 使用临时文件
    local temp_file="${agent_file}.tmp"
    cp "$agent_file" "$temp_file"
    
    # 在标记之后插入内容
    sed -i "/<!-- SPECIFY_END -->/i\\
## New Technologies\\
\\
- $technologies\\
" "$temp_file"
    
    # 替换原文件
    mv "$temp_file" "$agent_file"
}
```

**保护机制**：
1. 备份到临时文件
2. 在标记之间插入新内容
3. 保留手动添加的内容
4. 仅在必要时更新

---

## 🎯 添加新代理

### 步骤指南

#### 1. 添加到 AGENT_CONFIG

在 `src/specify_cli/__init__.py` 中添加：

```python
AGENT_CONFIG = {
    # ... 现有代理
    "new-agent-cli": {  # 使用实际 CLI 工具名
        "name": "New Agent Display Name",
        "folder": ".newagent/",
        "install_url": "https://example.com/install",
        "requires_cli": True,  # False 如果是 IDE-based
    },
}
```

**重要**：
- 字典键必须是**实际的 CLI 工具名**
- 例如使用 `"cursor-agent"` 而不是 `"cursor"`

#### 2. 确定命令格式

检查代理支持哪种格式：
- Markdown (`.md`)
- TOML (`.toml`)
- JSON (`.json`)

#### 3. 添加到生成脚本

修改 `.github/workflows/scripts/create-release-packages.sh`：

```bash
# 添加到代理列表
ALL_AGENTS=(claude gemini copilot cursor-agent new-agent-cli)

# 添加 case 语句
case $agent in
    # ... 现有 case
    new-agent-cli)
        # 根据代理支持的格式
        generate_commands "$agent" "md" "\$ARGUMENTS" "$base_dir/.newagent/commands"
        ;;
esac
```

#### 4. 更新 CLI 帮助

修改 `--ai` 参数的 help 文本：

```python
ai_assistant: str = typer.Option(
    None, 
    "--ai", 
    help="AI assistant to use: claude, gemini, copilot, cursor-agent, new-agent-cli, or q"
)
```

#### 5. 更新文档

更新 `AGENTS.md` 和 `README.md`：

```markdown
| Agent | Directory | Format | CLI Tool |
|--------|-----------|---------|-----------|
| New Agent | .newagent/ | Markdown | new-agent-cli |
```

#### 6. 创建测试

测试新代理集成：

```bash
# 初始化新项目
specify init test-new-agent --ai new-agent-cli

# 验证目录结构
ls -la .newagent/

# 测试命令
# 根据代理的调用方式测试
```

---

## 📊 代理特性对比

| 特性 | Claude | Gemini | Copilot | Cursor | Windsurf |
|------|---------|---------|---------|--------|----------|
| **CLI 支持** | ✅ | ✅ | ❌ | ✅ | ❌ |
| **命令格式** | Markdown | TOML | Markdown | Markdown | Markdown |
| **参数占位符** | `{ARGS}` | `{{args}}` | `{ARGS}` | `{ARGS}` | `{ARGS}` |
| **目录** | `.claude/` | `.gemini/` | `.github/` | `.cursor/` | `.windsurf/` |
| **上下文文件** | specify-rules.md | specify.toml | specify.md | specify.md | specify.md |
| **上下文更新** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **多项目支持** | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## 🔍 故障排除

### 问题 1: CLI 工具检查失败

**症状**：
```
[red]Error:[/red] Claude CLI is required for Claude projects
```

**解决方案**：

1. 确认工具已安装：
```bash
which claude  # Bash
Get-Command claude  # PowerShell
```

2. 检查 AGENT_CONFIG 配置：
```python
"claude": {
    "requires_cli": True,  # 确认设置为 True
    "install_url": "https://...",
}
```

3. 对于 Claude CLI 特殊情况：
```python
# 检查是否使用 migrate-installer
CLAUDE_LOCAL_PATH = Path.home() / ".claude" / "local" / "claude"
```

### 问题 2: 上下文更新失败

**症状**：
```
[specify] Warning: Failed to update agent context
```

**解决方案**：

1. 检查标记是否存在：
```bash
grep "<!-- SPECIFY_START -->" .claude/commands/specify-rules.md
```

2. 验证文件权限：
```bash
ls -la .claude/commands/specify-rules.md
```

3. 手动更新：
```bash
# 在标记之间添加内容
# <!-- SPECIFY_START -->
# New content here
# <!-- SPECIFY_END -->
```

### 问题 3: 参数占位符不工作

**症状**：
```
Command executed with literal "{ARGS}" instead of actual arguments
```

**解决方案**：

1. 确认使用正确的占位符：
   - Claude: `{ARGS}`
   - Gemini: `{{args}}`
   - Copilot: `{ARGS}`

2. 检查命令文件格式：
```markdown
# Claude (Markdown)
scripts:
  sh: "scripts/bash/script.sh --json \"{ARGS}\""

# Gemini (TOML)
[scripts]
sh = "scripts/bash/script.sh --json '{{args}}'"
```

---

## 🎓 AI 代理集成学习要点

1. **统一配置**：AGENT_CONFIG 作为单真源
2. **格式适配**：支持 Markdown/TOML/JSON 多种格式
3. **CLI vs IDE**：区分需要 CLI 的代理和 IDE 原生代理
4. **上下文管理**：通过标记系统自动更新代理上下文
5. **双脚本支持**：Bash 和 PowerShell 脚本
6. **自动检测**：根据目录结构自动检测代理类型
7. **占位符差异**：不同代理使用不同的占位符
8. **模板包生成**：GitHub Release 中预配置所有代理
9. **扩展性**：易于添加新代理
10. **向后兼容**：保持对现有代理的兼容性

下一节是总结文档。