# MD-- 1.0.8 GitHub Pages 静态版

本目录包含当前 1.0.8 前端的静态成品，可部署到：

```text
https://user.github.io/myMDeditor/
```

将 `user` 换成实际 GitHub 用户名。程序在访问者的浏览器内运行，部署端不需要安装 Node.js 或运行后端服务。

## 上传内容

把以下内容放到 GitHub 仓库 `myMDeditor` 的发布分支根目录，保持目录结构：

```text
index.html
favicon.svg
assets/
.nojekyll
README.md
```

`index.html` 必须直接位于发布目录根部，不要再套一层 `pages-version/` 或 `dist/`。`assets/` 包含脚本、样式、公式字体和按需加载的 PDF 导出组件，必须完整上传。`.nojekyll` 是空文件，也需要上传。

本地 `.git/` 用于保留仓库历史，不通过网页上传。现有 `.gitignore` 可保留；`PLAN.md` 可作为说明文档保留。本目录不包含源码、依赖目录、桌面 EXE 或用户数据。

## GitHub Pages 设置

1. 将上述静态文件上传到 `myMDeditor` 仓库的目标分支。
2. 打开仓库 **Settings → Pages**。
3. **Source** 选择 **Deploy from a branch**。
4. **Branch** 选择实际上传文件的分支，目录选择 **/(root)**，然后保存。
5. 等待 Pages 发布完成，再访问 `https://user.github.io/myMDeditor/`。

本地当前分支为 `feature/github-pages`；如果上传到 GitHub 的是 `main`，Pages 中就选择 `main`。本目录尚未配置远程仓库或执行线上发布。

资源使用相对路径，可在 `/myMDeditor/` 子路径下加载。未来更新时，从上级项目重新构建前端，再同步 `dist/` 的全部内容到本目录。

## 功能与数据

- 保留单行功能区、暗色编辑、文件夹与标签、折叠目录、可跳转 TOC、分栏伸缩、双向滚动同步及 Markdown/代码纯预览。
- PDF 沿用网页版逐页截图导出，提供进度和取消；不使用桌面版 Chromium 导出接口。
- 文档自动保存在当前浏览器的站点数据中，不会因为发布到 GitHub Pages 而自动同步到 GitHub、其他电脑或桌面版。清除站点数据会清除本地文档。
- 远程图片需要网络；当前版本没有保证离线重新打开的 Service Worker。
- 请通过 HTTP(S) 访问，不以双击 `index.html` 作为运行方式。

## 旧计划

[PLAN.md](./PLAN.md) 是此前独立重构版本的规划，保留供参考。其中 TypeScript、IndexedDB、浏览器打印等内容不代表本次静态成品已实现；本次交付以当前 1.0.8 网页版为准。
