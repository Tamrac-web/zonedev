# 可视化助手指南

基于浏览器的可视化头脑风暴助手，用于展示原型、图表和选项。

## 何时使用

逐问题判断，而非整个会话统一决定。判断标准：**用户看到它会比读到它理解得更好吗？**

**使用浏览器**——当内容本身就是视觉化的：

- **UI 原型** —— 线框图、布局、导航结构、组件设计
- **架构图** —— 系统组件、数据流、关系图
- **并排视觉对比** —— 对比两种布局、两套配色方案、两个设计方向
- **设计打磨** —— 当问题关乎观感、间距、视觉层级
- **空间关系** —— 状态机、流程图、以图表方式渲染的实体关系

**使用终端**——当内容是文本或表格性质的：

- **需求和范围问题** —— "X 是什么意思？"、"哪些功能在范围内？"
- **概念性 A/B/C 选择** —— 在用文字描述的方案中选择
- **权衡列表** —— 优缺点、对比表
- **技术决策** —— API 设计、数据建模、架构方案选择
- **澄清问题** —— 答案是文字而非视觉偏好的任何问题

关于 UI 话题的问题不一定就是视觉问题。"你想要什么样的向导？"是概念性的——用终端。"这些向导布局中哪个感觉更好？"是视觉性的——用浏览器。

## 工作原理

服务器监听一个目录中的 HTML 文件，将最新的文件提供给浏览器。你将 HTML 内容写入 `screen_dir`，用户在浏览器中看到并可以点击选择选项。选择记录会写入 `state_dir/events`，你在下一轮对话中读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器会原样提供（只注入辅助脚本）。否则，服务器会自动将你的内容包装在框架模板中——添加页头、CSS 主题、选择指示器和所有交互基础设施。**默认编写内容片段。** 只有当你需要完全控制页面时才编写完整文档。

## 启动会话

```bash
# 启动服务器并持久化（原型保存到项目中）
scripts/start-server.sh --project-dir /path/to/project

# 返回值示例：{"type":"server-started","port":52341,"url":"http://localhost:52341",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。告知用户打开该 URL。

**查找连接信息：** 服务器会将启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动了服务器而没有捕获 stdout，读取该文件以获取 URL 和端口。使用 `--project-dir` 时，检查 `<project>/.superpowers/brainstorm/` 目录查找会话目录。

**注意：** 传入项目根目录作为 `--project-dir`，这样原型会持久化到 `.superpowers/brainstorm/` 中，服务器重启后仍然保留。不传的话，文件会放到 `/tmp` 并被清理。提醒用户将 `.superpowers/` 添加到 `.gitignore`（如果还没有的话）。

**按平台启动服务器：**

**Claude Code (macOS / Linux)：**
```bash
# 默认模式即可——脚本自身会将服务器放到后台
scripts/start-server.sh --project-dir /path/to/project
```

**Claude Code (Windows)：**
```bash
# Windows 会自动检测并使用前台模式，这会阻塞 tool call。
# 在 Bash tool 调用上设置 run_in_background: true，使服务器在对话轮次之间存活。
scripts/start-server.sh --project-dir /path/to/project
```
通过 Bash tool 调用时，设置 `run_in_background: true`。然后在下一轮对话中读取 `$STATE_DIR/server-info` 获取 URL 和端口。

**Codex：**
```bash
# Codex 会回收后台进程。脚本会自动检测 CODEX_CI 并切换到前台模式。
# 正常运行即可——不需要额外参数。
scripts/start-server.sh --project-dir /path/to/project
```

**Gemini CLI：**
```bash
# 使用 --foreground，并在 shell tool 调用中设置 is_background: true，
# 使进程在对话轮次之间存活
scripts/start-server.sh --project-dir /path/to/project --foreground
```

**其他环境：** 服务器必须在对话轮次之间持续运行于后台。如果你的环境会回收分离的进程，使用 `--foreground` 并通过平台的后台执行机制启动命令。

如果 URL 无法从浏览器访问（在远程/容器化环境中常见），绑定一个非 loopback 主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 来控制返回的 URL JSON 中打印的主机名。

## 循环流程

1. **检查服务器存活**，然后**将 HTML 写入** `screen_dir` 中的新文件：
   - 每次写入前，检查 `$STATE_DIR/server-info` 是否存在。如果不存在（或 `$STATE_DIR/server-stopped` 存在），说明服务器已关闭——在继续之前用 `start-server.sh` 重新启动。服务器在 30 分钟不活动后会自动退出。
   - 使用语义化的文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **永远不要重用文件名** —— 每个屏幕都用新文件
   - 使用 Write tool —— **不要用 cat/heredoc**（会在终端中产生噪音输出）
   - 服务器自动提供最新的文件

2. **告知用户预期内容并结束本轮对话：**
   - 提醒他们 URL（每一步都提醒，不只是第一次）
   - 简要说明屏幕上显示了什么（例如"展示了 3 个主页布局选项"）
   - 请他们在终端中回复："看一看，告诉我你的想法。如果你喜欢某个选项，可以点击选择。"

3. **在你的下一轮对话中** —— 用户在终端回复之后：
   - 如果存在 `$STATE_DIR/events`，读取它——其中包含用户在浏览器中的交互（点击、选择），格式为 JSON 行
   - 将其与用户的终端文字合并以获得完整的反馈
   - 终端消息是主要反馈来源；`state_dir/events` 提供结构化的交互数据

4. **迭代或推进** —— 如果反馈需要修改当前屏幕，写入新文件（例如 `layout-v2.html`）。只有当前步骤验证通过后才进入下一个问题。

5. **返回终端时卸载内容** —— 当下一步不需要浏览器时（例如澄清问题、权衡讨论），推送一个等待屏幕以清除过时的内容：

   ```html
   <!-- 文件名：waiting.html（或 waiting-2.html 等） -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">在终端中继续...</p>
   </div>
   ```

   这可以防止用户盯着一个已经处理过的选择，而对话已经继续了。当下一个视觉问题出现时，照常推送新的内容文件。

6. 重复直到完成。

## 编写内容片段

只编写放在页面内部的内容。服务器会自动将其包装在框架模板中（页头、主题 CSS、选择指示器和所有交互基础设施）。

**最小示例：**

```html
<h2>哪种布局更好？</h2>
<p class="subtitle">考虑可读性和视觉层级</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>单栏布局</h3>
      <p>简洁、专注的阅读体验</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>双栏布局</h3>
      <p>侧边栏导航配合主内容区</p>
    </div>
  </div>
