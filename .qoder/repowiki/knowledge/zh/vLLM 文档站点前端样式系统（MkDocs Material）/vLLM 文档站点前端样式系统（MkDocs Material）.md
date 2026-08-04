---
kind: frontend_style
name: vLLM 文档站点前端样式系统（MkDocs Material）
category: frontend_style
scope:
    - '**'
source_files:
    - mkdocs.yaml
    - docs/mkdocs/stylesheets/extra.css
---

vLLM 项目不包含传统意义上的前端 UI 应用（如 Web 页面、组件库或 CSS 框架），其前端样式仅体现在基于 MkDocs Material 主题的官方文档站点中。该样式系统完全围绕文档展示需求构建，不涉及用户交互界面。

**系统与工具**
- 使用 MkDocs + Material for MkDocs 主题作为文档站点生成器
- 通过 `mkdocs.yaml` 配置主题、调色板、功能开关与插件
- 自定义 CSS 文件位于 `docs/mkdocs/stylesheets/extra.css`，用于覆盖默认样式
- 支持明暗双主题（default/slate），通过 `data-md-color-scheme` 属性切换

**核心文件与配置**
- `mkdocs.yaml`：定义主题名称为 `material`，配置 logo、favicon、调色板（自动/浅色/深色三种模式）、功能特性（编辑按钮、代码复制、标签页、导航等）以及插件体系（search、minify、mkdocstrings、glightbox 等）
- `docs/mkdocs/stylesheets/extra.css`：包含约 192 行自定义样式，主要覆盖外部链接图标、公告横幅、自定义 admonition（announcement/important/code/console）、编辑按钮、Slack/Forum 按钮、Logo 明暗切换、内容标签页边框等
- `docs/mkdocs/javascript/`：存放辅助 JavaScript 文件（run_llm_widget.js、mathjax.js、edit_and_feedback.js、slack_and_forum.js）
- `docs/mkdocs/overrides/`：自定义模板覆盖目录
- `docs/mkdocs/hooks/`：文档生成钩子（移除公告、生成示例、解析 argparse、指标生成、URL 方案处理、自动引用）

**架构与约定**
- 样式采用纯 CSS 覆盖方式，未引入任何 CSS 框架（无 Tailwind、Bootstrap 等）
- 使用 Material Design 的 CSS 变量（如 `--md-warning-bg-color`、`--md-default-fg-color`、`--md-accent-fg-color`）保持主题一致性
- 通过内联 SVG data URI 实现图标资源，避免额外 HTTP 请求
- 响应式行为依赖 Material 主题内置媒体查询，未在项目中自定义断点
- 所有样式均服务于文档阅读体验，无用户输入交互逻辑

**约束与规范**
- 样式文件仅存在于 `docs/mkdocs/stylesheets/extra.css`，禁止在其他位置添加 CSS
- 主题配置集中在 `mkdocs.yaml`，新增样式需同步考虑明暗双主题兼容性
- 自定义 admonition 类型需同时定义 CSS 变量、边框颜色、背景色和图标
- 生成的文档页面（examples/api 等）通过钩子排除编辑按钮显示
- 外部链接自动添加外链图标，但 localhost/127.0.0.1/docs.vllm.ai 等内部链接除外