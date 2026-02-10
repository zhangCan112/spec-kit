# 05 - 自动化脚本层

## 📋 概述

Spec Kit 的自动化脚本层是连接命令模板和实际操作的桥梁。脚本负责执行文件系统操作、Git 管理和平台特定的任务，确保跨平台兼容性。

### 脚本分类

| 类别 | 位置 | 功能 |
|------|------|------|
| **核心脚本** | `scripts/bash/` 和 `scripts/powershell/` | 执行创建新特性、设置计划等 |
| **公共函数** | `common.sh` 和 `common.ps1` | 共享的辅助函数 |
| **前置检查** | `check-prerequisites.*` | 验证工具和环境 |
| **上下文更新** | `update-agent-context.*` | 更新 AI 代理上下文 |

---

## 🔧 脚本架构设计

### 跨平台策略

Spec Kit 采用双脚本策略：

```
命令模板
    │
    ├─────────────┬─────────────┐
    ▼             ▼             ▼
Linux/macOS    Windows       公共接口
Bash 脚本    PowerShell      (JSON 输出)
```

**关键设计原则**：

1. **并行实现**：每个功能都有 Bash 和 PowerShell 版本
2. **统一接口**：两个版本接受相同参数，返回相同 JSON 格式
3. **平台检测**：自动选择合适的脚本版本
4. **公共函数**：共享逻辑放在 `common.*` 文件中

---

## 📂 目录结构

```
scripts/
├── bash/
│   ├── check-prerequisites.sh      # 前置条件检查
│   ├── common.sh                 # 公共函数库
│   ├── create-new-feature.sh     # 创建新特性（核心）
│   ├── setup-plan.sh            # 设置计划环境
│   └── update-agent-context.sh   # 更新 AI 代理上下文
└── powershell/
    ├── check-prerequisites.ps1
    ├── common.ps1
    ├── create-new-feature.ps1
    ├── setup-plan.ps1
    └── update-agent-context.ps1
```

---

## 🚀 核心脚本解析

### 1. create-new-feature.sh

这是最重要的脚本，负责创建新的功能分支和目录结构。

#### 参数解析

```bash
JSON_MODE=false
SHORT_NAME=""
BRANCH_NUMBER=""
ARGS=()

# 参数解析循环
while [ $i -le $# ]; do
    arg="${!i}"
    case "$arg" in
        --json) 
            JSON_MODE=true 
            ;;
        --short-name)
            i=$((i + 1))
            SHORT_NAME="${!i}"
            ;;
        --number)
            i=$((i + 1))
            BRANCH_NUMBER="${!i}"
            ;;
        *) 
            ARGS+=("$arg") 
            ;;
    esac
    i=$((i + 1))
done

FEATURE_DESCRIPTION="${ARGS[*]}"
```

**支持的参数**：

| 参数 | 说明 | 示例 |
|------|------|------|
| `--json` | 输出 JSON 格式 | `--json` |
| `--short-name` | 自定义短名称 | `--short-name "user-auth"` |
| `--number` | 手动指定分支号 | `--number 5` |
| `description` | 特性描述（位置参数） | `"Add user authentication"` |

#### 仓库根目录查找

```bash
find_repo_root() {
    local dir="$1"
    while [ "$dir" != "/" ]; do
        if [ -d "$dir/.git" ] || [ -d "$dir/.specify" ]; then
            echo "$dir"
            return 0
        fi
        dir="$(dirname "$dir")"
    done
    return 1
}

# 检测 Git 仓库
if git rev-parse --show-toplevel >/dev/null 2>&1; then
    REPO_ROOT=$(git rev-parse --show-toplevel)
    HAS_GIT=true
else
    REPO_ROOT="$(find_repo_root "$SCRIPT_DIR")"
    HAS_GIT=false
fi
```

这个设计支持：
- 有 Git 的仓库：使用 `git rev-parse` 获取根目录
- 无 Git 的仓库：向上搜索 `.git` 或 `.specify` 目录

#### 分支号计算算法

