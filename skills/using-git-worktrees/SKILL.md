---
name: using-git-worktrees
description: 在需要与当前工作区隔离的功能开发或执行实现计划之前使用 - 通过原生工具或 git worktree 回退方案确保隔离的工作区
---

# 使用 Git Worktree

## 概述

确保在隔离的工作区中进行开发。优先使用平台原生的 worktree 工具。仅在没有原生工具时回退到手动 git worktree。

**核心原则：** 先检测已有的隔离环境。再使用原生工具。最后回退到 git。永远不要与宿主环境对抗。

**开始时宣布：** "我正在使用 using-git-worktrees skill 来设置隔离的工作区。"

## 步骤 0：检测已有的隔离环境

**在创建任何东西之前，检查你是否已经在隔离的工作区中。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**子模块防护：** `GIT_DIR != GIT_COMMON` 在 git submodule 内也为真。在判断"已在 worktree 中"之前，先确认你不在子模块里：

```bash
# 如果返回了路径，说明你在子模块中，而非 worktree —— 按普通仓库处理
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是子模块）：** 你已经在一个链接的 worktree 中。跳到步骤 3（项目设置）。不要再创建 worktree。

报告分支状态：
- 在分支上："已在隔离工作区 `<path>`，分支 `<name>`。"
- 分离 HEAD："已在隔离工作区 `<path>`（分离 HEAD，外部管理）。完成时需要创建分支。"

**如果 `GIT_DIR == GIT_COMMON`（或在子模块中）：** 你在普通的仓库检出中。

用户是否已在指令中声明了 worktree 偏好？如果没有，在创建 worktree 前征求同意：

> "需要我设置一个隔离的 worktree 吗？它可以保护你当前的分支不受影响。"

如果已有声明的偏好，直接遵循，无需再问。如果用户拒绝，就在当前目录工作，跳到步骤 3。

## 步骤 1：创建隔离工作区

**你有两种机制。按以下顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

用户已同意创建隔离工作区（步骤 0 同意）。你是否已有创建 worktree 的方式？可能是名为 `EnterWorktree`、`WorktreeCreate` 的工具，`/worktree` 命令，或 `--worktree` 标志。如果有，使用它并跳到步骤 3。

原生工具会自动处理目录放置、分支创建和清理。当你有原生工具时使用 `git worktree add` 会创建宿主环境无法看到或管理的幽灵状态。

仅在没有原生 worktree 工具时才进入步骤 1b。

### 1b. Git Worktree 回退方案

**仅在步骤 1a 不适用时使用** —— 你没有原生 worktree 工具。手动使用 git 创建 worktree。

#### 目录选择

按以下优先级顺序。用户的明确偏好始终优先于观察到的文件系统状态。

1. **检查指令中是否声明了 worktree 目录偏好。** 如果用户已指定，直接使用，无需询问。

2. **检查项目本地是否已有 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏目录）
   ls -d worktrees 2>/dev/null      # 备选
   ```
   如果找到，使用它。如果两者都存在，`.worktrees` 优先。

3. **检查全局目录是否已存在：**
   ```bash
   project=$(basename "$(git rev-parse --show-toplevel)")
   ls -d ~/.config/zonedev/worktrees/$project 2>/dev/null
   ```
   如果找到，使用它（向后兼容旧的全局路径）。

4. **如果没有其他指引**，默认使用项目根目录下的 `.worktrees/`。

#### 安全验证（仅限项目本地目录）

**必须在创建 worktree 前验证目录已被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 添加到 .gitignore，提交更改，然后继续。

**为什么关键：** 防止将 worktree 内容意外提交到仓库。

全局目录（`~/.config/zonedev/worktrees/`）无需验证。

#### 创建 Worktree

```bash
project=$(basename "$(git rev-parse --show-toplevel)")

# 根据选定的位置确定路径
# 项目本地: path="$LOCATION/$BRANCH_NAME"
# 全局: path="~/.config/zonedev/worktrees/$project/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**沙箱回退：** 如果 `git worktree add` 因权限错误（沙箱拒绝）失败，告知用户沙箱阻止了 worktree 创建，你将在当前目录中工作。然后在当前位置运行设置和基线测试。

## 步骤 3：项目设置

自动检测并运行适当的设置：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## 步骤 4：验证干净的基线

运行测试确保工作区起始状态干净：

```bash
# 使用项目对应的命令
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，询问是否继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree 就绪于 <完整路径>
测试通过（<N> 个测试，0 个失败）
准备实现 <功能名称>
```

## 快速参考

| 情况 | 操作 |
|------|------|
| 已在链接的 worktree 中 | 跳过创建（步骤 0） |
| 在子模块中 | 按普通仓库处理（步骤 0 防护） |
| 有原生 worktree 工具 | 使用它（步骤 1a） |
| 没有原生工具 | Git worktree 回退方案（步骤 1b） |
| `.worktrees/` 已存在 | 使用它（验证已忽略） |
| `worktrees/` 已存在 | 使用它（验证已忽略） |
| 两者都存在 | 使用 `.worktrees/` |
| 都不存在 | 检查指令文件，然后默认 `.worktrees/` |
| 全局路径已存在 | 使用它（向后兼容） |
| 目录未被忽略 | 添加到 .gitignore + 提交 |
| 创建时权限错误 | 沙箱回退，在当前位置工作 |
| 基线测试失败 | 报告失败 + 询问 |
| 没有 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误

### 与宿主环境对抗

- **问题：** 当平台已提供隔离时还使用 `git worktree add`
- **修复：** 步骤 0 检测已有隔离。步骤 1a 让位于原生工具。

### 跳过检测

- **问题：** 在已有的 worktree 内创建嵌套的 worktree
- **修复：** 创建任何东西之前始终运行步骤 0

### 跳过忽略验证

- **问题：** Worktree 内容被跟踪，污染 git status
- **修复：** 创建项目本地 worktree 前始终使用 `git check-ignore`

### 假设目录位置

- **问题：** 造成不一致，违反项目约定
- **修复：** 遵循优先级：已存在 > 全局旧路径 > 指令文件 > 默认

### 在测试失败时继续

- **问题：** 无法区分新 bug 和已有问题
- **修复：** 报告失败，获得明确许可后再继续

## 危险信号

**永远不要：**
- 当步骤 0 检测到已有隔离时创建 worktree
- 当你有原生 worktree 工具（如 `EnterWorktree`）时使用 `git worktree add`。这是最常见的错误——如果你有它，就用它。
- 跳过步骤 1a 直接使用步骤 1b 的 git 命令
- 在未验证是否被忽略的情况下创建 worktree（项目本地）
- 跳过基线测试验证
- 在未询问的情况下带着失败的测试继续

**始终：**
- 先运行步骤 0 检测
- 优先使用原生工具而非 git 回退
- 遵循目录优先级：已存在 > 全局旧路径 > 指令文件 > 默认
- 对项目本地目录验证是否被忽略
- 自动检测并运行项目设置
- 验证测试基线干净
