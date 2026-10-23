# GitHub Pages 静态 Markdown 编辑器实施计划

## 本轮交付与边界

本轮只完成计划和分支，不实现应用、不安装依赖、不构建、不发布。

项目位置：`E:/CodeX/mdeditor/pages-version`。
独立仓库基线：`main`；开发分支：`feature/github-pages`。
上级目录目前没有 `.git`，因此在子目录建立独立仓库，避免改动现有网页和 Electron 版本。

后续完整交付：源码、可运行的 `dist/`、验收测试、GitHub Actions 和部署 README。

## 目标与技术取舍

- Vite + TypeScript + React；开启 TypeScript 严格检查。
- 纯静态 HTML / CSS / JavaScript，在访问者浏览器运行；没有后端、运行期服务或自定义构建密钥。
- 支持 `https://user.github.io/<repo>/`，也支持独立根站点。
- Notepad-- 风格：紧凑菜单 / 工具栏、文件侧栏、源代码与预览双栏、底部状态栏；支持浅色和深色主题。
- CodeMirror 6 编辑器，复用现有语言包。复用并迁移已有 Markdown 渲染、Lisp 词法和格式化工具，避免复制整套旧应用。
- KaTeX、样式和字体均随构建本地打包；代码高亮只注册要求的语言，避免加载所有语言。
- PDF 使用浏览器打印渲染后的预览，不引入 html2canvas、html2pdf 或 Electron。
- IndexedDB 使用浏览器原生 API 的小型 Promise 封装；ZIP 按需加载一个成熟库，不手写压缩格式。
- File System Access API 属可选增强，核心验收完成后再实施；不成为 Firefox / Safari 的使用前提。

## 计划目录

```text
.github/workflows/pages.yml
package.json / package-lock.json
tsconfig.json / vite.config.ts / index.html
src/
  main.tsx / App.tsx / styles.css
  model.ts
  storage/db.ts / storage/backup.ts
  state/useWorkspace.ts
  editor/Editor.tsx / editor/brackets.ts / editor/format.ts
  preview/markdown.ts / preview/Preview.tsx / preview/print.ts
  sidebar/Sidebar.tsx
  toolbar/Toolbar.tsx / toolbar/TableDialog.tsx
tests/
  storage.test.ts / saving.test.ts / backup.test.ts / brackets.test.ts
  acceptance.spec.ts
README.md
```

这是后续实现目录设计，当前并未创建应用源码或工作流。

## 阶段 1：静态工程与路径验证

1. 在开发分支建立独立 Vite + TypeScript + React 工程和锁文件。
2. `npm run build` 执行类型检查和生产构建，输出本仓库的 `dist/`。
3. Vite `base` 由公开的 `PAGES_BASE` 配置决定，项目站点为 `/<repo>/`，用户根站点为 `/`。不使用根绝对路径 `/assets/...`。
4. 字体、图标和静态资源通过模块导入或 `import.meta.env.BASE_URL` 定位；不使用 CDN。
5. 单页面无需 history 路由；如需保存选中文档到 URL，使用 hash，避免静态站点刷新时的路由 404。
6. 在本地静态服务器的 `/test-repo/` 子路径运行构建产物，确认脚本、样式、字体和懒加载模块没有 404。

完成标志：干净安装后可构建；根路径和子路径均能启动空壳页面。

## 阶段 2：IndexedDB 数据与保存语义

### 数据模型

- `files`：`id`、`name`、`content`、`language`、`folderId`、`tags`、创建 / 修改时间、修订号。
- `folders`：`id`、`name`、`parentId`、折叠状态；禁止循环父子关系。
- `settings`：主题、分栏比例、滚动同步、选中文件、备份格式版本等。
- schema 使用 IndexedDB 数据库版本管理；后续升级必须有可测试的迁移。
- IndexedDB 命名空间包含部署 base，避免同一个 `user.github.io` 下多个仓库误用同一数据库。
- 不使用 localStorage 保存文档或文件树。

### 保存规则

1. 首次加载从 IndexedDB 异步恢复；加载完成前禁止编辑，读取失败明确报错，不以空工作区覆盖原数据。
2. 文本修改先进入内存草稿，记录 dirty 文档集合与修订号。
3. 只有正文编辑重置一次性 5000ms 定时器；连续输入始终重新计时，停止输入满 5 秒再保存 dirty 集合。
4. 不使用定时轮询；首次打开、无修改停留和保存成功后继续空闲，都不得重复写入正文。
5. Save 按钮和 Ctrl+S / Cmd+S 立即提交 dirty 文档，并取消相应的待执行自动保存。
6. 只有 IndexedDB 事务 `complete` 后显示保存成功。写入失败保留 dirty 状态、保留草稿，并提供重试 / 导出备份入口。
7. 保存时记录提交的修订号；保存过程中又有输入时，旧事务成功不能把新内容标记为已保存。
8. 文件夹、拖放、标签和设置变化立即持久化；元数据更新只修改已存记录的指定字段，不把尚未满 5 秒的正文草稿顺带写入。
9. 切换文件不丢失内存草稿；未保存离开页面时提示。不能依赖卸载事件中的异步写库保证保存。
10. 状态显示“未保存 / 已保存到浏览器 / HH:MM 已自动保存 / 保存失败”。
11. 多标签页通过 BroadcastChannel 提示外部修改；已有本地草稿时不静默覆盖，提供重新载入或先备份选项。

