# 02 - CLI 核心实现深度解析

## 📋 概述

Specify CLI 是 Spec Kit 的核心工具，使用 Python 的 Typer 框架构建。所有实现集中在 `src/specify_cli/__init__.py` 单个文件中，约 1000+ 行代码。

### 技术栈

- **语言**: Python 3.11+
- **CLI 框架**: Typer (基于 Click)
- **终端 UI**: Rich (美化输出、进度条、表格）
- **HTTP 客户端**: httpx (带 socks 支持）
- **跨平台输入**: readchar (键盘输入）
- **SSL/TLS**: truststore (系统证书存储）
- **路径操作**: pathlib (Python 标准库）

---

## 🔧 核心配置与常量

### 1. 依赖声明（Script metadata）

```python
#!/usr/bin/env python3
# /// script
# requires-python = ">=3.11"
# dependencies = [
#     "typer",
#     "rich",
#     "platformdirs",
#     "readchar",
#     "httpx",
# ]
# ///
```

这使用了 Python 3.11+ 的 PEP 723 脚本元数据格式，允许 `uv` 直接运行脚本而无需安装。

### 2. AI 代理配置（AGENT_CONFIG）

这是整个系统的**单真源**，定义了所有支持的 AI 代理：

```python
AGENT_CONFIG = {
    "copilot": {
        "name": "GitHub Copilot",
        "folder": ".github/",
        "install_url": None,  # IDE-based, no CLI check needed
        "requires_cli": False,
    },
    "claude": {
        "name": "Claude Code",
        "folder": ".claude/",
        "install_url": "https://docs.anthropic.com/en/docs/claude-code/setup",
        "requires_cli": True,
    },
    # ... 其他 17 个代理
}
```

**关键设计原则**：
- **字典键是实际的 CLI 工具名**（不是缩写）
- 例如使用 `"cursor-agent"` 而不是 `"cursor"`
- 这消除了特殊映射的需要

### 3. 脚本类型选择

```python
SCRIPT_TYPE_CHOICES = {"sh": "POSIX Shell (bash/zsh)", "ps": "PowerShell"}
```

跨平台支持：Bash 用于 Linux/macOS，PowerShell 用于 Windows。

### 4. Claude 特殊路径

```python
CLAUDE_LOCAL_PATH = Path.home() / ".claude" / "local" / "claude"
```

特殊处理 Claude CLI 的 `migrate-installer` 命令问题（issue #123）。

---

## 🎨 核心类与函数

### 1. StepTracker 类

进度追踪器，用于显示分层步骤：

```python
class StepTracker:
    """Track and render hierarchical steps without emojis, similar to Claude Code tree output.
    Supports live auto-refresh via an attached refresh callback.
    """
    def __init__(self, title: str):
        self.title = title
        self.steps = []  # list of dicts: {key, label, status, detail}
        self.status_order = {"pending": 0, "running": 1, "done": 2, "error": 3, "skipped": 4}
        self._refresh_cb = None  # callable to trigger UI refresh
```

**核心方法**：

| 方法 | 功能 |
|------|------|
| `add(key, label)` | 添加新步骤 |
| `start(key, detail)` | 开始执行步骤 |
| `complete(key, detail)` | 完成步骤 |
| `error(key, detail)` | 标记错误 |
| `skip(key, detail)` | 跳过步骤 |
| `render()` | 渲染为 Rich 树形图 |

**使用示例**：

```python
tracker = StepTracker("Initialize Specify Project")
tracker.add("precheck", "Check required tools")
tracker.start("precheck", "checking...")
tracker.complete("precheck", "ok")
console.print(tracker.render())
```

**输出效果**：

```
Initialize Specify Project
○ Check required tools (ok)
● Download template
○ Extract template
```

### 2. BannerGroup 类

自定义 Typer 组，在帮助信息前显示横幅：

```python
class BannerGroup(TyperGroup):
    """Custom group that shows banner before help."""
    
    def format_help(self, ctx, formatter):
        # Show banner before help
        show_banner()
        super().format_help(ctx, formatter)
```

### 3. show_banner() 函数

显示 ASCII 艺术横幅：

```python
BANNER = """
███████╗██████╗ ███████╗ ██████╗██╗███████╗██╗   ██╗
██╔════╝██╔══██╗██╔════╝██╔════╝██║██╔════╝╚██╗ ██╔╝
███████╗██████╔╝█████╗  ██║     ██║█████╗   ╚████╔╝ 
╚════██║██╔═══╝ ██╔══╝  ██║     ██║██╔══╝    ╚██╔╝  
███████║██║     ███████╗╚██████╗██║██║        ██║   
╚══════╝╚═╝     ╚══════╝ ╚═════╝╚═╝╚═╝        ╚═╝   
"""

def show_banner():
    """Display the ASCII art banner."""
    banner_lines = BANNER.strip().split('\n')
    colors = ["bright_blue", "blue", "cyan", "bright_cyan", "white", "bright_white"]
    
    styled_banner = Text()
    for i, line in enumerate(banner_lines):
        color = colors[i % len(colors)]
        styled_banner.append(line + "\n", style=color)
    
    console.print(Align.center(styled_banner))
    console.print(Align.center(Text(TAGLINE, style="italic bright_yellow")))
    console.print()
```

---

## 🌐 GitHub API 集成

### 1. SSL 上下文配置

```python
import ssl
import truststore
from httpx import Client

ssl_context = truststore.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
client = httpx.Client(verify=ssl_context)
```

使用 `truststore` 确保使用系统证书存储，避免 SSL 问题。

### 2. GitHub Token 处理

```python
def _github_token(cli_token: str | None = None) -> str | None:
    """Return sanitized GitHub token (cli arg takes precedence) or None."""
    return ((cli_token or os.getenv("GH_TOKEN") or os.getenv("GITHUB_TOKEN") or "").strip()) or None

def _github_auth_headers(cli_token: str | None = None) -> dict:
    """Return Authorization header dict only when a non-empty token exists."""
    token = _github_token(cli_token)
    return {"Authorization": f"Bearer {token}"} if token else {}
```

**优先级**：CLI 参数 > GH_TOKEN 环境变量 > GITHUB_TOKEN 环境变量

### 3. 速率限制解析

```python
def _parse_rate_limit_headers(headers: httpx.Headers) -> dict:
    """Extract and parse GitHub rate-limit headers."""
    info = {}
    
    # Standard GitHub rate-limit headers
    if "X-RateLimit-Limit" in headers:
        info["limit"] = headers.get("X-RateLimit-Limit")
    if "X-RateLimit-Remaining" in headers:
        info["remaining"] = headers.get("X-RateLimit-Remaining")
    if "X-RateLimit-Reset" in headers:
        reset_epoch = int(headers.get("X-RateLimit-Reset", "0"))
        if reset_epoch:
            reset_time = datetime.fromtimestamp(reset_epoch, tz=timezone.utc)
            info["reset_epoch"] = reset_time
            info["reset_time"] = reset_time
            info["reset_local"] = reset_time.astimezone()
    
    return info
```

### 4. 速率限制错误格式化

```python
def _format_rate_limit_error(status_code: int, headers: httpx.Headers, url: str) -> str:
    """Format a user-friendly error message with rate-limit information."""
    rate_info = _parse_rate_limit_headers(headers)
    
    lines = [f"GitHub API returned status {status_code} for {url}"]
    lines.append("")
    
    if rate_info:
        lines.append("[bold]Rate Limit Information:[/bold]")
        if "limit" in rate_info:
            lines.append(f"  • Rate Limit: {rate_info['limit']} requests/hour")
        if "remaining" in rate_info:
            lines.append(f"  • Remaining: {rate_info['remaining']}")
        if "reset_local" in rate_info:
            reset_str = rate_info["reset_local"].strftime("%Y-%m-%d %H:%M:%S %Z")
            lines.append(f"  • Resets at: {reset_str}")
    
    # Add troubleshooting guidance
    lines.append("[bold]Troubleshooting Tips:[/bold]")
    lines.append("  • If you're on a shared CI or corporate environment, you may be rate-limited.")
    lines.append("  • Consider using a GitHub token via --github-token or the GH_TOKEN/GITHUB_TOKEN")
    lines.append("    environment variable to increase rate limits.")
    lines.append("  • Authenticated requests have a limit of 5,000/hour vs 60/hour for unauthenticated.")
    
    return "\n".join(lines)
```

---

## 📥 模板下载与解压

### 1. download_template_from_github()

核心下载函数：

```python
def download_template_from_github(
    ai_assistant: str, 
    download_dir: Path, 
    *, 
    script_type: str = "sh", 
    verbose: bool = True, 
    show_progress: bool = True, 
    client: httpx.Client = None, 
    debug: bool = False, 
    github_token: str = None
) -> Tuple[Path, dict]:
    """Download the latest release template from GitHub."""
```

**工作流程**：

```
1. 构建 API URL
   ↓
2. 获取最新 release 信息
   ↓
3. 匹配对应的 ZIP 包
   (spec-kit-template-{ai_assistant}-{script_type}.zip)
   ↓
4. 下载文件（带进度条）
   ↓
5. 返回文件路径和元数据
```

**关键代码片段**：

```python
# 构建 API URL
api_url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/releases/latest"

# 发送请求
response = client.get(
    api_url,
    timeout=30,
    follow_redirects=True,
    headers=_github_auth_headers(github_token),
)

# 解析 release 数据
release_data = response.json()
assets = release_data.get("assets", [])

# 匹配对应的模板
pattern = f"spec-kit-template-{ai_assistant}-{script_type}"
matching_assets = [
    asset for asset in assets
    if pattern in asset["name"] and asset["name"].endswith(".zip")
]
```

### 2. download_and_extract_template()

下载并解压模板到项目目录：

```python
def download_and_extract_template(
    project_path: Path, 
    ai_assistant: str, 
    script_type: str, 
    is_current_dir: bool = False, 
    *, 
    verbose: bool = True, 
    tracker: StepTracker | None = None,
    client: httpx.Client = None, 
    debug: bool = False, 
    github_token: str = None
) -> Path:
    """Download and extract the template."""
```

**关键特性**：

1. **当前目录合并模式** (`is_current_dir=True`):
   - 临时解压到 temp 目录
   - 合并文件到当前目录
   - 特殊处理 `.vscode/settings.json`（合并而非覆盖）

2. **新建目录模式** (`is_current_dir=False`):
   - 直接解压到新目录
   - 处理嵌套目录结构

3. **VS Code 设置智能合并**：

```python
def handle_vscode_settings(sub_item, dest_file, rel_path, verbose=False, tracker=None):
    """Handle merging or copying of .vscode/settings.json files."""
    try:
        with open(sub_item, 'r', encoding='utf-8') as f:
            new_settings = json.load(f)
        
        if dest_file.exists():
            merged = merge_json_files(dest_file, new_settings, verbose=verbose)
            with open(dest_file, 'w', encoding='utf-8') as f:
                json.dump(merged, f, indent=4)
                f.write('\n')
        else:
            shutil.copy2(sub_item, dest_file)
    except Exception as e:
        shutil.copy2(sub_item, dest_file)
```

---

## 🔧 JSON 深度合并算法

### merge_json_files()

递归合并两个 JSON 字典：

```python
def merge_json_files(existing_path: Path, new_content: dict, verbose: bool = False) -> dict:
    """Merge new JSON content into existing JSON file.
    
    Performs a deep merge where:
    - New keys are added
    - Existing keys are preserved unless overwritten by new content
    - Nested dictionaries are merged recursively
    - Lists and other values are replaced (not merged)
    """
    try:
        with open(existing_path, 'r', encoding='utf-8') as f:
            existing_content = json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        # If file doesn't exist or is invalid, just use new content
        return new_content
    
    def deep_merge(base: dict, update: dict) -> dict:
        """Recursively merge update dict into base dict."""
        result = base.copy()
        for key, value in update.items():
            if key in result and isinstance(result[key], dict) and isinstance(value, dict):
                # Recursively merge nested dictionaries
                result[key] = deep_merge(result[key], value)
            else:
                # Add new key or replace existing value
                result[key] = value
        return result
    
    merged = deep_merge(existing_content, new_content)
    return merged
```

**合并规则**：

| 情况 | 行为 |
|------|------|
| 新键 | 添加到结果 |
| 嵌套字典 | 递归合并 |
| 非字典值 | 新值覆盖旧值 |
| 列表 | 新列表替换旧列表 |

**示例**：

```python
existing = {
    "a": 1,
    "b": {"x": 10, "y": 20},
    "c": [1, 2, 3]
}
new = {
    "a": 100,
    "b": {"y": 200, "z": 30},
    "d": "new"
}

# 结果
merged = {
    "a": 100,           # 覆盖
    "b": {"x": 10, "y": 200, "z": 30},  # 合并
    "c": [1, 2, 3],       # 保留（列表不合并）
    "d": "new"            # 新增
}
```

---

## 🔍 Git 集成

### 1. is_git_repo()

检查路径是否在 Git 仓库中：

```python
def is_git_repo(path: Path = None) -> bool:
    """Check if the specified path is inside a git repository."""
    if path is None:
        path = Path.cwd()
    
    if not path.is_dir():
        return False
    
    try:
        subprocess.run(
            ["git", "rev-parse", "--is-inside-work-tree"],
            check=True,
            capture_output=True,
            cwd=path,
        )
        return True
    except (subprocess.CalledProcessError, FileNotFoundError):
        return False
```

### 2. init_git_repo()

初始化 Git 仓库：

```python
def init_git_repo(project_path: Path, quiet: bool = False) -> Tuple[bool, Optional[str]]:
    """Initialize a git repository in the specified path.
    
    Returns:
        Tuple of (success: bool, error_message: Optional[str])
    """
    try:
        original_cwd = Path.cwd()
        os.chdir(project_path)
        if not quiet:
            console.print("[cyan]Initializing git repository...[/cyan]")
        subprocess.run(["git", "init"], check=True, capture_output=True, text=True)
        subprocess.run(["git", "add", "."], check=True, capture_output=True, text=True)
        subprocess.run(["git", "commit", "-m", "Initial commit from Specify template"], 
                      check=True, capture_output=True, text=True)
        if not quiet:
            console.print("[green]✓[/green] Git repository initialized")
        return True, None
    except subprocess.CalledProcessError as e:
        error_msg = f"Command: {' '.join(e.cmd)}\nExit code: {e.returncode}"
        if e.stderr:
            error_msg += f"\nError: {e.stderr.strip()}"
        return False, error_msg
    finally:
        os.chdir(original_cwd)
```

**执行步骤**：
1. `git init` - 初始化仓库
2. `git add .` - 添加所有文件
3. `git commit` - 创建初始提交

---

## 🎛️ 交互式 UI 组件

### 1. get_key()

跨平台键盘输入：

```python
import readchar

def get_key():
    """Get a single keypress in a cross-platform way using readchar."""
    key = readchar.readkey()
    
    if key == readchar.key.UP or key == readchar.key.CTRL_P:
        return 'up'
    if key == readchar.key.DOWN or key == readchar.key.CTRL_N:
        return 'down'
    if key == readchar.key.ENTER:
        return 'enter'
    if key == readchar.key.ESC:
        return 'escape'
    if key == readchar.key.CTRL_C:
        raise KeyboardInterrupt
    
    return key
```

### 2. select_with_arrows()

交互式选择菜单（使用 Rich Live）：

```python
def select_with_arrows(
    options: dict, 
    prompt_text: str = "Select an option", 
    default_key: str = None
) -> str:
    """
    Interactive selection using arrow keys with Rich Live display.
    
    Args:
        options: Dict with keys as option keys and values as descriptions
        prompt_text: Text to show above the options
        default_key: Default option key to start with
        
    Returns:
        Selected option key
    """
```

**功能特性**：
- ↑/↓ 键导航
- Enter 选择
- Esc 取消
- 支持 Ctrl+C 中断
- 实时刷新显示

**使用示例**：

```python
ai_choices = {
    "claude": "Claude Code",
    "gemini": "Gemini CLI",
    "copilot": "GitHub Copilot"
}

selected_ai = select_with_arrows(
    ai_choices, 
    "Choose your AI assistant:", 
    "copilot"  # 默认选择
)
```

---

## 🔧 工具函数

### 1. run_command()

执行 shell 命令：

```python
def run_command(
    cmd: list[str], 
    check_return: bool = True, 
    capture: bool = False, 
    shell: bool = False
) -> Optional[str]:
    """Run a shell command and optionally capture output."""
    try:
        if capture:
            result = subprocess.run(cmd, check=check_return, capture_output=True, text=True, shell=shell)
            return result.stdout.strip()
        else:
            subprocess.run(cmd, check=check_return, shell=shell)
            return None
    except subprocess.CalledProcessError as e:
        if check_return:
            console.print(f"[red]Error running command:[/red] {' '.join(cmd)}")
            console.print(f"[red]Exit code:[/red] {e.returncode}")
            raise
        return None
```

### 2. check_tool()

检查工具是否安装：

```python
def check_tool(tool: str, tracker: StepTracker = None) -> bool:
    """Check if a tool is installed. Optionally update tracker."""
    # Special handling for Claude CLI after `claude migrate-installer`
    if tool == "claude":
        if CLAUDE_LOCAL_PATH.exists() and CLAUDE_LOCAL_PATH.is_file():
            if tracker:
                tracker.complete(tool, "available")
            return True
    
    found = shutil.which(tool) is not None
    
    if tracker:
        if found:
            tracker.complete(tool, "available")
        else:
            tracker.error(tool, "not found")
    
    return found
```

### 3. ensure_executable_scripts()

设置脚本执行权限（仅 Unix-like 系统）：

```python
def ensure_executable_scripts(project_path: Path, tracker: StepTracker | None = None) -> None:
    """Ensure POSIX .sh scripts under .specify/scripts (recursively) have execute bits."""
    if os.name == "nt":
        return  # Windows: skip silently
    
    scripts_root = project_path / ".specify" / "scripts"
    if not scripts_root.is_dir():
        return
    
    failures = []
    updated = 0
    
    for script in scripts_root.rglob("*.sh"):
        try:
            if script.is_symlink() or not script.is_file():
                continue
            
            # Check shebang
            with script.open("rb") as f:
                if f.read(2) != b"#!":
                    continue
            
            st = script.stat()
            mode = st.st_mode
            
            if mode & 0o111:
                continue  # Already executable
            
            new_mode = mode
            if mode & 0o400: new_mode |= 0o100
            if mode & 0o040: new_mode |= 0o010
            if mode & 0o004: new_mode |= 0o001
            if not (new_mode & 0o100):
                new_mode |= 0o100
            
            os.chmod(script, new_mode)
            updated += 1
        except Exception as e:
            failures.append(f"{script.relative_to(scripts_root)}: {e}")
    
    if tracker:
        detail = f"{updated} updated" + (f", {len(failures)} failed" if failures else "")
        tracker.add("chmod", "Set script permissions recursively")
        (tracker.error if failures else tracker.complete)("chmod", detail)
```

---

## 📦 主命令实现

### 1. init() 命令

初始化新项目的核心命令：

```python
@app.command()
def init(
    project_name: str = typer.Argument(None, help="Name for your new project directory"),
    ai_assistant: str = typer.Option(None, "--ai", help="AI assistant to use"),
    script_type: str = typer.Option(None, "--script", help="Script type to use: sh or ps"),
    ignore_agent_tools: bool = typer.Option(False, "--ignore-agent-tools"),
    no_git: bool = typer.Option(False, "--no-git"),
    here: bool = typer.Option(False, "--here"),
    force: bool = typer.Option(False, "--force"),
    skip_tls: bool = typer.Option(False, "--skip-tls"),
    debug: bool = typer.Option(False, "--debug"),
    github_token: str = typer.Option(None, "--github-token"),
):
```

**执行流程**：

```
1. 显示横幅
   ↓
2. 参数验证
   - 检查项目名和 --here 冲突
   - 检查目录是否存在
   ↓
3. 交互式选择
   - 选择 AI 代理（如果未指定）
   - 选择脚本类型（如果未指定）
   ↓
4. 工具检查
   - 检查 Git（如果需要）
   - 检查 AI 工具（如果需要）
   ↓
5. 初始化进度追踪
   ↓
6. 下载并解压模板
   ↓
7. 设置脚本权限
   ↓
8. 初始化 Git 仓库
   ↓
9. 显示后续步骤
```

**关键代码段**：

```python
# 交互式选择 AI 代理
if ai_assistant:
    if ai_assistant not in AGENT_CONFIG:
        console.print(f"[red]Error:[/red] Invalid AI assistant '{ai_assistant}'")
        raise typer.Exit(1)
    selected_ai = ai_assistant
else:
    ai_choices = {key: config["name"] for key, config in AGENT_CONFIG.items()}
    selected_ai = select_with_arrows(ai_choices, "Choose your AI assistant:", "copilot")

# 使用 Rich Live 显示进度
tracker = StepTracker("Initialize Specify Project")
with Live(tracker.render(), console=console, refresh_per_second=8, transient=True) as live:
    tracker.attach_refresh(lambda: live.update(tracker.render()))
    try:
        download_and_extract_template(
            project_path, selected_ai, selected_script, here,
            verbose=False, tracker=tracker, client=local_client,
            debug=debug, github_token=github_token
        )
        ensure_executable_scripts(project_path, tracker=tracker)
        ensure_constitution_from_template(project_path, tracker=tracker)
        
        if not no_git:
            tracker.start("git")
            if is_git_repo(project_path):
                tracker.complete("git", "existing repo detected")
            elif should_init_git:
                success, error_msg = init_git_repo(project_path, quiet=True)
                if success:
                    tracker.complete("git", "initialized")
                else:
                    tracker.error("git", "init failed")
            else:
                tracker.skip("git", "git not available")
        
        tracker.complete("final", "project ready")
    except Exception as e:
        tracker.error("final", str(e))
        raise
```

### 2. check() 命令

检查已安装的工具：

```python
@app.command()
def check():
    """Check that all required tools are installed."""
    show_banner()
    console.print("[bold]Checking for installed tools...[/bold]\n")
    
    tracker = StepTracker("Check Available Tools")
    
    # Check Git
    tracker.add("git", "Git version control")
    git_ok = check_tool("git", tracker=tracker)
    
    # Check all AI agents
    for agent_key, agent_config in AGENT_CONFIG.items():
        agent_name = agent_config["name"]
        requires_cli = agent_config["requires_cli"]
        
        tracker.add(agent_key, agent_name)
        
        if requires_cli:
            check_tool(agent_key, tracker=tracker)
        else:
            tracker.skip(agent_key, "IDE-based, no CLI check")
    
    # Check VS Code
    tracker.add("code", "Visual Studio Code")
    check_tool("code", tracker=tracker)
    
    tracker.add("code-insiders", "Visual Studio Code Insiders")
    check_tool("code-insiders", tracker=tracker)
    
    console.print(tracker.render())
```

### 3. version() 命令

显示版本和系统信息：

```python
@app.command()
def version():
    """Display version and system information."""
    import platform
    import importlib.metadata
    
    show_banner()
    
    # Get CLI version from package metadata
    cli_version = "unknown"
    try:
        cli_version = importlib.metadata.version("specify-cli")
    except Exception:
        pass
    
    # Fetch latest template release version
    api_url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/releases/latest"
    
    template_version = "unknown"
    try:
        response = client.get(api_url, timeout=10, follow_redirects=True)
        if response.status_code == 200:
            release_data = response.json()
            template_version = release_data.get("tag_name", "unknown")
            if template_version.startswith("v"):
                template_version = template_version[1:]
    except Exception:
        pass
    
    # Display information table
    info_table = Table(show_header=False, box=None, padding=(0, 2))
    info_table.add_column("Key", style="cyan", justify="right")
    info_table.add_column("Value", style="white")
    
    info_table.add_row("CLI Version", cli_version)
    info_table.add_row("Template Version", template_version)
    info_table.add_row("Python", platform.python_version())
    info_table.add_row("Platform", platform.system())
    
    console.print(Panel(info_table, title="[bold cyan]Specify CLI Information[/bold cyan]"))
```

---

## 🎯 关键设计模式

### 1. 单一职责原则

每个函数/类都有明确的单一职责：
- `StepTracker` - 追踪进度
- `download_template_from_github` - 下载模板
- `merge_json_files` - 合并 JSON
- `init_git_repo` - 初始化 Git

### 2. 错误处理模式

```python
try:
    # 操作
except subprocess.CalledProcessError as e:
    # 处理进程错误
except Exception as e:
    # 处理一般错误
finally:
    # 清理资源
```

### 3. 可选依赖模式

```python
if tracker:
    tracker.complete("task", "done")
```

允许函数在有或没有追踪器的情况下工作。

### 4. 跨平台抽象

```python
if os.name == "nt":
    # Windows 特定逻辑
else:
    # Unix-like 特定逻辑
```

### 5. 类型提示

广泛使用 Python 3.11+ 的类型提示：

```python
def download_template_from_github(
    ai_assistant: str,
    download_dir: Path,
    *,
    script_type: str = "sh",
    verbose: bool = True,
    client: httpx.Client = None,
    debug: bool = False,
    github_token: str = None
) -> Tuple[Path, dict]:
```

---

## 📊 性能优化

### 1. 流式下载

使用 `httpx.stream()` 进行大文件下载：

```python
with client.stream("GET", download_url, timeout=60) as response:
    with open(zip_path, 'wb') as f:
        for chunk in response.iter_bytes(chunk_size=8192):
            f.write(chunk)
```

### 2. 延迟渲染

Rich Live 只在需要时更新 UI：

```python
def _maybe_refresh(self):
    if self._refresh_cb:
        try:
            self._refresh_cb()
        except Exception:
            pass
```

### 3. 条件输出

仅在 verbose 模式下输出详细信息：

```python
if verbose:
    console.print(f"[cyan]Downloading template...[/cyan]")
```

---

## 🔐 安全考虑

### 1. SSL/TLS 验证

默认使用系统证书存储：

```python
ssl_context = truststore.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
```

可通过 `--skip-tls` 禁用（不推荐）。

### 2. Token 安全

Token 仅在需要时添加到请求头：

```python
def _github_auth_headers(cli_token: str | None = None) -> dict:
    token = _github_token(cli_token)
    return {"Authorization": f"Bearer {token}"} if token else {}
```

### 3. 路径验证

使用 `pathlib` 确保路径操作安全：

```python
project_path = Path(project_name).resolve()
```

---

## 🎓 学习要点

1. **配置驱动**：`AGENT_CONFIG` 作为单真源
2. **错误友好**：详细的错误消息和故障排除建议
3. **跨平台**：Bash/PowerShell 双脚本支持
4. **进度可视化**：Rich Live 实时更新
5. **类型安全**：完整的类型提示
6. **模块化**：清晰的函数职责划分
7. **Git 集成**：深度集成 Git 工作流
8. **智能合并**：JSON 深度合并算法
9. **交互友好**：键盘导航的交互式选择
10. **异步支持**：使用 httpx 进行高效网络请求

下一节将深入 SDD 工作流的实现原理。