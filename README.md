# 待办清单 (Todo PWA)

纯 HTML 待办清单网页应用，可安装为 PWA，使用 Gitee 仓库存储数据。

## 快速开始

### 1. 获取 Gitee 令牌

前往 [Gitee 私人令牌](https://gitee.com/profile/personal_access_tokens) 创建一个访问令牌，勾选 `projects` 权限。

### 2. 启动服务

```bash
cd C:\Users\Sean\Documents\note
python -m http.server 8080
```

### 3. 打开浏览器

访问 **http://localhost:8080**，首次打开会提示输入令牌。

## 功能

| 功能 | 说明 |
|------|------|
| 多级子任务 | 最多 6 层嵌套，超层隐藏 + 按钮、阻止拖入 |
| 拖拽排序 | 长按拖拽调整顺序与层级（桌面鼠标 + 移动端触控） |
| 折叠展开 | 有子任务时显示 ▸/▾ 切换，已完成默认折叠 |
| 任务完成 | 点击圆形勾选框，自动排到底部并置灰 |
| 显示切换 | 设置中切换显示/隐藏已完成待办 |
| 回收站 | 删除后进入回收站，可恢复或彻底删除 |
| Gitee 同步 | 手动保存到仓库 `oooohcan/note`，冲突自动重试 |
| 自动加载 | 每次打开页面自动拉取最新数据 |
| PWA 安装 | iOS Safari / Chrome / Edge 均可安装到桌面 |

## 数据存储

所有待办数据存储在 Gitee 仓库 `oooohcan/note` 的 `todos.json` 文件中，不触碰仓库其他文件。

## 技术架构

- 单文件 index.html，内嵌 HTML + CSS + JS
- **CSS** — 9 大模块分区：变量→布局→列表→控件→回收站→模态框→拖拽→加载→响应式
- **JS** — App 命名空间体系：
  - `h()` — 轻量 DOM 构建器，textContent 防 XSS
  - `DragDrop` — 独立拖拽类
  - `App.state` / `App.dom` — 状态与 DOM 缓存
  - `App.api*` — Gitee API v5 封装
  - `App.tree*` — 递归树操作（查找、删除、排序、移动）
  - `App.render*` — 渲染引擎
  - `App.action*` — 业务操作
  - `App.clickRoutes` — 事件路由表
- Gitee API v5 读写文件，base64 编码，SHA 冲突重试
- localStorage 存储访问令牌