```bash
# 从 specs 目录获取最大号
get_highest_from_specs() {
    local specs_dir="$1"
    local highest=0
    
    if [ -d "$specs_dir" ]; then
        for dir in "$specs_dir"/*; do
            [ -d "$dir" ] || continue
            dirname=$(basename "$dir")
            number=$(echo "$dirname" | grep -o '^[0-9]\+' || echo "0")
            number=$((10#$number))  # 强制十进制
            if [ "$number" -gt "$highest" ]; then
                highest=$number
            fi
        done
    fi
    
    echo "$highest"
}

# 从 Git 分支获取最大号
get_highest_from_branches() {
    local highest=0
    
    # 获取所有分支（本地和远程）
    branches=$(git branch -a 2>/dev/null || echo "")
    
    if [ -n "$branches" ]; then
        while IFS= read -r branch; do
            # 清理分支名
            clean_branch=$(echo "$branch" | sed 's/^[* ]*//; s|^remotes/[^/]*/||')
            
            # 提取特性号
            if echo "$clean_branch" | grep -q '^[0-9]\{3\}-'; then
                number=$(echo "$clean_branch" | grep -o '^[0-9]\{3\}' || echo "0")
                number=$((10#$number))
                if [ "$number" -gt "$highest" ]; then
                    highest=$number
                fi
            fi
        done <<< "$branches"
    fi
    
    echo "$highest"
}
```

**算法特点**：
1. 从多个来源收集信息：本地分支、远程分支、specs 目录
2. 提取匹配 `###-feature-name` 模式的编号
3. 使用 `$((10#$number))` 防止八进制转换（如 `010` → 10 而不是 8）
4. 返回最大号 + 1 作为新的特性号

#### 智能分支名生成

```bash
generate_branch_name() {
    local description="$1"
    
    # 停止词列表
    local stop_words="^(i|a|an|the|to|for|of|in|on|at|by|with|from|is|are|was|were|be|been|being|have|has|had|do|does|did|will|would|should|could|can|may|might|must|shall|this|that|these|those|my|your|our|their|want|need|add|get|set)$"
    
    # 转换为小写并分割
    local clean_name=$(echo "$description" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/ /g')
    
    # 过滤停止词和短词
    local meaningful_words=()
    for word in $clean_name; do
        [ -z "$word" ] && continue
        
        # 跳过停止词
        if ! echo "$word" | grep -qiE "$stop_words"; then
            # 保留长度 ≥3 的词，或大写缩写
            if [ ${#word} -ge 3 ]; then
                meaningful_words+=("$word")
            elif echo "$description" | grep -q "\b${word^^}\b"; then
                meaningful_words+=("$word")
            fi
        fi
    done
    
    # 组合结果
    if [ ${#meaningful_words[@]} -gt 0 ]; then
        local max_words=3
        if [ ${#meaningful_words[@]} -eq 4 ]; then max_words=4; fi
        
        local result=""
        local count=0
        for word in "${meaningful_words[@]}"; do
            if [ $count -ge $max_words ]; then break; fi
            if [ -n "$result" ]; then result="$result-"; fi
            result="$result$word"
            count=$((count + 1))
        done
        echo "$result"
    fi
}
```

**算法流程**：

```
输入: "Build a photo album organizer with drag-and-drop albums"
  │
  ├─→ 转小写: "build a photo album organizer with drag-and-drop albums"
  │
  ├─→ 分割单词: ["build", "a", "photo", "album", "organizer", "with", "drag", "and", "drop", "albums"]
  │
  ├─→ 过滤停止词: ["build", "photo", "album", "organizer", "drag", "drop", "albums"]
  │  (移除: a, with, and)
  │
  ├─→ 保留 3-4 个词: ["photo", "album", "organizer"]
  │
  └─→ 组合: "photo-album-organizer"
```

**示例**：

| 输入 | 输出 |
|------|------|
| "Add user authentication system" | "user-authentication" |
| "Create a new dashboard for analytics" | "new-dashboard-analytics" |
| "Implement OAuth2 integration for API" | "oauth2-integration-api" |

#### GitHub 分支长度限制

