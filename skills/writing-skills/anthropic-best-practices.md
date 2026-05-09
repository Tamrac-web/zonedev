# Skill 编写最佳实践

> 学习如何编写 Claude 能够发现并成功使用的高效 Skill。

好的 Skill 应当简洁、结构清晰且经过真实场景测试。本指南提供实用的编写决策建议，帮助你编写 Claude 能够有效发现和使用的 Skill。

关于 Skill 工作原理的概念背景，请参阅 [Skills 概述](/en/docs/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows) 是公共资源。你的 Skill 与 Claude 需要知道的一切共享上下文窗口，包括：

* 系统提示
* 对话历史
* 其他 Skill 的元数据
* 你的实际请求

并非 Skill 中的每个 token 都有即时成本。启动时，只有所有 Skill 的元数据（name 和 description）会被预加载。Claude 仅在 Skill 变得相关时才读取 SKILL.md，仅在需要时才读取附加文件。但在 SKILL.md 中保持简洁仍然重要：一旦 Claude 加载它，每个 token 都在与对话历史和其他上下文竞争。

**默认假设**：Claude 本身就非常聪明

只添加 Claude 尚不具备的上下文。对每条信息提出质疑：

* "Claude 真的需要这个解释吗？"
* "我能假设 Claude 知道这个吗？"
* "这段话值得它的 token 开销吗？"

**好的示例：简洁**（约 50 个 token）：

````markdown  theme={null}
## 提取 PDF 文本

使用 pdfplumber 提取文本：

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**差的示例：过于冗长**（约 150 个 token）：

```markdown  theme={null}
## 提取 PDF 文本

PDF（便携式文档格式）文件是一种常见的文件格式，包含
文本、图像和其他内容。要从 PDF 中提取文本，你需要
使用一个库。有很多库可用于 PDF 处理，但我们
推荐 pdfplumber，因为它易于使用且能处理大多数情况。
首先，你需要使用 pip 安装它。然后你可以使用下面的代码……
```

简洁版假设 Claude 知道什么是 PDF 以及库如何工作。

### 设定适当的自由度

让指令的具体程度与任务的脆弱性和可变性相匹配。

**高自由度**（基于文本的指令）：

适用于：

* 多种方法都有效
* 决策取决于上下文
* 启发式方法指导方向

示例：

```markdown  theme={null}
## 代码审查流程

1. 分析代码结构和组织
2. 检查潜在的 bug 或边界情况
3. 建议可读性和可维护性方面的改进
4. 验证是否遵循项目约定
```

**中等自由度**（伪代码或带参数的脚本）：

适用于：

* 存在首选模式
* 一定程度的变化可以接受
* 配置影响行为

示例：

````markdown  theme={null}
## 生成报告

使用此模板并按需定制：

```python
def generate_report(data, format="markdown", include_charts=True):
    # 处理数据
    # 按指定格式生成输出
    # 可选地包含可视化
```
````

**低自由度**（特定脚本，几乎没有参数）：

适用于：

* 操作脆弱且容易出错
* 一致性至关重要
* 必须遵循特定顺序

示例：

````markdown  theme={null}
## 数据库迁移

严格运行此脚本：

```bash
python scripts/migrate.py --verify --backup
```

不要修改命令或添加额外标志。
````

**类比**：把 Claude 想象成一个在路上探索的机器人：

* **两侧有悬崖的窄桥**：只有一条安全的路。提供具体的护栏和精确的指令（低自由度）。例如：必须按精确顺序执行的数据库迁移。
* **没有障碍的开阔地**：多条路都能到达目的地。给出大方向，信任 Claude 找到最佳路线（高自由度）。例如：代码审查中由上下文决定最佳方法。

### 在所有计划使用的模型上测试

Skill 是对模型的扩展，因此效果取决于底层模型。在所有计划使用的模型上测试你的 Skill。

**按模型的测试考量：**

* **Claude Haiku**（快速、经济）：Skill 是否提供了足够的指导？
* **Claude Sonnet**（均衡）：Skill 是否清晰高效？
* **Claude Opus**（强推理能力）：Skill 是否避免了过度解释？

对 Opus 完美适用的内容可能需要为 Haiku 增加更多细节。如果你计划跨多个模型使用 Skill，目标是让指令在所有模型上都能良好工作。

## Skill 结构