验收中的“5 秒”指前台环境的 5000ms debounce 规则；浏览器后台节流或系统繁忙可能延迟实际执行，不承诺实时调度。

完成标志：正文不依赖 localStorage，手动保存、自动保存、写入错误与重启恢复测试通过。

## 阶段 3：编辑、预览与组织

1. 完成可调整宽度的源代码 / 预览双栏、主题和编辑器滚动。
2. Markdown 使用 CommonMark 兼容配置，启用表格、任务列表、删除线、自动链接和脚注；原始 HTML 禁用，渲染输出仍经 DOMPurify 清洗。
3. 用固定样例验证 `$...$` 与 `$$...$$`；KaTeX `trust: false`，禁用危险指令。
4. 预览代码块覆盖 JavaScript、Java、Python、C、C++、Lisp；未知语言保留可读的转义代码。
5. 编辑器提供 Markdown 与代码语言模式，语法主题跟随界面切换。
6. 彩虹括号基于语法 token / 语法树忽略字符串和注释中的括号，对 `()`、`[]`、`{}` 的深度循环着色，未匹配与错配括号单独标红。
7. Markdown 中只在支持的 fenced code 区域启用代码括号检查，避免把链接或普通文字里的符号误判为程序错误。
8. Lisp 使用现有词法能力并补充对应括号扫描；明确不把 JSX、模板字符串或正则中的字符盲目当括号。
9. 侧栏支持文件 / 文件夹管理、拖放移动、标签维护与标签过滤。
10. 全文搜索涵盖标题和内容，显示摘要及命中数；点击结果打开文件并将编辑器光标滚动到命中位置，可跳转下一个匹配。
11. 格式化工具栏支持粗体、斜体、删除线、标题、链接、图片、行内 / 块代码、引用、列表、任务列表、分隔线和表格。
12. 表格对话框选择行列数，始终生成包含表头分隔行的有效 GFM 表格；操作作用于编辑器选区。

完成标志：所有要求的语法样例正确显示；拖放后刷新仍在目标文件夹；搜索能跳到正文命中。

## 阶段 4：备份与打印 PDF

### JSON 与 ZIP

- JSON 带 `format` 和 `version`，完整包含文件、文件夹、标签与设置。
- 导出基于当前内存快照，包含尚未自动保存的草稿，避免备份遗漏最新输入。
- 导入前校验结构、版本、重复 ID、字段类型、文件夹循环和引用，失败不改动现有数据。
- 导入展示文件数量并确认是否替换工作区；替换使用一个事务，失败可回滚。
- ZIP 导出保留文件夹结构和 `.md` 文件；附带清单文件记录 ID、标签、设置等，使本应用生成的 ZIP 也可完整恢复。
- ZIP 导入同时兼容普通 `.md` 文件合集；没有清单时只导入路径和正文，不声称恢复缺失元数据。
- 导入 ZIP 防止路径穿越、重复路径及超量解压；设置明确的总大小限制并提示用户。
- 页面说明：数据只在当前浏览器；清理站点数据、更换浏览器或无痕模式结束可能丢失，请定期导出。

### 浏览器打印

- 点击“打印 / 导出 PDF”后等待预览中的字体与图片准备，再调用 `window.print()`；资源失败给出可理解的提示。
- `@media print` 隐藏菜单、侧栏、编辑器、状态栏和对话框，仅打印渲染后的预览。
- 固定浅色打印背景，保留代码高亮和 KaTeX 字体；解除滚动容器高度 / overflow 限制。
- 设置页边距、代码块换行和适当的表格行 / 图片避断规则；超长块允许跨页，不能为避免分页丢失内容。
- 不把“打印窗口已打开”显示为“文件已保存”；实际 PDF 保存由浏览器 / 系统负责。

完成标志：JSON 全量往返一致；ZIP 可还原 `.md` 层次；长文档打印内容完整且含可读公式、高亮代码。

## 阶段 5：GitHub Pages 发布

### 地址与配置

| 部署地址 | Vite `base` | `package.json` 的可选 `homepage` |
| --- | --- | --- |
| `https://user.github.io/<repo>/` | `/<repo>/` | 对应完整项目网址 |
| `https://user.github.io/` | `/` | 对应根站点网址 |