```bash
# GitHub 强制 244 字节限制
MAX_BRANCH_LENGTH=244

if [ ${#BRANCH_NAME} -gt $MAX_BRANCH_LENGTH ]; then
    # 计算需要修剪的长度
    MAX_SUFFIX_LENGTH=$((MAX_BRANCH_LENGTH - 4))  # 3位数字 + 1个连字符
    
    # 在词边界截断
    TRUNCATED_SUFFIX=$(echo "$BRANCH_SUFFIX" | cut -c1-$MAX_SUFFIX_LENGTH)
    TRUNCATED_SUFFIX=$(echo "$TRUNCATED_SUFFIX" | sed 's/-$//')
    
    ORIGINAL_BRANCH_NAME="$BRANCH_NAME"
    BRANCH_NAME="${FEATURE_NUM}-${TRUNCATED_SUFFIX}"
    
    >&2 echo "[specify] Warning: Branch name exceeded GitHub's 244-byte limit"
    >&2 echo "[specify] Original: $ORIGINAL_BRANCH_NAME (${#ORIGINAL_BRANCH_NAME} bytes)"
    >&2 echo "[specify] Truncated to: $BRANCH_NAME (${#BRANCH_NAME} bytes)"
fi
```

#### 执行流程

```
1. 解析参数
   │
2. 查找仓库根目录
   │
3. 生成短名称
   │
4. 计算特性号
   │
5. 创建分支名 (###-short-name)
   │
6. 检查 GitHub 长度限制
   │
7. 创建 Git 分支（如果有 Git）
   │
8. 创建 specs/###-short-name/ 目录
   │
9. 复制 spec-template.md
   │
10. 输出 JSON 或文本
```

### 2. setup-plan.sh

这个脚本准备计划阶段的环境。

#### 功能

```bash
#!/usr/bin/env bash
set -e

# 查找当前特性的规范文件
find_current_spec() {
    local feature_dir="$1"
    if [ -f "$feature_dir/spec.md" ]; then
        echo "$feature_dir/spec.md"
        return 0
    fi
    return 1
}

# 创建必要的子目录
setup_plan_directories() {
    local feature_dir="$1"
    mkdir -p "$feature_dir/contracts"
    mkdir -p "$feature_dir/checklists"
}

# 输出 JSON 给 AI
output_json() {
    local spec_file="$1"
    local feature_dir="$2"
    
    cat << EOF
{
  "status": "ready",
  "spec_file": "$spec_file",
  "feature_dir": "$feature_dir",
  "contracts_dir": "$feature_dir/contracts",
  "checklists_dir": "$feature_dir/checklists"
}
EOF
}
```

### 3. update-agent-context.sh

这个脚本更新 AI 代理的上下文文件。

#### 工作原理

```bash
#!/usr/bin/env bash
set -e

# 检测当前使用的 AI 代理
detect_agent_type() {
    if [ -f ".claude/commands/specify-rules.md" ]; then
        echo "claude"
    elif [ -f ".gemini/commands/specify-rules.md" ]; then
        echo "gemini"
    elif [ -f ".windsurf/rules/specify-rules.md" ]; then
        echo "windsurf"
    else
        echo ""
    fi
}

# 更新代理上下文文件
update_agent_file() {
    local agent_file="$1"
    local agent_name="$2"
    local technologies="$3"
    
    if [ ! -f "$agent_file" ]; then
        return 0
    fi
    
    # 在标记之间添加新技术
    # <!-- SPECIFY_START -->
    # <!-- SPECIFY_END -->
    
    local temp_file="${agent_file}.tmp"
    cp "$agent_file" "$temp_file"
    
    # 插入新技术信息
    sed -i "/<!-- SPECIFY_START -->/a\\
## New Technologies\\
\\
- ${technologies}\\
" "$temp_file"
    
    mv "$temp_file" "$agent_file"
}

# 主流程
main() {
    local agent_type="$1"
    local technologies="$2"
    
    if [ -z "$agent_type" ]; then
        agent_type=$(detect_agent_type)
    fi
    
    case "$agent_type" in
        claude)
            update_agent_file ".claude/commands/specify-rules.md" "Claude" "$technologies"
            ;;
        gemini)
            update_agent_file ".gemini/commands/specify-rules.md" "Gemini" "$technologies"
            ;;
        windsurf)
            update_agent_file ".windsurf/rules/specify-rules.md" "Windsurf" "$technologies"
            ;;
        *)
            # 如果没有明确指定，更新所有存在的代理
            for file in .claude/commands/specify-rules.md \
                       .gemini/commands/specify-rules.md \
                       .windsurf/rules/specify-rules.md; do
                if [ -f "$file" ]; then
                    update_agent_file "$file" "$(basename $(dirname $(dirname $file)))" "$technologies"
                fi
            done
            ;;
    esac
}
```

