# 根因追踪

## 概述

Bug 通常在调用栈的深处显现（在错误的目录执行 git init、文件创建在错误的位置、数据库使用了错误的路径）。你的本能是在错误出现的地方修复，但那是在治标。

**核心原则：** 沿调用链反向追踪，直到找到原始触发点，然后在源头修复。

## 何时使用

```dot
digraph when_to_use {
    "Bug 出现在栈的深处？" [shape=diamond];
    "能反向追踪吗？" [shape=diamond];
    "在症状处修复" [shape=box];
    "追踪到原始触发点" [shape=box];
    "更好的做法：同时添加纵深防御" [shape=box];

    "Bug 出现在栈的深处？" -> "能反向追踪吗？" [label="是"];
    "能反向追踪吗？" -> "追踪到原始触发点" [label="是"];
    "能反向追踪吗？" -> "在症状处修复" [label="否 - 死胡同"];
    "追踪到原始触发点" -> "更好的做法：同时添加纵深防御";
}
```

**使用场景：**
- 错误发生在执行的深处（不是入口处）
- 堆栈跟踪显示很长的调用链
- 不清楚无效数据从哪里来
- 需要找到哪个测试/代码触发了问题

## 追踪过程

### 1. 观察症状
```
Error: git init failed in ~/project/packages/core
```

### 2. 找到直接原因
**什么代码直接导致了这个问题？**
```typescript
await execFileAsync('git', ['init'], { cwd: projectDir });
```

### 3. 问：什么调用了这里？
```typescript
WorktreeManager.createSessionWorktree(projectDir, sessionId)
  → 被 Session.initializeWorkspace() 调用
  → 被 Session.create() 调用
  → 被测试中的 Project.create() 调用
```

### 4. 持续向上追踪
**传入了什么值？**
- `projectDir = ''`（空字符串！）
- 空字符串作为 `cwd` 会解析为 `process.cwd()`
- 那就是源代码目录！

### 5. 找到原始触发点
**空字符串从哪里来？**
```typescript
const context = setupCoreTest(); // 返回 { tempDir: '' }
Project.create('name', context.tempDir); // 在 beforeEach 之前就访问了！
```

## 添加堆栈跟踪

当你无法手动追踪时，添加诊断埋点：

```typescript
// 在有问题的操作之前
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    nodeEnv: process.env.NODE_ENV,
    stack,
  });

  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**关键：** 在测试中使用 `console.error()`（不要用 logger —— 可能不会显示）

**运行并捕获：**
```bash
npm test 2>&1 | grep 'DEBUG git init'
```

**分析堆栈跟踪：**
- 查找测试文件名
- 找到触发调用的行号
- 识别模式（同一个测试？同一个参数？）

## 找到哪个测试造成了污染

如果某些东西在测试期间出现但你不知道是哪个测试：

使用本目录中的二分查找脚本 `find-polluter.sh`：

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

逐个运行测试，在找到第一个污染者时停止。查看脚本了解使用方法。

## 实际案例：空的 projectDir

**症状：** `.git` 被创建在了 `packages/core/`（源代码目录）

**追踪链：**
1. `git init` 在 `process.cwd()` 中执行 ← 空的 cwd 参数
2. WorktreeManager 被传入了空的 projectDir
3. Session.create() 传入了空字符串
4. 测试在 beforeEach 之前访问了 `context.tempDir`
5. setupCoreTest() 初始时返回 `{ tempDir: '' }`

**根本原因：** 顶层变量初始化时访问了空值

**修复：** 将 tempDir 改为 getter，在 beforeEach 之前访问时抛出异常

**同时添加了纵深防御：**
- 第一层：Project.create() 校验目录
- 第二层：WorkspaceManager 校验非空
- 第三层：NODE_ENV 守卫拒绝在 tmpdir 之外执行 git init
- 第四层：git init 之前记录堆栈跟踪

## 核心原则

```dot
digraph principle {
    "找到直接原因" [shape=ellipse];
    "能再往上追踪一层吗？" [shape=diamond];
    "反向追踪" [shape=box];
    "这是源头吗？" [shape=diamond];
    "在源头修复" [shape=box];
    "在每一层添加校验" [shape=box];
    "Bug 不可能发生" [shape=doublecircle];
    "绝不只修复症状" [shape=octagon, style=filled, fillcolor=red, fontcolor=white];

    "找到直接原因" -> "能再往上追踪一层吗？";
    "能再往上追踪一层吗？" -> "反向追踪" [label="是"];
    "能再往上追踪一层吗？" -> "绝不只修复症状" [label="否"];
    "反向追踪" -> "这是源头吗？";
    "这是源头吗？" -> "反向追踪" [label="否 - 继续向上"];
    "这是源头吗？" -> "在源头修复" [label="是"];
    "在源头修复" -> "在每一层添加校验";
    "在每一层添加校验" -> "Bug 不可能发生";
}
```

**绝不只在错误出现的地方修复。** 反向追踪以找到原始触发点。

## 堆栈跟踪技巧

**在测试中：** 使用 `console.error()` 而非 logger —— logger 可能被抑制
**在操作之前：** 在危险操作之前记录日志，而不是在失败之后
**包含上下文：** 目录、cwd、环境变量、时间戳
**捕获堆栈：** `new Error().stack` 显示完整的调用链

## 实际效果

来自调试实践（2025-10-03）的数据：
- 通过 5 层追踪找到了根本原因
- 在源头修复（getter 校验）
- 添加了 4 层防御
- 1847 个测试通过，零污染
