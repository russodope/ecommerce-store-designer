# ecommerce-store-designer

**电商店铺首页设计 agent skill** — 先问出图尺寸，按品牌做差异化设计，生成可视化编辑器，导出**任意平台尺寸**的图片。

A platform-agnostic **skill for AI coding agents** to design e-commerce store homepages / decorations — it asks your export size first, produces brand-specific (non-templated) designs, generates a single-file visual editor, and exports images at whatever width your platform needs.

> 适用于淘宝/千牛、抖音小店、京东、拼多多、Shopify、独立站等**任意电商平台**；可配合 Claude Code、Cursor、Cline、Windsurf、Codex 等**各类 coding agent** 使用 —— 不绑定平台、不绑定某个 agent。
> Works for any store platform (Taobao, TikTok Shop, JD, Shopify, custom sites…) and with any coding agent (Claude Code, Cursor, Cline, Windsurf, Codex…).

---

## 它能做什么 / What it does

引导你的 coding agent 走一套工作流：

1. **调研需求** — 先问**出图尺寸**（平台 / 宽度，如 750 / 1080 / 1242 / 1440），再问品牌信息、风格、以及**这次主推 / 上架什么**（决定首页"卖什么、怎么排"）。
2. **研究驱动的差异化设计** — 由品牌的**艺术方向 (concept)** 同时驱动配色/字体**和版面结构**；方案里每个版块都要能回指 concept 某条；内置「防脊柱趋同」自检，避免同品类千篇一律。
3. **生成单文件 HTML 编辑器** — 左侧「添加模块」面板 + **拖拽到画布指定位置**、文字点击即改、图片点击即换、每个板块 ⚙ 调字号/行距/颜色/对齐/高度/图文比/镜像，一键导出**按你指定宽度**的整套 PNG。

Guides the agent through: research (incl. **export size**) → concept-driven differentiated design → a single-file HTML editor for in-browser editing and export at your chosen width.

---

## 核心特性 / Highlights

- 🎯 **先问尺寸，按平台出图**：内部统一按 **1440px 参考宽度**设计，导出时缩放到你指定的 `BRAND.exportWidth`（任意平台尺寸；高度 / 文件大小上限也按平台）。
- 🧩 **组件 kit + 概念组合**：13 个导出安全的版面组件（多种 hero / 左右分栏 / 等高网格 / 错落网格 / 横向 lookbook / 色卡条 / 规格大数字 / 不对称编辑式 / 巨字陈述 / 过渡 / 品牌故事…），由 concept **组合**出每个品牌专属骨架，**不是固定模板换皮**。
- ✏️ **可视化编辑器**：从左侧面板**拖拽插入**模块、逐板块排版/布局配置、文字与图片即点即改；所有编辑入口在**导出时自动隐藏**。
- 🌏 **国内可用 + 字体可靠**：脚本走 BootCDN、字体走 `fonts.googleapis.cn`；并用 **FontFace** 把字体烤进导出图。
- 📐 **导出安全**：图片区一律 `background-image`、禁用高级 SVG、模块高度 200–2000px 超限拆图、压缩到 ≤2MB（可按平台调）。

---

## 工作原理 / How it works

导出的不是网页，而是**图片** —— 因为大多数电商平台的移动端装修是「模块 + 图片」拼装，不吃自定义 HTML。skill 生成的是一个**本地设计工具**（单文件 HTML）：在浏览器里设计 → 导出图片 → 上传到你店铺的对应模块。

The output is **images**, not a web page — most store platforms assemble mobile homepages from "modules + images" and don't accept custom HTML. The generated single-file HTML is a **local design tool**: design in the browser, export images, upload to your store's modules.

---

## 文件结构 / Files

| 文件 | 作用 |
|---|---|
| `SKILL.md` | **决策层**：工作流、调研问题、concept→结构、组件 kit 用法、差异化质检清单。 |
| `references/code-patterns.md` | **代码层（经过验证的实现）**：`BRAND` 配置、固定机器（`makeUploadable` / `renderMod`+FontFace / 导出缩放 / `expZip` / `expLong`）、13 组件 kit、编辑器外壳（左侧添加模块 + 拖拽 + 配置面板）。 |
| `references/industry-*.md` | 行业配色 / 版式参考（服装、美妆香氛、家居食品、数码运动母婴）。 |

---

## 用法 / Usage

本质是一套 **markdown 指令 + 代码模板**，任何能读取本地文件 / 上下文的 coding agent 都能用：

- **Claude Code / Claude Desktop**：下载 [`ecommerce-store-designer.skill`](./ecommerce-store-designer.skill)（或 [Releases](../../releases)）导入 skills；或把本目录放进 skills 目录。
- **Cursor / Cline / Windsurf / Codex 等**：把 `SKILL.md` + `references/` 作为 rules / context / 附件喂给 agent，让它按 `SKILL.md` 的流程执行即可。

It's just **markdown instructions + code templates** — usable by any agent that can read local files/context. For Claude, import the `.skill`; for other agents, feed `SKILL.md` + `references/` as rules/context.

> `.skill` 文件本质是 ZIP，改后缀为 `.zip` 解压即可读源码。 / The `.skill` is just a ZIP — rename to `.zip` to read the source.

---

## License

MIT