</div>
```

就这样。不需要 `<html>`、CSS 或 `<script>` 标签。服务器会提供这一切。

## 可用的 CSS 类

框架模板为你的内容提供以下 CSS 类：

### Options（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>标题</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

**多选：** 在容器上添加 `data-multiselect` 属性，允许用户选择多个选项。每次点击切换选中状态。指示栏显示选中数量。

```html
<div class="options" data-multiselect>
  <!-- 相同的 option 标记——用户可以选择/取消选择多个 -->
</div>
```

### Cards（视觉设计卡片）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- 原型内容 --></div>
    <div class="card-body">
      <h3>名称</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">预览：仪表盘布局</div>
  <div class="mockup-body"><!-- 你的原型 HTML --></div>
</div>
```

### Split 视图（并排展示）

```html
<div class="split">
  <div class="mockup"><!-- 左侧 --></div>
  <div class="mockup"><!-- 右侧 --></div>
</div>
```

### 优缺点对比

```html
<div class="pros-cons">
  <div class="pros"><h4>优点</h4><ul><li>好处</li></ul></div>
  <div class="cons"><h4>缺点</h4><ul><li>不足</li></ul></div>
</div>
```

### Mock 元素（线框图构建块）

```html
<div class="mock-nav">Logo | 首页 | 关于 | 联系</div>
<div style="display: flex;">
  <div class="mock-sidebar">导航</div>
  <div class="mock-content">主内容区</div>
</div>
<button class="mock-button">操作按钮</button>
<input class="mock-input" placeholder="输入字段">
<div class="placeholder">占位区域</div>
```

### 排版与分区

- `h2` —— 页面标题
- `h3` —— 章节标题
- `.subtitle` —— 标题下方的辅助文字
- `.section` —— 带底部间距的内容块
- `.label` —— 小号大写标签文字

## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互会记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。推送新屏幕时该文件会自动清除。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流展示了用户的探索路径——他们可能点击多个选项后才做出决定。最后一个 `choice` 事件通常是最终选择，但点击的模式可能揭示犹豫或值得追问的偏好。

如果 `$STATE_DIR/events` 不存在，说明用户没有在浏览器中交互——仅使用他们的终端文字。

## 设计提示

- **根据问题调节保真度** —— 布局问题用线框图，打磨问题用精细设计
- **在每个页面上解释问题** —— "哪种布局感觉更专业？"而不只是"选一个"
- **先迭代再推进** —— 如果反馈需要修改当前屏幕，先写一个新版本
- 每屏最多 **2-4 个选项**
- **在重要时使用真实内容** —— 对于摄影作品集，用真实图片（Unsplash）。占位内容会掩盖设计问题。
- **保持原型简洁** —— 聚焦于布局和结构，而非像素级完美的设计

## 文件命名

- 使用语义化的名称：`platform.html`、`visual-style.html`、`layout.html`
- 永远不要重用文件名——每个屏幕必须是新文件
- 迭代时追加版本后缀，如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，原型文件会持久化在 `.superpowers/brainstorm/` 中以供后续参考。只有 `/tmp` 会话会在停止时被删除。

## 参考

- 框架模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
