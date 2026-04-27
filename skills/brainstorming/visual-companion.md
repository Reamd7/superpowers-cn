# 可视化辅助指南

基于浏览器的可视化头脑风暴辅助工具，用于展示模型图、架构图和选项。

## 何时使用

按问题决定，而非按会话决定。判断标准：**用户看到它是否比读到它更容易理解？**

**内容本身是视觉化的时，使用浏览器：**

- **UI 模型图** — 线框图、布局、导航结构、组件设计
- **架构图** — 系统组件、数据流、关系图
- **并排视觉对比** — 对比两种布局、两种配色方案、两种设计方向
- **设计打磨** — 当问题涉及外观和感觉、间距、视觉层次时
- **空间关系** — 以图表形式呈现的状态机、流程图、实体关系

**内容是文字或表格时，使用终端：**

- **需求和范围问题** — "X 是什么意思？"，"哪些功能在范围内？"
- **概念性的 A/B/C 选择** — 用文字描述的方案之间做选择
- **权衡清单** — 优缺点、对比表
- **技术决策** — API 设计、数据建模、架构方案选择
- **澄清问题** — 答案是文字而非视觉偏好的任何情况

关于 UI 话题的问题不等于自动需要可视化。"你想要什么样的向导？"是概念性的 — 用终端。"这些向导布局哪个感觉更合适？"是视觉性的 — 用浏览器。

## 工作原理

服务器监听目录中的 HTML 文件，并将最新的文件提供给浏览器。你将 HTML 内容写入 `screen_dir`，用户在浏览器中查看并可以点击选择选项。选择结果会记录到 `state_dir/events`，你在下一轮读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器会原样提供（仅注入辅助脚本）。否则，服务器会自动将你的内容包裹在框架模板中 — 添加页头、CSS 主题、选择指示器和所有交互基础设施。**默认写内容片段。** 只有当你需要完全控制页面时才写完整文档。

## 启动会话

