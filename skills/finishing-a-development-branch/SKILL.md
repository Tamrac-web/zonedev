---
name: finishing-a-development-branch
description: 当实现完成、所有测试通过、需要决定如何整合工作时使用——通过展示合并、PR 或清理的结构化选项来引导开发工作的收尾
---

# Finishing a Development Branch

## 概述

通过展示清晰的选项并处理所选的工作流，引导开发工作的收尾。

**核心原则：** 验证测试 → 检测环境 → 展示选项 → 执行选择 → 清理。

**开始时宣告：** "我正在使用 finishing-a-development-branch skill 来完成这项工作。"

## 流程

### 步骤 1：验证测试

**在展示选项之前，验证测试通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
测试失败（<N> 个失败）。必须在完成前修复：

[显示失败信息]

在测试通过之前无法进行合并/PR。
```

停下。不要进入步骤 2。

**如果测试通过：** 继续步骤 2。

### 步骤 2：检测环境

**在展示选项之前，确定工作空间状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这决定了显示哪个菜单以及清理方式：

| 状态 | 菜单 | 清理 |
|------|------|------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无 worktree 需清理 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 4 选项 | 基于来源判断（见步骤 6） |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 精简 3 选项（无合并） | 无需清理（外部管理） |

### 步骤 3：确定基础分支

```bash
# 尝试常见的基础分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或者询问："此分支从 main 分出——对吗？"

### 步骤 4：展示选项

**普通仓库和命名分支 worktree——展示以下 4 个选项：**

```
实现完成。你想怎么处理？

1. 本地合并回 <base-branch>
2. 推送并创建 Pull Request
3. 保持分支现状（我稍后处理）
4. 丢弃这些工作

选择哪个？
```

**Detached HEAD——展示以下 3 个选项：**

```
实现完成。你当前在 detached HEAD 上（外部管理的工作空间）。

1. 作为新分支推送并创建 Pull Request
2. 保持现状（我稍后处理）
3. 丢弃这些工作

选择哪个？
```

**不要添加说明** ——保持选项简洁。

### 步骤 5：执行选择

#### 选项 1：本地合并

```bash
# 获取主仓库根目录以确保 CWD 安全
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并——确认成功后再移除任何内容
git checkout <base-branch>
git pull
git merge <feature-branch>

# 在合并结果上验证测试
<test command>

# 仅在合并成功后：清理 worktree（步骤 6），然后删除分支
```

然后：清理 worktree（步骤 6），然后删除分支：

```bash
git branch -d <feature-branch>
```

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>

# 创建 PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## 摘要
<2-3 条变更要点>

## 测试计划
- [ ] <验证步骤>
EOF
)"
```

**不要清理 worktree** ——用户需要它来响应 PR 反馈。

#### 选项 3：保持现状

汇报："保持分支 <name>。Worktree 保留在 <path>。"

**不要清理 worktree。**

#### 选项 4：丢弃

**先确认：**
```
这将永久删除：
- 分支 <name>
- 所有 commit：<commit-list>
- 位于 <path> 的 worktree

输入 'discard' 确认。
```

等待精确的确认。

如果已确认：
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理 worktree（步骤 6），然后强制删除分支：
```bash
git branch -D <feature-branch>
```

### 步骤 6：清理工作空间

**仅在选项 1 和选项 4 时运行。** 选项 2 和 3 始终保留 worktree。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通仓库，无 worktree 需清理。完成。

**如果 worktree 路径在 `.worktrees/`、`worktrees/` 或 `~/.config/zonedev/worktrees/` 下：** Zonedev 创建了此 worktree——我们负责清理。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自我修复：清理任何过期的注册
```

**否则：** 宿主环境（harness）拥有此工作空间。不要移除它。如果你的平台提供了退出工作空间的工具，使用它。否则，保持工作空间不变。

## 快速参考

| 选项 | 合并 | 推送 | 保留 Worktree | 清理分支 |
|------|------|------|---------------|----------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持现状 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## 常见错误

**跳过测试验证**
- **问题：** 合并了有问题的代码，创建了失败的 PR
- **解决：** 在提供选项之前始终验证测试

**开放式提问**
- **问题：** "接下来应该做什么？"太模糊
- **解决：** 展示精确的 4 个结构化选项（detached HEAD 时为 3 个）

**在选项 2 时清理 worktree**
- **问题：** 移除了用户在 PR 迭代中需要的 worktree
- **解决：** 仅在选项 1 和选项 4 时清理

**在移除 worktree 之前删除分支**
- **问题：** `git branch -d` 失败，因为 worktree 仍在引用该分支
- **解决：** 先合并，移除 worktree，然后删除分支

**在 worktree 内部运行 git worktree remove**
- **问题：** 当 CWD 在被移除的 worktree 内部时命令会静默失败
- **解决：** 始终先 `cd` 到主仓库根目录再执行 `git worktree remove`

**清理由 harness 创建的 worktree**
- **问题：** 移除 harness 创建的 worktree 会导致幽灵状态
- **解决：** 仅清理 `.worktrees/`、`worktrees/` 或 `~/.config/zonedev/worktrees/` 下的 worktree

**丢弃时无确认**
- **问题：** 意外删除工作成果
- **解决：** 要求输入 "discard" 进行确认

## 红线

**绝不：**
- 在测试失败时继续推进
- 未验证合并结果的测试就合并
- 未经确认就删除工作成果
- 未经明确请求就 force-push
- 在确认合并成功之前移除 worktree
- 清理你没有创建的 worktree（来源检查）
- 在 worktree 内部运行 `git worktree remove`

**始终：**
- 在提供选项之前验证测试
- 在展示菜单之前检测环境
- 展示精确的 4 个选项（detached HEAD 时为 3 个）
- 选项 4 需要输入确认
- 仅在选项 1 和 4 时清理 worktree
- 在移除 worktree 前 `cd` 到主仓库根目录
- 移除后运行 `git worktree prune`