<Note>
  **YAML Frontmatter**：SKILL.md 的 frontmatter 需要两个字段：

  * `name` - Skill 的可读名称（最多 64 个字符）
  * `description` - 一行描述 Skill 的功能和使用时机（最多 1024 个字符）

  完整的 Skill 结构详情，请参阅 [Skills 概述](/en/docs/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名约定

使用一致的命名模式使 Skill 更易于引用和讨论。我们推荐使用**动名词形式**（动词 + -ing）作为 Skill 名称，这样能清楚地描述 Skill 提供的活动或能力。

**好的命名示例（动名词形式）**：

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**可接受的替代方案**：

* 名词短语："PDF Processing"、"Spreadsheet Analysis"
* 动作导向："Process PDFs"、"Analyze Spreadsheets"

**避免**：

* 模糊的名称："Helper"、"Utils"、"Tools"
* 过于通用："Documents"、"Data"、"Files"
* 在你的 skill 集合中使用不一致的模式

一致的命名使得：

* 在文档和对话中引用 Skill 更容易
* 一目了然地理解 Skill 的功能
* 组织和搜索多个 Skill
* 维护专业、连贯的 skill 库

### 编写有效的描述

`description` 字段用于 Skill 发现，应包含 Skill 做什么以及何时使用它。

<Warning>
  **始终使用第三人称**。描述会被注入系统提示，不一致的人称视角会导致发现问题。

  * **好：** "Processes Excel files and generates reports"
  * **避免：** "I can help you process Excel files"
  * **避免：** "You can use this to process Excel files"
</Warning>

**具体化并包含关键词**。同时包含 Skill 做什么以及何时使用它的具体触发条件/上下文。

每个 Skill 只有一个 description 字段。描述对于 skill 选择至关重要：Claude 使用它从可能 100+ 个可用 Skill 中选择正确的 Skill。你的描述必须提供足够的细节让 Claude 知道何时选择此 Skill，而 SKILL.md 的其余部分提供实现细节。

有效示例：

**PDF 处理 skill：**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel 分析 skill：**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git Commit Helper skill：**

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

避免模糊的描述如：

```yaml  theme={null}
description: Helps with documents
```

```yaml  theme={null}
description: Processes data
```

```yaml  theme={null}
description: Does stuff with files
```

### 渐进式披露模式

SKILL.md 充当概览，按需引导 Claude 访问详细材料，就像入门指南中的目录。关于渐进式披露如何工作的解释，请参阅概述中的 [Skill 工作原理](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

**实用指导：**

* 保持 SKILL.md 主体在 500 行以内以获得最佳性能
* 接近此限制时将内容拆分到单独文件
* 使用以下模式有效组织指令、代码和资源

#### 视觉概览：从简单到复杂

一个基础 Skill 只需要一个包含元数据和指令的 SKILL.md 文件：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="Simple SKILL.md file showing YAML frontmatter and markdown body" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

随着 Skill 增长，你可以打包额外的内容，Claude 仅在需要时加载：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="Bundling additional reference files like reference.md and forms.md." data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

完整的 Skill 目录结构可能如下：

```
pdf/
├── SKILL.md              # 主要指令（触发时加载）
├── FORMS.md              # 表单填写指南（按需加载）
├── reference.md          # API 参考（按需加载）
├── examples.md           # 使用示例（按需加载）
└── scripts/
    ├── analyze_form.py   # 工具脚本（执行，不加载）
    ├── fill_form.py      # 表单填写脚本
    └── validate.py       # 验证脚本
```

#### 模式 1：高层指南加引用

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF 处理

## 快速开始

使用 pdfplumber 提取文本：
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## 高级功能

**表单填写**：参见 [FORMS.md](FORMS.md) 获取完整指南
**API 参考**：参见 [REFERENCE.md](REFERENCE.md) 获取所有方法
**示例**：参见 [EXAMPLES.md](EXAMPLES.md) 获取常见模式
````

Claude 仅在需要时加载 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：领域特定组织

对于包含多个领域的 Skill，按领域组织内容以避免加载不相关的上下文。当用户询问销售指标时，Claude 只需要读取与销售相关的 schema，而不是财务或营销数据。这可以保持 token 使用量低且上下文聚焦。

```
bigquery-skill/
├── SKILL.md（概览和导航）
└── reference/
    ├── finance.md（营收、计费指标）
    ├── sales.md（商机、管道）
    ├── product.md（API 使用量、功能）
    └── marketing.md（营销活动、归因）
```

````markdown SKILL.md theme={null}
# BigQuery 数据分析

## 可用数据集

**财务**：营收、ARR、计费 → 参见 [reference/finance.md](reference/finance.md)
**销售**：商机、管道、客户 → 参见 [reference/sales.md](reference/sales.md)
**产品**：API 使用量、功能、采纳率 → 参见 [reference/product.md](reference/product.md)
**营销**：营销活动、归因、邮件 → 参见 [reference/marketing.md](reference/marketing.md)

## 快速搜索

使用 grep 查找特定指标：

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式 3：条件细节

展示基础内容，链接到高级内容：

```markdown  theme={null}
# DOCX 处理

## 创建文档

使用 docx-js 创建新文档。参见 [DOCX-JS.md](DOCX-JS.md)。

## 编辑文档

对于简单编辑，直接修改 XML。

**修订标记**：参见 [REDLINING.md](REDLINING.md)
**OOXML 详情**：参见 [OOXML.md](OOXML.md)
```

Claude 仅在用户需要这些功能时才读取 REDLINING.md 或 OOXML.md。

### 避免深层嵌套引用

当引用来自其他被引用文件中的文件时，Claude 可能只会部分读取。遇到嵌套引用时，Claude 可能使用 `head -100` 等命令预览内容而非读取完整文件，导致信息不完整。

**保持引用从 SKILL.md 起只有一层深度**。所有引用文件应直接从 SKILL.md 链接，以确保 Claude 在需要时读取完整文件。

**差的示例：太深**：

```markdown  theme={null}
# SKILL.md
参见 [advanced.md](advanced.md)...

# advanced.md
参见 [details.md](details.md)...

# details.md
这里是实际信息……
```

**好的示例：一层深度**：

```markdown  theme={null}
# SKILL.md

**基础用法**：[SKILL.md 中的指令]
**高级功能**：参见 [advanced.md](advanced.md)
**API 参考**：参见 [reference.md](reference.md)
**示例**：参见 [examples.md](examples.md)
```

### 为较长的参考文件添加目录

对于超过 100 行的参考文件，在顶部添加目录。这确保 Claude 即使在部分读取时也能看到可用信息的全貌。

**示例**：

```markdown  theme={null}
# API 参考

## 目录
- 认证和设置
- 核心方法（创建、读取、更新、删除）
- 高级功能（批量操作、Webhook）
- 错误处理模式
- 代码示例

## 认证和设置
...

## 核心方法
...
```

Claude 可以读取完整文件或按需跳转到特定部分。

关于这种基于文件系统的架构如何实现渐进式披露的详情，请参阅下方高级部分的[运行时环境](#运行时环境)章节。

## 工作流和反馈循环

### 为复杂任务使用工作流

将复杂操作分解为清晰的顺序步骤。对于特别复杂的工作流，提供一个检查清单，Claude 可以复制到其回复中并随着进展逐项勾选。

**示例 1：研究综合工作流**（不含代码的 Skill）：

````markdown  theme={null}
## 研究综合工作流

复制此检查清单并跟踪你的进度：

```
研究进度：
- [ ] 步骤 1：阅读所有源文档
- [ ] 步骤 2：识别关键主题
- [ ] 步骤 3：交叉引用论述
- [ ] 步骤 4：创建结构化摘要
- [ ] 步骤 5：验证引用
```

**步骤 1：阅读所有源文档**

审阅 `sources/` 目录中的每个文档。记录主要论点和支持证据。

**步骤 2：识别关键主题**

在各源文档中寻找模式。哪些主题反复出现？源文档在哪些方面一致或不一致？

**步骤 3：交叉引用论述**

对于每个主要论述，验证它出现在源材料中。记录哪个来源支持每个论点。

**步骤 4：创建结构化摘要**

按主题组织发现。包括：
- 主要论述
- 来自源文档的支持证据
- 冲突的观点（如有）

**步骤 5：验证引用**

检查每个论述是否引用了正确的源文档。如果引用不完整，返回步骤 3。
````

此示例展示了工作流如何应用于不需要代码的分析任务。检查清单模式适用于任何复杂的多步骤过程。

**示例 2：PDF 表单填写工作流**（含代码的 Skill）：

````markdown  theme={null}
## PDF 表单填写工作流

复制此检查清单并在完成时逐项勾选：

```
任务进度：
- [ ] 步骤 1：分析表单（运行 analyze_form.py）
- [ ] 步骤 2：创建字段映射（编辑 fields.json）
- [ ] 步骤 3：验证映射（运行 validate_fields.py）
- [ ] 步骤 4：填写表单（运行 fill_form.py）
- [ ] 步骤 5：验证输出（运行 verify_output.py）
```

**步骤 1：分析表单**

运行：`python scripts/analyze_form.py input.pdf`

这会提取表单字段及其位置，保存到 `fields.json`。

**步骤 2：创建字段映射**

编辑 `fields.json` 为每个字段添加值。

**步骤 3：验证映射**

运行：`python scripts/validate_fields.py fields.json`

在继续之前修复所有验证错误。

**步骤 4：填写表单**

运行：`python scripts/fill_form.py input.pdf fields.json output.pdf`

**步骤 5：验证输出**

运行：`python scripts/verify_output.py output.pdf`

如果验证失败，返回步骤 2。
````

清晰的步骤防止 Claude 跳过关键验证。检查清单帮助 Claude 和你跟踪多步骤工作流的进度。

### 实现反馈循环

**常见模式**：运行验证器 → 修复错误 → 重复

此模式大幅提升输出质量。

**示例 1：风格指南合规**（不含代码的 Skill）：

```markdown  theme={null}
## 内容审查流程

1. 按照 STYLE_GUIDE.md 中的指南起草内容
2. 对照检查清单审查：
   - 检查术语一致性
   - 验证示例遵循标准格式
   - 确认所有必需部分都存在
3. 如发现问题：
   - 记录每个问题并引用具体章节
   - 修订内容
   - 再次审查检查清单
4. 仅在满足所有要求后才继续
5. 最终定稿并保存文档
```

这展示了使用参考文档而非脚本的验证循环模式。"验证器"是 STYLE\_GUIDE.md，Claude 通过读取和比较来执行检查。

**示例 2：文档编辑流程**（含代码的 Skill）：

```markdown  theme={null}
## 文档编辑流程

1. 在 `word/document.xml` 中进行编辑
2. **立即验证**：`python ooxml/scripts/validate.py unpacked_dir/`
3. 如果验证失败：
   - 仔细审查错误消息
   - 修复 XML 中的问题
   - 再次运行验证
4. **仅在验证通过后才继续**
5. 重建：`python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. 测试输出文档
```

验证循环能及早发现错误。

## 内容指南

### 避免时效性信息

不要包含会过时的信息：

**差的示例：时效性**（会变得不正确）：

```markdown  theme={null}
如果你在 2025 年 8 月之前做这件事，使用旧 API。
2025 年 8 月之后，使用新 API。
```

**好的示例**（使用"旧模式"部分）：

```markdown  theme={null}
## 当前方法

使用 v2 API 端点：`api.example.com/v2/messages`

## 旧模式

<details>
<summary>旧版 v1 API（2025-08 已弃用）</summary>

v1 API 使用的是：`api.example.com/v1/messages`

此端点已不再支持。
</details>
```

旧模式部分提供历史上下文而不会使主要内容混乱。

### 使用一致的术语

选定一个术语并在整个 Skill 中一致使用：

**好——一致**：

* 始终使用"API endpoint"
* 始终使用"field"
* 始终使用"extract"

**差——不一致**：

* 混用"API endpoint"、"URL"、"API route"、"path"
* 混用"field"、"box"、"element"、"control"
* 混用"extract"、"pull"、"get"、"retrieve"

一致性帮助 Claude 理解和遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。根据你的需求匹配严格程度。

**对于严格要求**（如 API 响应或数据格式）：

````markdown  theme={null}
## 报告结构

务必使用此精确的模板结构：

```markdown
# [分析标题]

## 执行摘要
[关键发现的一段概述]

## 关键发现
- 发现 1 附带支持数据
- 发现 2 附带支持数据
- 发现 3 附带支持数据

## 建议
1. 具体可操作的建议
2. 具体可操作的建议
```
````

**对于灵活指导**（当适应性有用时）：

````markdown  theme={null}
## 报告结构

以下是一个合理的默认格式，但请根据分析结果自行判断：

```markdown
# [分析标题]

## 执行摘要
[概述]

## 关键发现
[根据你的发现调整章节]

## 建议
[根据具体上下文定制]
```

根据具体分析类型按需调整章节。
````

### 示例模式

对于输出质量依赖于看到示例的 Skill，提供输入/输出对，就像常规提示一样：

````markdown  theme={null}
## 提交消息格式

按照以下示例生成提交消息：

**示例 1：**
输入：添加了基于 JWT token 的用户认证
输出：
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**示例 2：**
输入：修复了报告中日期显示不正确的 bug
输出：
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**示例 3：**
输入：更新了依赖并重构了错误处理
输出：
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

遵循此风格：type(scope): 简要描述，然后详细说明。
````

示例比单纯的描述更能帮助 Claude 理解期望的风格和详细程度。

### 条件工作流模式

引导 Claude 通过决策点：

```markdown  theme={null}
## 文档修改工作流

1. 确定修改类型：

   **创建新内容？** → 遵循下方"创建工作流"
   **编辑现有内容？** → 遵循下方"编辑工作流"

2. 创建工作流：
   - 使用 docx-js 库
   - 从零构建文档
   - 导出为 .docx 格式

3. 编辑工作流：
   - 解包现有文档
   - 直接修改 XML
   - 每次修改后验证
   - 完成后重新打包
```

<Tip>
  如果工作流变得庞大或复杂、步骤很多，考虑将它们推入单独的文件，并告诉 Claude 根据手头的任务读取相应的文件。
</Tip>

## 评估和迭代

### 先构建评估

**在编写大量文档之前先创建评估。** 这确保你的 Skill 解决的是真实问题，而不是记录想象出来的问题。

**评估驱动开发：**

1. **识别差距**：在没有 Skill 的情况下让 Claude 执行代表性任务。记录具体的失败或缺失的上下文
2. **创建评估**：构建三个测试这些差距的场景
3. **建立基线**：衡量没有 Skill 时 Claude 的表现
4. **编写最少的指令**：只创建足够填补差距和通过评估的内容
5. **迭代**：执行评估，与基线比较，并改进

这种方法确保你解决的是实际问题，而不是预测可能永远不会出现的需求。

**评估结构**：

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

<Note>
  此示例展示了一个数据驱动的评估和简单的测试标准。我们目前不提供内置的运行这些评估的方法。用户可以创建自己的评估系统。评估是衡量 Skill 有效性的真实来源。
</Note>

### 与 Claude 迭代开发 Skill

最有效的 Skill 开发过程涉及 Claude 本身。与一个 Claude 实例（"Claude A"）合作创建一个将被其他实例（"Claude B"）使用的 Skill。Claude A 帮助你设计和完善指令，而 Claude B 在真实任务中测试它们。这行得通是因为 Claude 模型既理解如何编写有效的 agent 指令，也理解 agent 需要什么信息。

**创建新 Skill：**

1. **在没有 Skill 的情况下完成任务**：通过常规提示与 Claude A 一起解决问题。在工作过程中，你会自然地提供上下文、解释偏好和分享过程性知识。注意你反复提供的信息。

2. **识别可复用的模式**：完成任务后，识别你提供的哪些上下文对未来类似任务有用。

   **示例**：如果你完成了一个 BigQuery 分析，你可能提供了表名、字段定义、过滤规则（如"始终排除测试账户"）和常见查询模式。

3. **请 Claude A 创建 Skill**："创建一个 Skill 来捕获我们刚才使用的 BigQuery 分析模式。包括表 schema、命名约定和关于过滤测试账户的规则。"

   <Tip>
     Claude 模型原生理解 Skill 格式和结构。你不需要特殊的系统提示或"编写 skill"的 skill 来让 Claude 帮助创建 Skill。只需请 Claude 创建一个 Skill，它就会生成结构正确的 SKILL.md 内容，包含适当的 frontmatter 和主体内容。
   </Tip>

4. **审查简洁性**：检查 Claude A 是否添加了不必要的解释。问："删除关于'什么是 win rate'的解释——Claude 已经知道了。"

5. **改进信息架构**：请 Claude A 更有效地组织内容。例如："把表 schema 组织到单独的参考文件中。我们以后可能会添加更多表。"

6. **在类似任务上测试**：在相关用例中使用该 Skill 与 Claude B（加载了 Skill 的全新实例）合作。观察 Claude B 是否能找到正确的信息、正确应用规则并成功处理任务。

7. **基于观察迭代**：如果 Claude B 遇到困难或遗漏了什么，带着具体情况回到 Claude A："当 Claude 使用这个 Skill 时，它忘记按 Q4 的日期过滤了。我们应该添加一个关于日期过滤模式的部分吗？"

**改进现有 Skill：**

同样的层次化模式在改进 Skill 时继续适用。你在以下之间交替：

* **与 Claude A 合作**（帮助完善 Skill 的专家）
* **与 Claude B 测试**（使用 Skill 执行实际工作的 agent）
* **观察 Claude B 的行为** 并将见解带回 Claude A

1. **在真实工作流中使用 Skill**：给 Claude B（加载了 Skill）实际任务，而非测试场景

2. **观察 Claude B 的行为**：注意它在哪里遇到困难、成功或做出意外选择

   **观察示例**："当我让 Claude B 做区域销售报告时，它写了查询但忘记过滤测试账户，即使 Skill 提到了这个规则。"

3. **回到 Claude A 进行改进**：分享当前的 SKILL.md 并描述你的观察。问："我注意到 Claude B 在我要求区域报告时忘记过滤测试账户。Skill 提到了过滤，但可能不够醒目？"

4. **审查 Claude A 的建议**：Claude A 可能建议重新组织以使规则更突出，使用更强的措辞如"必须过滤"而非"始终过滤"，或重构工作流部分。

5. **应用并测试变更**：使用 Claude A 的改进更新 Skill，然后在类似请求上再次与 Claude B 测试

6. **基于使用重复**：随着你遇到新场景，继续这个观察-完善-测试循环。每次迭代都基于真实 agent 行为改进 Skill，而非基于假设。

**收集团队反馈：**

1. 与团队成员分享 Skill 并观察他们的使用
2. 询问：Skill 是否在预期时激活？指令是否清晰？缺少什么？
3. 纳入反馈以弥补你自己使用模式中的盲区

**为什么这种方法有效**：Claude A 理解 agent 需求，你提供领域专业知识，Claude B 通过真实使用揭示差距，迭代改进基于观察到的行为而非假设改进 Skill。

### 观察 Claude 如何导航 Skill

在迭代 Skill 时，留意 Claude 在实践中实际如何使用它们。观察：

* **意外的探索路径**：Claude 是否以你未预料的顺序读取文件？这可能表明你的结构不如你想的那么直觉
* **遗漏的连接**：Claude 是否未能跟随到重要文件的引用？你的链接可能需要更明确或更醒目
* **对某些部分的过度依赖**：如果 Claude 反复读取同一个文件，考虑是否应该将该内容放到主 SKILL.md 中
* **被忽略的内容**：如果 Claude 从未访问某个打包文件，它可能是不必要的或在主指令中信号不足

基于这些观察而非假设进行迭代。Skill 元数据中的 'name' 和 'description' 特别关键。Claude 使用它们来决定是否在当前任务中触发 Skill。确保它们清楚地描述 Skill 做什么以及何时应该使用。

## 应避免的反模式

### 避免 Windows 风格路径

始终在文件路径中使用正斜杠，即使在 Windows 上：

* 好：`scripts/helper.py`、`reference/guide.md`
* 避免：`scripts\helper.py`、`reference\guide.md`

Unix 风格路径在所有平台上都有效，而 Windows 风格路径在 Unix 系统上会导致错误。

### 避免提供过多选项

除非必要，不要展示多种方案：

````markdown  theme={null}
**差的示例：太多选择**（令人困惑）：
"你可以使用 pypdf，或 pdfplumber，或 PyMuPDF，或 pdf2image，或……"

**好的示例：提供默认方案**（附带备选）：
"使用 pdfplumber 提取文本：
```python
import pdfplumber
```

对于需要 OCR 的扫描 PDF，改用 pdf2image 加 pytesseract。"
````

## 高级：含可执行代码的 Skill

以下部分聚焦于包含可执行脚本的 Skill。如果你的 Skill 仅使用 Markdown 指令，跳到 [高效 Skill 检查清单](#高效-skill-检查清单)。

### 解决问题，不要推诿

为 Skill 编写脚本时，处理错误条件而不是推诿给 Claude。

**好的示例：明确处理错误**：

```python  theme={null}
def process_file(path):
    """处理文件，如果不存在则创建。"""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # 创建默认内容的文件而非失败
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # 提供替代方案而非失败
        print(f"Cannot access {path}, using default")
        return ''
```

**差的示例：推诿给 Claude**：

```python  theme={null}
def process_file(path):
    # 直接失败让 Claude 自己想办法
    return open(path).read()
```

配置参数也应该有理有据并加以文档化，以避免"巫术常量"（Ousterhout 法则）。如果你不知道正确的值，Claude 又怎么能确定？

**好的示例：自文档化**：

```python  theme={null}
# HTTP 请求通常在 30 秒内完成
# 更长的超时考虑了慢速连接
REQUEST_TIMEOUT = 30

# 三次重试平衡了可靠性和速度
# 大多数间歇性故障在第二次重试时解决
MAX_RETRIES = 3
```

**差的示例：魔法数字**：

```python  theme={null}
TIMEOUT = 47  # 为什么是 47？
RETRIES = 5   # 为什么是 5？
```

### 提供工具脚本

即使 Claude 能自己写脚本，预制脚本仍有优势：

**工具脚本的好处**：

* 比生成的代码更可靠
* 节省 token（无需在上下文中包含代码）
* 节省时间（无需生成代码）
* 确保跨使用的一致性

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="Bundling executable scripts alongside instruction files" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

上面的图表展示了可执行脚本如何与指令文件一起工作。指令文件（forms.md）引用脚本，Claude 可以执行它而不将其内容加载到上下文中。

**重要区分**：在指令中明确 Claude 应该：

* **执行脚本**（最常见）："运行 `analyze_form.py` 以提取字段"
* **作为参考阅读**（对于复杂逻辑）："参见 `analyze_form.py` 了解字段提取算法"

对于大多数工具脚本，执行是首选，因为更可靠和高效。参见下方[运行时环境](#运行时环境)部分了解脚本执行的工作原理。

**示例**：

````markdown  theme={null}
## 工具脚本

**analyze_form.py**：从 PDF 提取所有表单字段

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

输出格式：
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**：检查重叠的边界框

```bash
python scripts/validate_boxes.py fields.json
# 返回："OK" 或列出冲突
```

**fill_form.py**：将字段值应用到 PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 使用视觉分析

当输入可以渲染为图像时，让 Claude 分析它们：

````markdown  theme={null}
## 表单布局分析

1. 将 PDF 转换为图像：
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. 分析每个页面图像以识别表单字段
3. Claude 可以通过视觉识别字段位置和类型
````

<Note>
  在此示例中，你需要自己编写 `pdf_to_images.py` 脚本。
</Note>

Claude 的视觉能力有助于理解布局和结构。

### 创建可验证的中间输出

当 Claude 执行复杂的开放式任务时，可能会犯错。"计划-验证-执行"模式通过让 Claude 先以结构化格式创建计划，然后用脚本验证计划后再执行，从而及早发现错误。

**示例**：想象让 Claude 根据电子表格更新 PDF 中的 50 个表单字段。没有验证的话，Claude 可能引用不存在的字段、创建冲突的值、遗漏必填字段或错误地应用更新。

**解决方案**：使用上面展示的工作流模式（PDF 表单填写），但添加一个中间的 `changes.json` 文件，在应用变更前进行验证。工作流变为：分析 → **创建计划文件** → **验证计划** → 执行 → 验证。

**为什么此模式有效：**

* **及早发现错误**：验证在变更应用前发现问题
* **机器可验证**：脚本提供客观验证
* **可逆的计划**：Claude 可以迭代计划而不触碰原件
* **清晰的调试**：错误消息指向具体问题

**何时使用**：批量操作、破坏性变更、复杂验证规则、高风险操作。

**实现提示**：使验证脚本提供详细的具体错误消息，如 "Field 'signature\_date' not found. Available fields: customer\_name, order\_total, signature\_date\_signed"，以帮助 Claude 修复问题。

### 打包依赖

Skill 在代码执行环境中运行，有平台特定的限制：

* **claude.ai**：可以从 npm 和 PyPI 安装包，也可以从 GitHub 仓库拉取
* **Anthropic API**：没有网络访问，没有运行时包安装

在 SKILL.md 中列出所需的包，并验证它们在[代码执行工具文档](/en/docs/agents-and-tools/tool-use/code-execution-tool)中可用。

### 运行时环境

Skill 在具有文件系统访问、bash 命令和代码执行能力的代码执行环境中运行。关于此架构的概念解释，请参阅概述中的 [Skill 架构](/en/docs/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**这对你的编写有什么影响：**

**Claude 如何访问 Skill：**

1. **元数据预加载**：启动时，所有 Skill 的 YAML frontmatter 中的 name 和 description 被加载到系统提示中
2. **文件按需读取**：Claude 在需要时使用 bash Read 工具从文件系统访问 SKILL.md 和其他文件
3. **脚本高效执行**：工具脚本可以通过 bash 执行而无需将其完整内容加载到上下文中。只有脚本的输出消耗 token
4. **大文件无上下文惩罚**：参考文件、数据或文档在实际读取前不消耗上下文 token

* **文件路径很重要**：Claude 像操作文件系统一样导航你的 skill 目录。使用正斜杠（`reference/guide.md`），而非反斜杠
* **描述性文件命名**：使用表明内容的名称：`form_validation_rules.md`，而非 `doc2.md`
* **组织以便发现**：按领域或功能构建目录
  * 好：`reference/finance.md`、`reference/sales.md`
  * 差：`docs/file1.md`、`docs/file2.md`
* **打包全面的资源**：包含完整的 API 文档、大量示例、大型数据集；在访问前无上下文惩罚
* **确定性操作优先使用脚本**：编写 `validate_form.py` 而非让 Claude 生成验证代码
* **明确执行意图**：
  * "运行 `analyze_form.py` 以提取字段"（执行）
  * "参见 `analyze_form.py` 了解提取算法"（作为参考阅读）
* **测试文件访问模式**：通过真实请求验证 Claude 能否导航你的目录结构

**示例：**

```
bigquery-skill/
├── SKILL.md（概览，指向参考文件）
└── reference/
    ├── finance.md（营收指标）
    ├── sales.md（管道数据）
    └── product.md（使用量分析）
```

当用户询问营收时，Claude 读取 SKILL.md，看到指向 `reference/finance.md` 的引用，然后调用 bash 只读取该文件。sales.md 和 product.md 文件保留在文件系统上，在需要之前消耗零上下文 token。这种基于文件系统的模型是渐进式披露的基础。Claude 可以导航并选择性地加载每个任务所需的内容。

完整的技术架构详情，请参阅 Skills 概述中的 [Skill 工作原理](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

如果你的 Skill 使用 MCP（Model Context Protocol）工具，始终使用完全限定的工具名称以避免"tool not found"错误。

**格式**：`ServerName:tool_name`

**示例**：

```markdown  theme={null}
使用 BigQuery:bigquery_schema 工具检索表 schema。
使用 GitHub:create_issue 工具创建 issue。
```

其中：

* `BigQuery` 和 `GitHub` 是 MCP 服务器名称
* `bigquery_schema` 和 `create_issue` 是这些服务器中的工具名称

没有服务器前缀，Claude 可能无法找到工具，尤其是当有多个 MCP 服务器可用时。

### 不要假设工具已安装

不要假设包已可用：

````markdown  theme={null}
**差的示例：假设已安装**：
"使用 pdf 库处理文件。"

**好的示例：明确依赖**：
"安装所需的包：`pip install pypdf`

然后使用它：
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技术说明

### YAML frontmatter 要求

SKILL.md 的 frontmatter 需要 `name`（最多 64 个字符）和 `description`（最多 1024 个字符）字段。参见 [Skills 概述](/en/docs/agents-and-tools/agent-skills/overview#skill-structure) 了解完整的结构详情。

### Token 预算

保持 SKILL.md 主体在 500 行以内以获得最佳性能。如果内容超过此限制，使用前面描述的渐进式披露模式将其拆分到单独文件。架构详情请参阅 [Skills 概述](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

## 高效 Skill 检查清单

在分享 Skill 之前，验证：

### 核心质量

* [ ] 描述具体且包含关键词
* [ ] 描述同时包含 Skill 做什么以及何时使用
* [ ] SKILL.md 主体在 500 行以内
* [ ] 额外细节在单独文件中（如需要）
* [ ] 没有时效性信息（或在"旧模式"部分中）
* [ ] 全篇术语一致
* [ ] 示例具体而非抽象
* [ ] 文件引用只有一层深度
* [ ] 适当使用渐进式披露
* [ ] 工作流有清晰的步骤

### 代码和脚本

* [ ] 脚本解决问题而非推诿给 Claude
* [ ] 错误处理明确且有帮助
* [ ] 没有"巫术常量"（所有值都有依据）
* [ ] 所需的包在指令中列出且已验证可用
* [ ] 脚本有清晰的文档
* [ ] 没有 Windows 风格路径（全部使用正斜杠）
* [ ] 关键操作有验证/核实步骤
* [ ] 质量关键任务包含反馈循环

### 测试

* [ ] 至少创建三个评估
* [ ] 在 Haiku、Sonnet 和 Opus 上测试过
* [ ] 在真实使用场景中测试过
* [ ] 已纳入团队反馈（如适用）

## 后续步骤

<CardGroup cols={2}>
  <Card title="开始使用 Agent Skills" icon="rocket" href="/en/docs/agents-and-tools/agent-skills/quickstart">
    创建你的第一个 Skill
  </Card>

  <Card title="在 Claude Code 中使用 Skills" icon="terminal" href="/en/docs/claude-code/skills">
    在 Claude Code 中创建和管理 Skills
  </Card>

  <Card title="通过 API 使用 Skills" icon="code" href="/en/api/skills-guide">
    以编程方式上传和使用 Skills
  </Card>
</CardGroup>