**关键特性**：

1. **自动检测**：根据目录结构检测使用的 AI 代理
2. **标记插入**：在 `<!-- SPECIFY_START -->` 和 `<!-- SPECIFY_END -->` 之间插入内容
3. **手动保护**：保留标记之间的手动添加内容
4. **多代理支持**：可以同时更新多个代理的上下文

---

## 🔧 公共函数库

### common.sh

```bash
#!/usr/bin/env bash

# 颜色输出
log_info() {
    echo "[INFO] $*"
}

log_error() {
    echo "[ERROR] $*" >&2
}

log_success() {
    echo "[SUCCESS] $*"
}

# JSON 输出
json_output() {
    local key="$1"
    local value="$2"
    echo "\"$key\": \"$value\""
}

# 检查命令是否存在
command_exists() {
    command -v "$1" >/dev/null 2>&1
}

# 获取相对路径
relative_path() {
    local target="$1"
    local base="${2:-$(pwd)}"
    
    python3 -c "import os, sys; print(os.path.relpath('$target', '$base'))" 2>/dev/null || \
    perl -MFile::Spec -e "print File::Spec->abs2rel('$target', '$base')"
}
```

### common.ps1

```powershell
# 颜色输出
function Log-Info {
    param([string]$Message)
    Write-Host "[INFO] $Message" -ForegroundColor Cyan
}

function Log-Error {
    param([string]$Message)
    Write-Host "[ERROR] $Message" -ForegroundColor Red
}

function Log-Success {
    param([string]$Message)
    Write-Host "[SUCCESS] $Message" -ForegroundColor Green
}

# JSON 输出
function ConvertTo-JsonOutput {
    param(
        [string]$Key,
        [string]$Value
    )
    "`"$Key`": `"$Value`""
}

# 检查命令是否存在
function Test-CommandExists {
    param([string]$Command)
    $null = Get-Command $Command -ErrorAction SilentlyContinue
    return $?
}

# 获取相对路径
function Get-RelativePath {
    param(
        [string]$TargetPath,
        [string]$BasePath = (Get-Location).Path
    )
    Resolve-Path -Relative $TargetPath -RelativeBasePath $BasePath
}
```

---

## 📊 Bash vs PowerShell 对比

### 参数处理

| 功能 | Bash | PowerShell |
|------|-------|-----------|
| 位置参数 | `$1`, `$2`, ... | `$args[0]`, `$args[1]`, ... |
| 命名参数 | 手动解析 | `param([string]$ParamName)` |
| 布尔开关 | `--flag` case 判断 | `[switch]$Flag` |
| 帮助 | `--help`, `-h` | `[CmdletBinding()]` |

### 文件操作

| 功能 | Bash | PowerShell |
|------|-------|-----------|
| 检查存在 | `[ -f "$path" ]` | `Test-Path $path` |
| 创建目录 | `mkdir -p` | `New-Item -ItemType Directory -Force` |
| 复制文件 | `cp` | `Copy-Item` |
| 移动文件 | `mv` | `Move-Item` |
| 删除文件 | `rm` | `Remove-Item` |

### JSON 处理

| 功能 | Bash | PowerShell |
|------|-------|-----------|
| 生成 JSON | `cat << EOF` | `ConvertTo-Json` |
| 解析 JSON | `jq` (外部工具) | `ConvertFrom-Json` |
| 字符串转义 | 手动转义 | 自动转义 |

---

## 🎯 脚本设计模式

### 1. 错误处理

```bash
#!/usr/bin/env bash
set -e  # 遇到错误立即退出
set -u  # 使用未定义变量时退出
set -o pipefail  # 管道中任何命令失败都退出

