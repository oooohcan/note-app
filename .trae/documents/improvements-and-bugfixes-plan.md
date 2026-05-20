# 待办清单改进与Bug修复计划

## 项目分析

项目位于 `/workspace/index.html`，是一个单文件待办清单应用（Vanilla JS，无框架），包含：
- 自定义 DOM 构建器 `h()`（L513-L535）
- 拖拽排序模块 `DragDrop`（L540-L608）
- 应用核心 `App` 命名空间（L613-L1846）
- Gitee API 数据持久化
- 树形嵌套待办结构

---

## 1. 输入框展开功能

**问题**: 顶部添加任务输入框 `.add-bar input`（L461）是固定单行 `<input>`，长文本不易查看和编辑。

**实现方案**:

### 1.1 HTML 结构修改
- 将 `<input type="text" id="addInput" ...>` 改为 `<textarea id="addInput" ...>`，支持多行文本展开

### 1.2 CSS 调整
- 将 `.add-bar input` 样式改为适用于 `textarea`：
  - 默认高度为单行（～46px），`resize: none` 防止手动拖拽 resize
  - 添加 `min-height` 和 `max-height`（如 max-height: 200px）
  - 设置 `overflow-y: auto` 允许滚动
  - `transition: height 0.15s ease` 平滑过渡
- 垂直布局调整：当 textarea 高度变化时 `.add-bar` 适配

### 1.3 JS 自适应高度
在 `bindEvents()` 中为 `#addInput` 绑定 `input` 事件，实现自动扩展：
- 每次输入时重置高度为 `min-height`，然后设置为 `scrollHeight`
- 限制最大高度为 `max-height`

### 1.4 快捷键适配
- `Enter` 键提交（保持现有行为）
- `Shift+Enter` 换行

