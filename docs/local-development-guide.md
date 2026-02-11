# Spec Kit 本地开发指南

本文档介绍如何在本地环境中安装、开发和测试 Spec Kit。

## 📋 前提条件

- **Python 3.11+**: [下载](https://www.python.org/downloads/)
- **Git**: [下载](https://git-scm.com/downloads/)
- **pip**: Python 自带的包管理器

## 🚀 快速开始

### 1. 克隆或获取代码

如果您已经有 Spec Kit 代码（当前目录），可以直接开始。如果需要克隆：

```bash
git clone https://github.com/github/spec-kit.git
cd spec-kit
```

### 2. 创建虚拟环境（推荐）

```bash
# Windows
python -m venv .venv
.venv\Scripts\activate

# macOS/Linux
python3 -m venv .venv
source .venv/bin/activate
```

### 3. 安装 Spec Kit（可编辑模式）

```bash
pip install -e .
```

**什么是可编辑模式（-e / --editable）？**

- `-e` 标志表示"可编辑"安装
- 代码修改后立即生效，无需重新安装
- 适合本地开发和调试
- 包实际上是从源代码目录引用，而不是复制到 site-packages

### 4. 验证安装

```bash
# 检查命令是否可用
specify --help

# 检查系统依赖
specify check

# 查看版本
specify --version
```

## 🔧 开发工作流

### 日常开发流程

```bash
# 1. 激活虚拟环境（如果未激活）
# Windows: .venv\Scripts\activate
# macOS/Linux: source .venv/bin/activate

# 2. 修改代码（编辑 src/specify_cli/ 中的文件）

# 3. 直接测试，无需重新安装
# 因为使用了可编辑模式，代码更改会立即生效
specify init test-project --ai claude

# 4. 如需安装额外的开发依赖
pip install <package_name>

# 5. 重新安装（仅当需要时）
pip install -e . --force-reinstall --no-deps
```

### 安装开发依赖

如果项目有开发依赖（在 pyproject.toml 的 dev 部分）：

```bash
# 查看 pyproject.toml 中的依赖
cat pyproject.toml

# 安装额外的开发工具
pip install pytest black ruff
```

## 🧪 测试本地修改

### 创建测试项目

```bash
# 创建一个测试项目来验证您的修改
specify init test-spec-kit --ai claude

# 进入测试项目
cd test-spec-kit

# 测试 /speckit.* 命令
# 在 AI 助手中运行：
/speckit.constitution Test constitution
```

### 调试技巧

#### 1. 添加调试输出

在 `src/specify_cli/__init__.py` 中添加调试语句：

```python
def main():
    print("DEBUG: Starting specify command...")
    # ... 原有代码
```

#### 2. 使用 Python 直接运行

```bash
# 直接运行 Python 模块
python -m specify_cli --help

# 这样可以看到 Python 错误堆栈
python -m specify_cli init test-project
```

#### 3. 检查代码是否生效

```python
# 在 __init__.py 的 main 函数中添加
import sys
print(f"DEBUG: Working directory: {sys.path[0]}")
```

## 📦 项目结构

```
spec-kit/
├── src/
│   └── specify_cli/
│       └── __init__.py    # 主要代码文件
├── templates/             # 模板文件
├── scripts/               # 脚本文件
├── docs/                 # 文档
├── pyproject.toml         # 项目配置
├── README.md             # 项目说明
└── .venv/               # 虚拟环境（不提交到 Git）
```

## 🔄 更新到最新版本

如果您想从本地切换回官方最新版本：

```bash
# 卸载本地版本
pip uninstall specify-cli

# 安装官方版本
pip install git+https://github.com/github/spec-kit.git
```

或者使用 uv（如果您配置了）：

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
```

## 🐛 常见问题

### 问题 1: 找不到 specify 命令

**症状**：
```
'specify' is not recognized as an internal or external command
```

**解决方案**：
```bash
# 1. 确认虚拟环境已激活
# Windows: echo $env:VIRTUAL_ENV
# macOS/Linux: echo $VIRTUAL_ENV

# 2. 检查 Scripts 或 bin 目录是否在 PATH 中

# 3. 重新安装
pip install -e . --force-reinstall
```

### 问题 2: 代码修改不生效

**症状**：修改代码后，运行命令时行为没有变化

**解决方案**：
```bash
# 1. 确认使用了 -e 标志安装
pip show specify-cli

# 查看 Location: 是否指向源代码目录，而不是 site-packages

# 2. 如果不是，重新安装
pip uninstall specify-cli
pip install -e .

# 3. 清除 Python 缓存
find . -type d -name __pycache__ -exec rm -rf {} +  # macOS/Linux
# Windows: 手动删除 __pycache__ 文件夹
```

### 问题 3: 导入错误

**症状**：
```
ModuleNotFoundError: No module named 'typer'
```

**解决方案**：
```bash
# 安装所有依赖
pip install -e .

# 或单独安装
pip install typer rich httpx platformdirs readchar truststore
```

### 问题 4: 权限错误（Windows）

**症状**：
```
PermissionError: [Errno 13] Permission denied
```

**解决方案**：
```bash
# 以管理员身份运行 PowerShell
# 或
# 使用用户安装模式
pip install -e . --user
```

## 📝 开发最佳实践

1. **使用虚拟环境**：避免污染系统 Python 环境
2. **可编辑模式**：使用 `pip install -e .` 进行开发
3. **小步测试**：每次修改后立即测试
4. **保持更新**：定期从主分支拉取最新代码
5. **版本管理**：修改代码前，建议创建新分支

## 🔄 从开发到生产

当您准备好发布更改时：

1. 更新 `pyproject.toml` 中的版本号
2. 更新 `CHANGELOG.md`
3. 提交并创建 Pull Request
4. 等待代码审查和合并

## 📚 相关资源

- [Spec Kit README](../README.md) - 项目概述
- [安装文档](installation.md) - 官方安装指南
- [快速开始](quickstart.md) - 使用 Spec Kit 的快速入门
- [升级指南](upgrade.md) - 升级到新版本

## 💡 贡献

欢迎贡献！请阅读 [CONTRIBUTING.md](../CONTRIBUTING.md) 了解贡献流程。

---

**文档版本**: 1.0  
**最后更新**: 2026-02-11  
**作者**: Spec Kit Team