trap 'log_error "Script failed at line $LINENO"; exit 1' ERR
```

```powershell
$ErrorActionPreference = "Stop"
trap {
    Log-Error "Script failed: $_"
    exit 1
}
```

### 2. 参数验证

```bash
# 检查必需参数
if [ -z "$FEATURE_DESCRIPTION" ]; then
    log_error "Feature description is required"
    exit 1
fi

# 检查目录存在
if [ ! -d "$REPO_ROOT" ]; then
    log_error "Repository root not found: $REPO_ROOT"
    exit 1
fi
```

```powershell
# 检查必需参数
if (-not $FeatureDescription) {
    Log-Error "Feature description is required"
    exit 1
}

# 检查目录存在
if (-not (Test-Path $RepoRoot)) {
    Log-Error "Repository root not found: $RepoRoot"
    exit 1
}
```

### 3. JSON 输出

```bash
# 输出单个键值对
json_output "status" "success"

# 输出多个键值对
cat << EOF
{
  "status": "success",
  "branch": "$BRANCH_NAME",
  "spec_file": "$SPEC_FILE"
}
EOF
```

```powershell
# 输出单个键值对
ConvertTo-JsonOutput -Key "status" -Value "success"

# 输出多个键值对
@{
    status = "success"
    branch = $BranchName
    spec_file = $SpecFile
} | ConvertTo-Json
```

### 4. 平台特定逻辑

```bash
# Unix-like 特定逻辑
if [ "$OS" != "Windows_NT" ]; then
    chmod +x scripts/*.sh
fi
```

```powershell
# Windows 特定逻辑
if ($IsWindows) {
    # Windows 特定逻辑
}
```

---

## 🔌 脚本扩展

### 添加新脚本

1. **创建 Bash 版本**：

```bash
#!/usr/bin/env bash
set -e

# 加载公共函数
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/common.sh"

# 主逻辑
main() {
    log_info "Executing my custom script"
    # ... 脚本逻辑
}

main "$@"
```

2. **创建 PowerShell 版本**：

```powershell
#!/usr/bin/env pwsh
$ErrorActionPreference = "Stop"

# 加载公共函数
$ScriptDir = Split-Path -Parent $MyInvocation.MyCommand.Path
. "$ScriptDir\common.ps1"

# 主逻辑
function Main {
    Log-Info "Executing my custom script"
    # ... 脚本逻辑
}

Main $args
```

3. **在命令模板中引用**：

```markdown
---
scripts:
  sh: scripts/bash/my-custom-script.sh
  ps: scripts/powershell/my-custom-script.ps1
---
```

### 自定义公共函数

在 `common.sh` 和 `common.ps1` 中添加：

```bash
# common.sh
my_custom_function() {
    local input="$1"
    # ... 函数逻辑
    echo "$result"
}
```

```powershell
# common.ps1
function Invoke-MyCustomFunction {
    param([string]$Input)
    # ... 函数逻辑
    return $result
}
```

---

## 📈 性能优化

### 1. 并行操作

```bash
# 并行运行多个 git 操作
git fetch --all &
git prune &
wait

# 并行创建目录
mkdir -p "$FEATURE_DIR/contracts" &
mkdir -p "$FEATURE_DIR/checklists" &
wait
```

### 2. 缓存

```bash
# 缓存 Git 仓库根目录
if [ -z "$REPO_ROOT_CACHED" ]; then
    REPO_ROOT_CACHED=$(find_repo_root "$SCRIPT_DIR")
fi
```

### 3. 最小化文件操作

```bash
# 批量写入而非多次写入
cat << 'EOF' > "$OUTPUT_FILE"
content
more content
even more content
EOF
```

---

## 🎓 脚本系统学习要点

1. **双脚本策略**：Bash 和 PowerShell 并行实现
2. **统一接口**：相同参数和 JSON 输出格式
3. **错误处理**：set -e 和 trap
4. **公共函数**：共享逻辑减少重复
5. **智能编号**：从多源计算最大特性号
6. **智能命名**：过滤停止词，生成有意义分支名
7. **平台检测**：自动选择合适的脚本版本
8. **JSON 输出**：结构化输出便于 AI 解析
9. **长度限制**：处理 GitHub 分支长度限制
10. **上下文更新**：自动更新 AI 代理上下文

下一节将深入 AI 代理集成机制。