Vite 实际读取 `base`；`homepage` 仅作为项目元信息，单独修改它不会修复 Vite 的资源路径。
Actions 从公开的仓库名推导项目 base；仓库名等于 `<user>.github.io` 时使用 `/`。本地构建可显式设置 `PAGES_BASE`。

### 工作流方案

1. `main` 更新或手动触发时：checkout → Node → `npm ci` → 类型检查 / 测试 → 生产构建。
2. 将 `dist/` 同步到独立的 `gh-pages` 产物分支，并包含 `.nojekyll`；该分支只存构建结果。
3. 同一工作流上传相同 `dist/` 为 Pages artifact，再使用 GitHub 官方 `deploy-pages` 发布。
4. 仓库 Settings → Pages 的 Source 选择 **GitHub Actions**。
5. 这是有意同时保留产物分支和官方发布步骤：使用 `GITHUB_TOKEN` 推送 `gh-pages` 本身不会触发分支式 Pages 构建，不能只推分支后声称已部署。
6. 仅使用 GitHub 内置 `GITHUB_TOKEN` / OIDC；不用 PAT、部署私钥或应用级构建 secrets。
7. 产物分支写入任务授予 `contents: write`；发布任务授予 `pages: write` 与 `id-token: write`，其余任务保持只读。
8. Pull request 只测试和构建，不发布；部署设置并发控制，防止旧提交覆盖新站点。
9. README 分别说明根站点、项目子路径、Pages 设置、数据备份、浏览器限制和打印步骤。

完成标志：仓库连接 GitHub 并实际部署后，用真实项目 URL 验证资源无 404；未部署前只能标记“本地子路径验收通过”。

## 可选阶段：真实本地文件夹

- 在支持且用户明确点击时调用 `showDirectoryPicker()`，按授权读写 `.md` 文件。
- 拒绝授权、取消选择或 API 不存在时继续使用 IndexedDB 与 JSON / ZIP 导入导出。
- 手动导出 / 保存到文件夹，不在未获授权时改写真实磁盘文件。
- 文件句柄权限可能需要重新授予，不能把浏览器存有句柄视为永久授权。

## 验收矩阵

| 编号 | 场景 | 验证方式 |
| --- | --- | --- |
| A1 | 根路径与 `/<repo>/` 资源完整 | 生产构建、子路径静态服务、请求错误断言和字体检查 |
| A2 | CommonMark / 所列 GFM 特性 | 固定语法样例检查 DOM；安全输入验证无脚本执行 |
| A3 | 行内 / 块公式和六类代码 | KaTeX 元素、字体加载与高亮 token 检查 |
| A4 | 彩虹括号 / 未匹配括号 | 嵌套、错配、跨行、字符串、注释和 fenced code 样例 |
| A5 | 5 秒自动保存 | 可控计时：4999ms 不写、5000ms 发起一次写入；输入重置计时；已保存空闲不写 |
| A6 | Save / Ctrl+S 与保存失败 | 立即提交、重复保存无多余正文写入、事务失败保留 dirty |
| A7 | 拖放持久化 | UI 拖动后刷新，验证 IndexedDB 与文件树位置 |
| A8 | 标签 / 搜索跳转 | title + content 搜索、标签组合过滤和编辑器选区位置 |
| A9 | JSON / ZIP 往返 | 文件正文、层次、标签、设置全量比较；损坏 / 非法导入不破坏数据 |
| A10 | 浏览器打印 PDF | Chromium 自动打印后提取 PDF 文字与截图；人工验证 Chrome / Firefox / Safari 打印预览 |
| A11 | 真实 GitHub Pages | 发布后检查子路径、网络资源与首次访问；明确区分本地通过和线上通过 |
| A12 | 浏览器兼容 / 数据风险 | Firefox / Safari 无目录 API 时导入导出可用；显示站点数据清除警告 |

## 建议提交顺序

1. `chore: scaffold typed static app and subpath build`
2. `feat: add IndexedDB workspace and explicit debounced saving`
3. `feat: add editor preview and bracket diagnostics`
4. `feat: add persistent tree tags and search navigation`
5. `feat: add backup import export and print styles`
6. `ci: publish gh-pages artifacts and deploy Pages`
7. `test: cover static-host acceptance and document delivery`

可选目录 API 单独提交，避免扩大核心版本范围。

## 参考资料

- [Vite 静态部署与 GitHub Pages base](https://vite.dev/guide/static-deploy.html)
- [GitHub Pages 发布源及 GITHUB_TOKEN 限制](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)
- [GitHub Pages 官方自定义工作流](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages)
- [MDN：IndexedDB 使用与事务](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB)
- [MDN：showDirectoryPicker 兼容性与权限](https://developer.mozilla.org/en-US/docs/Web/API/Window/showDirectoryPicker)
