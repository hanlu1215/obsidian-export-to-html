# CODEBUDDY.md This file provides guidance to CodeBuddy when working with code in this repository.

## 常用命令

- `npm install` — 安装依赖（Node 16+，仓库 Volta 固定为 24.15.0）。
- `npm run dev` — esbuild 监听模式，增量编译 `src/main.ts` 到根目录 `main.js`（带 inline sourcemap，不压缩）。
- `npm run build` — 先 `tsc -noEmit -skipLibCheck` 做类型检查，再以 production 模式打包（压缩、无 sourcemap）。提交前/CI 使用的命令。
- `npm run lint` — 用 ESLint 9 + `neostandard`（`semi: true`, `ts: true`）检查整个项目；`main.js`、`node_modules` 被忽略。修 lint 问题直接跑这条即可，无单独的 `--fix` 脚本。
- `npm version <patch|minor|major>` — 触发 `version-bump.mjs`：把 `package.json` 版本写入 `manifest.json`，并向 `versions.json` 追加 `版本 -> minAppVersion` 映射，然后 `git add` 这两个文件。不要在 `manifest.json` 里手写版本号。
- 无测试框架，仓库中没有测试命令；验证靠类型检查 + lint + 在 Obsidian 中手动加载插件。

## 代码架构

这是一个 **Obsidian 桌面端插件**（`manifest.json` 中 `isDesktopOnly: true`、id 为 `export-to-html-han`，显示名 `Export to HTML Han`），作用是把当前笔记渲染成 HTML，复制到剪贴板或下载为 `.html` 文件。整体只有四个源文件，数据流是单向且很短：取 Markdown 源文本 → 借 Obsidian 的渲染器渲染到离屏 DOM → 后处理 DOM（删按钮、把媒体地址还原为 `file:///` 绝对路径）→ 要么塞进 `ClipboardItem`，要么包一层 HTML 外壳后触发下载。

**入口与主流程（`src/main.ts`）**

`ExportToHtmlPlugin extends Plugin`，`onload()` 里注册两条命令，均使用 `editorCallback`，且第一条语句都是 `if (!('data' in view)) return;`。这个守卫是必需的：`MarkdownView | MarkdownFileInfo` 联合类型里只有 `MarkdownView` 带 `data` 字段（即完整 Markdown 源文本），另一分支没有，因此命令在非编辑态/无数据时静默返回。修改命令逻辑时请保留该守卫，否则会得到 `undefined` 输入。

- `copy-to-clipboard-as-html`：渲染后同时构造 `text/plain` 与 `text/html` 两个 `Blob`，放进一个 `ClipboardItem`（键名用 `blob.type`），通过 `window.navigator.clipboard.write([...])` 写入。同时提供纯文本与 HTML 两种 flavor 是刻意为之——粘贴到 Word/邮件时保留样式，粘贴到普通编辑器时退化为文本。失败时捕获异常并 `new Notice(...)`（`main.ts` 顶部有 `eslint-disable no-new`，因为 Notice 只做副作用）。
- `download-as-html`：渲染结果被嵌入一个手写 HTML 文档模板（`<!doctype html>` + `<head>` 内联 `<style>` + `<body class="markdown-body">`），再交给 `downloadBlob()`。

注意 `main.ts` 里还有一段 `customCss` 模板字符串（限制 `.markdown-body` 宽度、居中、内边距及窄屏适配），它**只在下载路径中注入**；复制到剪贴板的 HTML 不带任何 `<style>`，只依赖粘贴目标已有的样式。改样式时要明确是在改哪一路径。

**渲染器（`src/HtmlRenderer.ts`）**

核心是 `render(markdownContent: string): Promise<string>`：

1. `activeDocument.body.createDiv()` 创建一个不挂载到笔记视图的临时容器（用 `activeDocument`/`activeWindow` 全局，而非 `window`，符合 Obsidian 多窗口约定）。
2. `MarkdownRenderer.render(this.app, markdownContent, el, '.', this.component)` —— 复用 Obsidian 官方渲染管线，因此 Obsidian 支持的语法（callout、内部链接、代码块高亮等）自动被支持。第 4 个参数 `'.'` 是解析相对链接的源路径；第 5 个参数是 `Component`，用于让渲染出的动态组件随插件生命周期正确卸载（构造时传入的是插件实例本身）。
3. 后处理：删除所有 `.copy-code-button`（Obsidian 注入的复制按钮，导出无意义）。
4. 媒体地址还原：遍历 `el.querySelectorAll('img, video, source')`，把 `src` 交给 `toFileUrl()` 改成 `file:///` 绝对路径；其中 `<video>` 会被强制写入 `VIDEO_STYLE`（`width:80%; height:auto; max-width:1920px; max-height:1080px;`），覆盖 Obsidian 自带的 `width:100%` 内联样式。最后同步返回 `el.innerHTML`。