```bash
# 启动服务器并持久化（模型图保存到项目中）
scripts/start-server.sh --project-dir /path/to/project

# 返回: {"type":"server-started","port":52341,"url":"http://localhost:52341",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

保存响应中的 `screen_dir` 和 `state_dir`。告诉用户打开 URL。

**查找连接信息：** 服务器将启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动了服务器但未捕获标准输出，可以读取该文件获取 URL 和端口。使用 `--project-dir` 时，检查 `<project>/.superpowers/brainstorm/` 中的会话目录。

**注意：** 将项目根目录作为 `--project-dir` 传入，这样模型图会持久保存在 `.superpowers/brainstorm/` 中，在服务器重启后仍然保留。不传的话，文件会保存到 `/tmp` 并被清理。提醒用户将 `.superpowers/` 添加到 `.gitignore`（如果还没有的话）。

**各平台启动服务器的方式：**

**Claude Code (macOS / Linux):**
```bash
# 默认模式即可 — 脚本会自动将服务器置于后台
scripts/start-server.sh --project-dir /path/to/project
```

**Claude Code (Windows):**
```bash
# Windows 自动检测并使用前台模式，这会阻塞工具调用。
# 在 Bash 工具调用上设置 run_in_background: true，使服务器在
# 对话轮次之间保持运行。
scripts/start-server.sh --project-dir /path/to/project
```
通过 Bash 工具调用时，设置 `run_in_background: true`。然后在下一轮读取 `$STATE_DIR/server-info` 获取 URL 和端口。

**Codex:**
```bash
# Codex 会回收后台进程。脚本自动检测 CODEX_CI 并
# 切换到前台模式。正常运行即可 — 无需额外参数。
scripts/start-server.sh --project-dir /path/to/project
```

**Gemini CLI:**
```bash
# 使用 --foreground 并在 shell 工具调用上设置 is_background: true，
# 使进程在轮次之间保持运行
scripts/start-server.sh --project-dir /path/to/project --foreground
```

**其他环境：** 服务器必须在对话轮次之间持续在后台运行。如果你的环境会回收分离的进程，使用 `--foreground` 并配合平台的后台执行机制启动命令。

如果从浏览器无法访问 URL（在远程/容器化环境中常见），绑定非回环地址：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 循环流程

1. **检查服务器是否存活**，然后 **写入 HTML** 到 `screen_dir` 的新文件中：
   - 每次写入前，检查 `$STATE_DIR/server-info` 是否存在。如果不存在（或 `$STATE_DIR/server-stopped` 存在），说明服务器已关闭 — 用 `start-server.sh` 重启后再继续。服务器在 30 分钟不活动后自动退出。
   - 使用语义化文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **永远不要复用文件名** — 每个屏幕使用新文件
   - 使用 Write 工具 — **永远不要用 cat/heredoc**（会在终端输出噪音）
   - 服务器自动提供最新的文件

2. **告诉用户预期内容，然后结束你的回合：**
   - 提醒他们 URL（每一步都要，不只是第一步）
   - 简要文字说明屏幕上的内容（例如："展示首页的 3 种布局选项"）
   - 请他们在终端回复："看一下，告诉我你的想法。如果想选的话可以点击选择。"

3. **在你的下一轮** — 用户在终端回复后：
   - 如果 `$STATE_DIR/events` 存在则读取 — 包含用户在浏览器中的交互（点击、选择），格式为 JSON 行
   - 与用户的终端文字合并以获取完整信息
   - 终端消息是主要反馈；`state_dir/events` 提供结构化的交互数据

4. **迭代或推进** — 如果反馈改变了当前屏幕，写入新文件（例如 `layout-v2.html`）。只有当前步骤确认后才进入下一个问题。

5. **返回终端时卸载** — 当下一步不需要浏览器时（例如澄清问题、权衡讨论），推送一个等待屏幕以清除过时内容：

   ```html
   <!-- filename: waiting.html (or waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   这可以防止用户盯着一个已解决的选择，而对话已经进行到下一步。当下一个视觉问题出现时，像往常一样推送新的内容文件。

6. 重复直到完成。

## 编写内容片段

只需写入页面中的内容。服务器会自动将其包裹在框架模板中（页头、主题 CSS、选择指示器和所有交互基础设施）。

**最小示例：**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

就是这样。不需要 `<html>`、不需要 CSS、不需要 `<script>` 标签。服务器会提供所有这些。

## 可用的 CSS 类

框架模板为你的内容提供以下 CSS 类：

### Options（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**多选：** 在容器上添加 `data-multiselect` 让用户选择多个选项。每次点击切换选中状态。指示条显示选中数量。

```html
<div class="options" data-multiselect>
  <!-- 同样的 option 标记 — 用户可以选择/取消选择多个 -->
</div>
```

### Cards（视觉设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### Split view（并排视图）

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### Pros/Cons（优缺点）

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### Mock 元素（线框构建块）

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 排版和分区

- `h2` — 页面标题
- `h3` — 分区标题
- `.subtitle` — 标题下的辅助文字
- `.section` — 带底部间距的内容块
- `.label` — 小号大写标签文字

## 浏览器事件格式

用户在浏览器中点击选项时，交互记录会写入 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新屏幕时，文件会自动清空。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流展示了用户的探索路径 — 他们可能在确定之前点击多个选项。最后一个 `choice` 事件通常是最终选择，但点击模式可能揭示犹豫或值得询问的偏好。

如果 `$STATE_DIR/events` 不存在，说明用户没有与浏览器交互 — 仅使用他们的终端文字。

## 设计技巧

- **根据问题调整保真度** — 布局问题用线框图，打磨问题用精细设计
- **在每个页面上说明问题** — "哪种布局感觉更专业？"而不是仅仅"选一个"
- **在推进之前迭代** — 如果反馈改变了当前屏幕，写一个新版本
- **每个屏幕最多 2-4 个选项**
- **在重要时使用真实内容** — 对于摄影作品集，使用真实图片（Unsplash）。占位内容会掩盖设计问题。
- **保持模型图简洁** — 关注布局和结构，而非像素级精确设计

## 文件命名

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 永远不要复用文件名 — 每个屏幕必须是新文件
- 迭代时：追加版本后缀，如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，模型图文件会持久保存在 `.superpowers/brainstorm/` 中供后续参考。只有 `/tmp` 会话在停止时会被删除。

## 参考

- Frame 模板（CSS 参考）：`scripts/frame-template.html`
- Helper 脚本（客户端）：`scripts/helper.js`
