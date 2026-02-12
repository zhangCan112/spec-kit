# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

**重要提示：请始终使用中文与用户交流。**

## 常用开发命令

### 安装和运行

```bash
# 安装依赖（使用 uv 进行快速的 Python 包管理）
uv sync

# 运行 specify CLI
uv run specify --help

# 或直接运行模块
python -m specify_cli --help
```

### 提交前的本地测试

修改模板或脚本时，请在提交前进行本地测试：

```bash
# 使用你的更改创建本地发布包
.github/workflows/scripts/create-release-packages.sh v1.0.0

# 复制到测试项目
cp -r .genreleases/spec-kit-template-claude-sh-1.0.0/. /path/to/test-project/

# 然后在你的 AI 代理中测试斜杠命令
```

### 版本号更新

对 `src/specify_cli/__init__.py` 的任何更改都需要：
1. 在 `pyproject.toml` 中递增版本号
2. 在 `CHANGELOG.md` 中添加变更日志条目

## 项目架构

### 核心理念：规格驱动开发（SDD）

Spec Kit 颠覆了传统开发模式：**规格说明是真实来源，而非代码**。工作流程强制执行这种分离：

1. **宪章**（`.specify/memory/constitution.md`）- 项目原则和架构规则
2. **规格说明**（`specs/###-feature/spec.md`）- WHAT 和 WHY（无技术细节）
3. **计划**（`specs/###-feature/plan.md`）- HOW 及技术栈选择
4. **任务**（`specs/###-feature/tasks.md`）- 可执行的任务分解
5. **实现** - 从上述内容生成的代码

这种分离确保规格说明在实现技术发生变化时保持稳定。

### 唯一真实来源：AGENT_CONFIG

`src/specify_cli/__init__.py` 中的 `AGENT_CONFIG` 字典是所有代理元数据的**唯一真实来源**。添加新代理时：

- 使用**实际 CLI 工具名称**作为键（例如 `"cursor-agent"`，而非 `"cursor"`）
- 这消除了在整个代码库中进行特殊映射的需求
- `check_tool()` 函数直接针对此键使用 `shutil.which(tool)`

详细的集成说明请参见 `AGENTS.md`。

### 模板系统架构

`templates/` 中的模板是**结构化提示**，用于约束 AI 行为以获得一致的输出：

- **模板即提示**：每个模板都旨在引导 LLM 生成特定格式的输出
- **强制分离**：`spec-template.md` 禁止实现细节；`plan-template.md` 要求包含这些细节
- **质量关卡**：模板内置的检查清单和验证步骤
- **占位符系统**：`{SCRIPT}`、`$ARGUMENTS`、`__AGENT__` 在运行时被代理替换

**关键见解**：模板不仅仅是文件脚手架——它们是通过构建 AI 思考需求的方式来确保规格质量的主要机制。

### 斜杠命令流程

`templates/commands/` 中的命令是 Spec Kit 的用户界面 API：

1. `/speckit.constitution` - 创建包含项目原则的 `.specify/memory/constitution.md`
2. `/speckit.specify` - 运行 `create-new-feature.sh` → 创建编号分支和规格说明
3. `/speckit.plan` - 从规格说明生成技术实现计划
4. `/speckit.tasks` - 将计划分解为可执行任务
5. `/speckit.implement` - 执行任务以构建功能
6. `/speckit.clarify` - （可选）规划前的结构化提问
7. `/speckit.analyze` - （可选）跨工件验证
8. `/speckit.checklist` - （可选）质量检查清单
9. `/speckit.taskstoissues` - 将任务转换为 GitHub 问题

### 脚本系统架构

`scripts/bash/` 和 `scripts/powershell/` 中的脚本处理跨平台自动化：

- **仓库根目录检测**：脚本搜索 `.git` 或 `.specify` 目录以查找仓库根目录
- **功能编号**：`create-new-feature.sh` 从现有最高分支/规格编号自动递增
- **分支命名**：格式 `###-short-name`（为 GitHub 限制截断至 244 字节）
- **JSON 输出模式**：`--json` 标志用于与 AI 代理的工具互操作性
- **环境变量**：`SPECIFY_FEATURE` 设置为当前分支名称供后续命令使用

### 宪章保留

`src/specify_cli/__init__.py` 中的 `ensure_constitution_from_template()` 函数在重新初始化期间保留现有宪章——仅当 `.specify/memory/constitution.md` 不存在时才从模板复制。

## 模板编辑指南

修改模板时：

1. **保持章节结构**：模板由脚本解析；保持标题层级
2. **保留占位符语法**：`{SCRIPT}`、`$ARGUMENTS`、`{{args}}` 必须符合代理预期
3. **更新 AGENTS.md**：记录任何新模式或要求
4. **本地测试**：提交前使用发布包工作流

## 重要设计模式

### 代理特定的文件格式

- **Markdown**：Claude、Cursor、Copilot、Windsurf、Amazon Q、Amp、SHAI、IBM Bob → `$ARGUMENTS`
- **TOML**：Gemini、Qwen → `{{args}}`
- **脚本占位符**：`{SCRIPT}` 替换为实际脚本路径
- **代理占位符**：`__AGENT__` 替换为代理名称

### 功能分支编号

分支命名约定使用 `###-feature-name` 格式：

1. 脚本检查远程分支（`git ls-remote`）、本地分支（`git branch`）和规格目录
2. 在所有三个来源中找到最高编号
3. 新功能自动递增到 N+1
4. 零填充到 3 位数字：`001`、`002`、`003`

### GitHub 发布集成

`init` 命令从 GitHub Releases 获取最新版本：
- API：`https://api.github.com/repos/github/spec-kit/releases/latest`
- 下载代理特定的 zip 包：`spec-kit-template-{agent}-{script_type}-{version}.zip`
- 支持 `GH_TOKEN`/`GITHUB_TOKEN` 进行身份验证请求（更高的速率限制）
- 带有用户友好错误消息的速率限制处理

## 技能集成

Spec Kit 为 AI 代理提供自定义技能（参见 `.claude/skills/speckit.*`）：

- `speckit.specify` - 功能规格说明工作流
- `speckit.plan` - 技术规划工作流
- `speckit.tasks` - 任务生成工作流
- `speckit.implement` - 实现执行工作流
- 以及更多（clarify、analyze、checklist、constitution、taskstoissues）

这些技能将命令模板包装在额外的上下文和错误处理中。
