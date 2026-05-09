---
name: requesting-code-review
description: 在完成任务、实现主要功能或合并前使用，以验证工作是否满足要求
---

# 请求代码审查

派发一个代码审查 subagent 来在问题蔓延前发现它们。审查者获得的是精心构建的评审上下文——永远不是你会话的历史。这让审查者聚焦于工作产出而非你的思考过程，同时也为你保留了自己的上下文以便继续工作。

**核心原则：** 早审查，勤审查。

## 何时请求审查

**强制：**
- subagent 驱动开发中每个任务完成后
- 完成主要功能后
- 合并到 main 之前

**可选但有价值：**
- 卡住时（获取新视角）
- 重构前（基线检查）
- 修复复杂 bug 后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 派发代码审查 subagent：**

使用 Task tool，类型为 `general-purpose`，填充 `code-reviewer.md` 中的模板

**占位符：**
- `{DESCRIPTION}` - 你构建内容的简要摘要
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始提交
- `{HEAD_SHA}` - 结束提交

**3. 处理反馈：**
- 立即修复 Critical 问题
- 在继续之前修复 Important 问题
- 记录 Minor 问题留待稍后处理
- 如果审查者有误，用理由反驳

## 示例

```
[刚完成任务 2：添加验证函数]

你：让我在继续前请求代码审查。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[派发代码审查 subagent]
  DESCRIPTION: 添加了 verifyIndex() 和 repairIndex()，支持 4 种问题类型
  PLAN_OR_REQUIREMENTS: docs/zonedev/plans/deployment-plan.md 中的任务 2
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[Subagent 返回]:
  优点：架构清晰，测试真实
  问题：
    Important：缺少进度指示器
    Minor：报告间隔的魔法数字 (100)
  评估：可以继续

你：[修复进度指示器]
[继续任务 3]
```

## 与工作流的集成

**Subagent 驱动开发：**
- 每个任务完成后审查
- 在问题叠加之前发现它们
- 修复后再进入下一个任务

**执行计划：**
- 每个任务或在自然检查点后审查
- 获取反馈，应用，继续

**临时开发：**
- 合并前审查
- 卡住时审查

## 危险信号

**永远不要：**
- 因为"太简单了"而跳过审查
- 忽略 Critical 问题
- 在未修复 Important 问题时继续
- 与有效的技术反馈争论

**如果审查者有误：**
- 用技术理由反驳
- 展示证明其可行的代码/测试
- 请求澄清

参见模板：requesting-code-review/code-reviewer.md