**媒体地址还原（`toFileUrl()`，本仓库相对上游插件的主要差异点）**：`MarkdownRenderer` 产出的 `src` 是形如 `app://<vaultId>/G:\...\eval.mp4?<mtime>` 的私有协议 URL，在导出的 HTML 里既不能显示图片也不能播放视频。因此这里剥掉 `app://<vaultId>/` 前缀 → 去掉 `?` 时间戳与 `#` 片段 → `decodeURIComponent` → 把反斜杠换成正斜杠并去掉多余前导斜杠 → `encodeURI`，最终输出 `file:///G:/.../eval.mp4`，与 Markdown 源文件中的 `file:///` 写法一致。

两条硬性约定，改这块代码时务必保持：
- **图片不再转 base64**（上游插件的做法）。本分支刻意改为绝对路径引用，因为 base64 会让导出体积暴涨、大图/多图时卡顿；剪贴板与下载两条命令现在行为一致，都用绝对路径。
- **只改写本地资源**：`toFileUrl()` 对不以 `app://` 或 `file://` 开头的 `src` 返回 `null`，调用方直接跳过，因此外链图片（`https://`）、`data:`/`blob:` 等保持原样，不会被错误地拼成 `file:///https://...`。

选择 `img, video, source` 是按标签而非按 Markdown 写法过滤的，因此 `![[a.png]]`、`![[a.png|900]]`、`![](a.png)`、`<img>`、`<video>`、`<source>` 走的是同一条路径；`|900` 这类尺寸语法由 Obsidian 渲染成 `width`/`style` 属性，后处理用 `addStyle()` **追加**而不是覆盖样式，所以宽高指定会被保留。

**图片/视频居中并独占一段**：`![[x.png]]`、`![[x.mp4]]` 被 Obsidian 包在 `span.internal-embed.media-embed` 里，只给 `img`/`video` 加样式不会让它独占一行，因此 `getEmbedWrapper()` 会先把这个 span 变成 `display:block; text-align:center;`，没有该 span 时（如 `![](x.png)`）才直接给元素本身加 `display:block; margin:auto`。样式一律通过 `addStyle()` 追加在已有 `style` 之后，保证覆盖 Obsidian 默认值（例如视频的 `width:100%`）。`VIDEO_STYLE` / `IMAGE_STYLE` / `EMBED_WRAPPER_STYLE` 三个常量定义了这些展示规则，改排版只需动这里。

**工具（`src/utils.ts`）**

- `downloadBlob(blob, filename)`：用 `URL.createObjectURL` + 临时 `<a download>` + `a.click()` 触发下载，并在 click 后 150ms `revokeObjectURL` 且移除监听器。`createEl` 是 Obsidian 全局。
- `arrayBufferToBase64(buffer)`：手工循环 + `btoa`。目前 `HtmlRenderer` 不再做 base64 内联，**没有任何调用方**（Obsidian API 自己也提供同名函数），保留它只是为了将来需要内联时使用。

**样式（`src/css.ts`）**

默认导出一个巨大的模板字符串，内容是第三方 `github-markdown-css`（README 中已注明来源），约 1550 行。它是**生成/拷贝产物，不要手工格式化或逐条编辑**；需要更新时应整体替换。

**构建与工程约定**

- `esbuild.config.mjs`：入口固定 `src/main.ts`，`bundle: true`，输出 `format: 'cjs'`（Obsidian 运行时要求 CommonJS），target `es2018`，输出到仓库根目录 `main.js`。`obsidian`、`electron`、`@codemirror/*`、`@lezer/*` 及所有 Node 内置模块都标记为 `external`（运行时由宿主提供，切勿把它们打进 bundle）。production 模式会 `minify` 且关闭 sourcemap，构建完直接 `process.exit(0)`；dev 模式进入 `context.watch()`。
- `tsconfig.json`：`module: nodenext`，因此 **`src` 内部互相 import 必须写 `.js` 后缀**（如 `import css from './css.js'`），即使源文件是 `.ts`。新增模块时也照此书写，否则类型检查失败。开启了 `strictNullChecks`、`noImplicitAny`、`isolatedModules`。
- `main.js` 在 `.gitignore` 中，构建产物不入库；仓库里存在的 `main.js` 只是本地构建结果。
- 风格：2 空格缩进、LF、分号（`.editorconfig` + neostandard）。
- 发布：打 tag 推送会触发 `.github/workflows/release.yml`，它 `npm install && npm run build`，对 `main.js` 生成 artifact attestation，然后用 `gh release create --draft` 上传 `main.js` 与 `manifest.json`。所以发布产物只含这两个文件，不需要提交 `main.js`。

**移动端**：README 明确说明 Obsidian 移动应用尚不支持，开发时无需考虑移动端 API 兼容。
