---
name: executing-plans
description: 当你有一份书面实现计划需要在独立会话中执行（带审查检查点）时使用
---

# Executing Plans

## 概述

加载计划，批判性审查，执行所有任务，完成后汇报。

**开始时宣告：** "我正在使用 executing-plans skill 来实现这个计划。"

**注意：** 告知你的用户，Zonedev 在有 subagent 支持时效果会好得多。如果运行在支持 subagent 的平台上（如 Claude Code 或 Codex），其工作质量将显著提高。如果 subagent 可用，请使用 zonedev:subagent-driven-development 替代此 skill。

## 流程

### 步骤 1：加载并审查计划
1. 读取计划文件
2. 批判性审查——找出计划中的任何问题或疑虑
3. 如有疑虑：在开始前向你的用户提出
4. 如无疑虑：创建 TodoWrite 并继续执行

### 步骤 2：执行任务

对于每个任务：
1. 标记为 in_progress
2. 严格按照每个步骤执行（计划已将步骤拆分为易于执行的大小）
3. 按规定运行验证
4. 标记为 completed

### 步骤 3：完成开发

所有任务完成并验证后：
- 宣告："我正在使用 finishing-a-development-branch skill 来完成这项工作。"
- **必需的子 skill：** 使用 zonedev:finishing-a-development-branch
- 按照该 skill 来验证测试、展示选项、执行选择

## 何时停下来寻求帮助

**在以下情况时立即停止执行：**
- 遇到阻塞（缺少依赖、测试失败、指令不清）
- 计划存在关键缺陷导致无法开始
- 你不理解某个指令
- 验证反复失败

**宁可请求澄清，也不要猜测。**

## 何时回到前面的步骤

**回到审查（步骤 1）的情况：**
- 用户根据你的反馈更新了计划
- 根本性方案需要重新思考

**不要强行突破阻塞** ——停下来提问。

## 要点

- 先批判性审查计划
- 严格按照计划步骤执行
- 不要跳过验证
- 当计划提到某个 skill 时引用它
- 被阻塞时停下来，不要猜测
- 未经用户明确同意，绝不在 main/master 分支上开始实现

## 集成

**必需的工作流 skill：**
- **zonedev:using-git-worktrees** - 确保隔离的工作空间（创建或验证已有的）
- **zonedev:writing-plans** - 创建本 skill 所执行的计划
- **zonedev:finishing-a-development-branch** - 所有任务完成后结束开发