### 涉及文件/位置
- [index.html:L84-L91](file:///workspace/index.html#L84-L91) — `.add-bar input` CSS
- [index.html:L461](file:///workspace/index.html#L461) — HTML input 元素
- [index.html:L1722-L1736](file:///workspace/index.html#L1722-L1736) — 添加待办事件绑定

---

## 2. 设置增加默认折叠开关

**问题**: 需要在设置面板中添加一个开关，控制子任务是否默认折叠。

**实现方案**:

### 2.1 状态管理
- 在 `App.state` 中新增字段 `defaultCollapse`（L616-L627），从 `localStorage` 读取：
  ```js
  defaultCollapse: localStorage.getItem('defaultCollapse') === 'true'
  ```

### 2.2 设置面板 HTML
- 在设置弹窗（L475-L493）中添加新的 toggle 行，放在「显示已完成待办」切换下方：
  ```html
  <div class="toggle-row">
    <span>默认折叠子任务</span>
    <label class="toggle-switch">
      <input type="checkbox" id="toggleDefaultCollapse">
      <span class="toggle-slider"></span>
    </label>
  </div>
  ```

### 2.3 设置面板 JS
- `settingsShow()` 中同步 checkbox 状态（L1634-L1639）
- 添加 `change` 事件监听（参考 L1745-L1752 的 `toggleShowCompleted` 实现）
- 切换时保存到 `localStorage`

### 2.4 渲染时应用默认折叠
修改 `actionToggleComplete()`（L1390-L1469）和 `renderTodoItem()`（L1194-L1236）：
- 在 `renderTodoItem()` 中，当 `App.state.defaultCollapse` 为 `true` 且 item 有 children 时，初始渲染为折叠状态
- 具体做法：在渲染时，如果 `defaultCollapse` 为 true 且 `collapsed` 状态中未显式存储该 item，自动设置 `App.state.collapsed[id] = true`

### 涉及文件/位置
- [index.html:L616-L627](file:///workspace/index.html#L616-L627) — App.state 定义
- [index.html:L477-L493](file:///workspace/index.html#L477-L493) — 设置面板 HTML
- [index.html:L1634-L1639](file:///workspace/index.html#L1634-L1639) — settingsShow
- [index.html:L1745-L1752](file:///workspace/index.html#L1745-L1752) — 参考：showCompleted 切换
- [index.html:L1194-L1236](file:///workspace/index.html#L1194-L1236) — renderTodoItem

---

## 3. 拖动灵敏度提升

**问题**: 拖拽触发延时较长（鼠标 400ms / 触摸 500ms），且只在按住非交互元素时触发。用户希望长按整行卡片即可拖拽。

**实现方案**:

### 3.1 降低长按延时
- 鼠标拖拽延时：400ms → 250ms
- 触摸拖拽延时：500ms → 300ms

### 3.2 整行卡片可拖拽
当前 `mousedown`/`touchstart` 处理中会检查 `e.target.closest('button,input,[data-action]')` 然后 return（L1787, L1808）。修改为：
- 同样检查这些元素，但如果命中的是 `.btn-check`（复选框）或 `.btn-collapse`（折叠按钮），仍然允许拖拽（这些元素较小，长按整行更自然）
- 保留对 `.todo-text`、`.btn-del`、`.btn-sub`、`input` 等元素的排除
- 或者更简单：只排除 `input` 和 `.btn-del.confirm` 和 `.inline-add-row` 内元素

### 3.3 移动阈值调整
位移阈值保持 10px（L1796, L1819），此值已合理。

### 涉及文件/位置
- [index.html:L1784-L1793](file:///workspace/index.html#L1784-L1793) — 鼠标 mousedown
- [index.html:L1806-L1814](file:///workspace/index.html#L1806-L1814) — 触摸 touchstart

---

## 4. Bug修复：点击已有待办文本时全选文字

**问题**: `renderStartEdit()`（L1325-L1348）中调用 `input.select()` 导致 iOS/Safari 全选文字，不方便修改。

**原因分析**: L1332 行 `input.select()` 会将输入框内容全选，这在移动端 Safari 上行为特别激进。

**修复方案**:
- 将 `input.select()` 改为将光标定位到文本末尾：
  ```js
  input.setSelectionRange(input.value.length, input.value.length);
  ```
- 如果用户希望更方便定位，可以改为单击时把光标放到点击位置。但这里是用 `input` 替换 `span`，进入编辑模式时用 `focus()` 获得焦点即可，不需要 `select()`。

### 涉及文件/位置
- [index.html:L1325-L1348](file:///workspace/index.html#L1325-L1348) — renderStartEdit，第 L1332 行

---

## 5. Bug修复：iOS 输入英文时光标跳至前一个中文字符前

**问题**: 在 iOS/Safari 上编辑已有待办时，输入英文后光标会跳到前一个中文字符之前。

**原因分析**: 这是一个已知的 iOS Safari contenteditable/input 组件的渲染 bug。当包含中英文混合文本时，Safari 的 composition（输入法合成）事件处理可能导致光标位置计算错误。`input.select()` 全选后，用户取消选择开始输入时，光标位置在 iOS 上表现异常。

**修复方案**:
- 方案 A（推荐）：将编辑模式下的 `<input>` 改为 `<textarea>` 或使用 `contenteditable` div。文本区域在 iOS 上对中文输入支持更好。但考虑到一致性，使用 `<textarea>` 更简单。
- 方案 B：在 `renderStartEdit` 中使用 `contenteditable` 的 `<span>` 而不是创建新的 `<input>`，利用元素自身的 `contenteditable` 属性。
- 方案 C：将 `<input>` 的 `type` 保持但添加 `autocorrect="off"` `autocapitalize="none"` `spellcheck="false"` 属性，并在 `compositionend` 事件中手动修正光标位置。

**推荐方案 B**（最简洁）：将 `renderStartEdit` 中的 `<input>` 替换为直接在 `.todo-text` 元素上设置 `contenteditable="true"`：
- 设置 `textEl.contentEditable = 'true'`
- 使用 `textEl.focus()` 并设置光标到末尾
- 在 `blur` / `keydown` 事件中提交，恢复 `contentEditable = 'false'`
- 需要更新 CSS：已经定义了 `[contenteditable]` 的 `user-select: text`（L50），但需要添加编辑态样式（如边框高亮）

### 涉及文件/位置
- [index.html:L50](file:///workspace/index.html#L50) — contenteditable CSS
- [index.html:L1325-L1348](file:///workspace/index.html#L1325-L1348) — renderStartEdit
- [index.html:L214-L218](file:///workspace/index.html#L214-L218) — .inline-input 样式（可能需要新增 .todo-text.editing 样式）

---

## 实施步骤总结

| 步骤 | 内容 | 影响范围 |
|------|------|----------|
| 1 | 输入框改为 textarea，添加自适应高度 | HTML + CSS + JS |
| 2 | 设置面板添加「默认折叠子任务」开关 | HTML + JS state + 渲染逻辑 |
| 3 | 降低拖拽延时 + 整行卡片可拖拽 | JS（拖拽事件处理） |
| 4 | 修复编辑时全选文字 → 光标定位到末尾 | JS（renderStartEdit） |
| 5 | 修复 iOS 输入英文光标跳动 → 用 contenteditable 替代 input | JS + CSS |