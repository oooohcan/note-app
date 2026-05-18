# 待办清单 (Todo PWA)

纯 HTML 单文件待办清单 PWA，使用 Gitee API v5 存储数据到 `oooohcan/note` 仓库的 `todos.json`。

## 项目结构

```
note-app/
├── index.html      # 主应用（HTML + CSS + JS 全部内嵌）
├── manifest.json   # PWA 清单
└── README.md
```

## 快速启动

```bash
python -m http.server 8080
# 访问 http://localhost:8080
```

## 技术架构

- **CSS** — 9 大模块分区：变量 → 布局 → 列表 → 控件 → 回收站 → 模态框 → 拖拽 → 加载 → 响应式
- **JS** — App 命名空间体系：
  - `h()` — 轻量 DOM 构建器，textContent 防 XSS
  - `DragDrop` — 独立拖拽类（mouse + touch）
  - `App.state` / `App.dom` — 状态与 DOM 缓存
  - `App.api*` — Gitee API v5 封装（获取文件、创建/更新文件、base64 编解码）
  - `App.tree*` — 递归树操作（findById, removeById, getDepth, filter, sort, moveItem）
  - `App.render*` — 渲染引擎（renderTodoItem, renderCheckbox, renderInlineAdd, renderStartEdit）
  - `App.action*` — 业务操作（addTodo, toggleComplete, deleteTodo 等）
  - `App.clickRoutes` — 事件路由表（替代 switch-case）

## 关键约束

- 最多 6 层嵌套（depth 0-5），超出隐藏添加按钮并阻止拖入
- 数据仅操作 `todos.json`，不触碰仓库其他文件
- 认证令牌存储在 localStorage (`gitee_token`)
- 保存冲突时自动重新获取 SHA 并重试
- Apple 风格配色：蓝紫渐变 `#4B7BFF → #6C5CE7`，背景 `#F5F5F7`
- iOS PWA 安全区域：`viewport-fit=cover` + `env(safe-area-inset-top)`，header 和 toast 需适配
