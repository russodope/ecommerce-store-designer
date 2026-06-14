---
name: ecommerce-store-designer
description: |
  Design and generate e-commerce store homepage / decoration module images for any online-store platform. Use this skill whenever a user wants to:
  - Design an online store homepage or decoration (店铺装修 / 首页装修 / 店铺首页)
  - Create store banner images, series showcases, category navigation, or brand story sections
  - Build a modular store-design tool in HTML that exports module images at a user-specified width
  - Design e-commerce product showcase / homepage pages for any platform
  - Create a customizable store template for different brands
  Trigger on "店铺装修", "店铺首页", "首页设计", "banner", "详情页", "电商装修", "store/shop decoration", or "store homepage" alongside design or image-generation requests.
  ⚠️ The skill FIRST asks the user for the target image size (platform / 出图宽度), then designs at a 1440px internal reference and exports at the user's width.
---

# 电商店铺首页设计器

> **阅读顺序**：本文件是决策层（何时做什么），代码实现细节在 `references/code-patterns.md`。
> 生成代码前**必须**先读 `references/code-patterns.md`。

---

## 第一部分：工作流程（严格按顺序）

### ⚠️ 强制规则

1. **无论是否认识该品牌，必须先完成第一轮问答再生成代码。** 即使用户提到知名品牌，也禁止用训练数据替代用户输入。（**例外**：若用户已一次性给齐品牌+风格+销售/上架简报，可免问答直接进第二步。）
2. **先决策，后编码。** 方案摘要确认前不写任何 HTML/JS。
3. **先读 code-patterns.md，后写代码。** 生成 HTML 编辑器时，必须先查阅 `references/code-patterns.md` 获取经过验证的代码模板。

---

### 第一步：收集品牌信息

一次性发出所有问题（可用 ask_user_input 工具辅助收集选择题部分）：

```
我来帮你设计电商店铺首页，先了解几个基本信息：

1. **出图尺寸**（最重要）：你的店铺在哪个平台、需要多大尺寸的图？给我**出图宽度（px）**就行——不确定我按常见值推荐（手机端常见 750 / 1080 / 1242 / 1440）。单张图有没有**最大高度 / 文件大小**限制？
2. 品牌名称（中英文）
3. 卖什么品类？
4. 品牌风格关键词（选 2-3 个）：
   极简 / 复古 / 中式 / 活泼 / 高级感 / 自然 / 暗黑 / 甜美 / 工装 / 清新 / 科技
5. 有哪些系列或分类？（首页按系列或品类组织）
6. 有没有品牌色偏好？（没有的话我来推荐）
7. 有没有上传产品图/品牌素材？（有的话请一起发）

（销售/上架——决定首页"卖什么、怎么排"）
8. 这次首页**主推什么**？主打 1 个系列 / 多系列并列 / 按品类铺
9. 这次**上架或要展示哪些**？（具体系列/品类、大概几款）有 **新品 / 爆款 / 活动** 要突出吗？
10. 主推款要不要 **C 位放大**？有没有必须靠前的卖点（功效 / 材质 / 认证 / 价格）？
```

**关于用户上传图片的处理**：
- 如果用户上传了产品图/品牌LOGO，在方案中说明如何使用它们
- 如果用户没有上传图片，所有图片区使用占位色块 + 上传入口，不要编造假图
- 永远不要用外部图片URL作为占位图（不可控且可能失效）

---

### 第二步：调研 → 艺术方向(concept) → 方案摘要

> ⭐ **这是反"千篇一律"的核心**：先得出**品牌专属的艺术方向(concept)**，再让 concept **同时驱动配色/字体 _和_ 版面结构**。结构由 concept 决定，**不再按"系列数量"套固定骨架**。

**2a. 调研出艺术方向（深度自适应，由你判断）**
- 有上传素材/参考图 → **先逐张看过**，提取调性（色/质感/构图/留白/影像风格）。**排除截图类素材**（别家页面/装修工具界面的截屏不是产品/品牌图，误用会把界面元素烤进成品）。
- 品牌可识别且信息不足 → 可用 `WebSearch`/`WebFetch` 查品牌官网 + 1~2 竞品 + 品类视觉惯例。**联网失败兜底**：回退到第一步问答 + 风格关键词，并告知用户"未联网，基于你提供的信息设计"。不得用训练数据替代用户输入。
- 产出 **concept**（结构化，内部思考）：
  - 布局哲学（编辑杂志 / 全图叙事 / 网格目录 / 极限留白 / 不对称拼贴 / 建筑规整 …）
  - 签名视觉手法、字体搭配理由、配色逻辑（用 `industry-*.md` 配方做**种子**再按素材调校）、影像方向
  - **为该品牌定制的 2~3 个版块构想**
  - **导出安全边界**：签名手法必须用 html2canvas 安全手段实现（纯 CSS / `background-image` / 简单 `path`），想不出安全实现就降级。

**2b. concept + 上架计划 → 结构**（结构同时服务"美学概念"和"这次要卖/主推什么"，**不是系列数量**）
```
concept(布局哲学) + 上架计划(主推/品类/新品爆款)
   ├── 配色种子(industry-*.md) → 按素材调校 → BRAND 色值
   ├── 字体搭配 → BRAND.fontCN/fontEN/.cn URL
   └── 版面结构 ← 由 concept + 上架计划共同决定：
        • 主打单系列→hero/叙事重；多品类→分类导航网格；有新品→新品专区；主推款→C位放大(用 specBlock/featureSplit 突出卖点)
        • 从第三部分 kit 选哪些组件、什么顺序、各组件 cfg
        • ≥2 个板块按 concept 定制（结构上不同于默认，不是改个名）
        • 背景色/高度按第四部分节奏排
```

**输出给用户的方案摘要**（确认后才进第三步）：

```
## 首页方案
**品牌**：[名称]/[英文]　**装修端**：手机端(1440px)
**艺术方向(concept)**：[布局哲学一句话] + [签名手法]
**色彩**：主[值] / 交替[值] / 文字[值] / 点缀[值]
**字体**：[中文] + [英文]

**版块结构推导表**（每个板块必须能回指 concept 或上架计划某条；填"好看/平衡"等通用词=未驱动，打回重做）
| # | 版块(组件) | 结构特征(图文比/密度/栏数/对称/对齐) | 承载什么货/目标 | 由 concept/上架计划哪条推导 | 定制? |
|---|---|---|---|---|---|
| 1 | [组件] | [如 居中浮图·大留白] | [主推系列/新品/卖点] | [concept 或上架某条] | 否 |
| 3 | [组件] | [如 非对称·右下文字块] | [次推/品类导航] | [concept 或上架某条] | 是 |

**视觉串联**：[贯穿全页的统一元素]
确认后我就生成，有要调整的告诉我。
```

---

### 第三步：生成 HTML 编辑器

用户确认后：
1. **先读 `references/code-patterns.md`**，获取所有代码模板
2. 生成完整的单文件 HTML

> ⭐ **核心原则：所有内容都能由用户自己编辑，且编辑入口必须【始终可见】。** 文字点击即改（hover 有提示）；图片**即使已填**也保留"⤓ 换图"角标 + hover"点击替换"遮罩；每个板块右上有 ⚙ 调版式（高度/字号/颜色/对齐/图文比/镜像）；编辑器顶部放一条操作提示条。所有这些入口在**导出时自动隐藏**（见 code-patterns.md）。用户不该以为这是改不了的成品。

**HTML 编辑器必须包含**：

| 功能 | 说明 |
|---|---|
| 左侧"添加模块"面板 | 列出本品牌各模块（图标+名称+×高度），**点击即添加**；底部放 打包ZIP / 合并长图 按钮 + 尺寸说明（见 code-patterns.md"编辑器外壳"）|
| 中间画布 | 1440px 缩放到 360px 预览；**干净白画布，不要黑色手机外框、不要限高**（整页可滚动、完整显示）；画布初始已铺好按 concept 推荐的差异化组合，每模块右上 ⚙↑↓✕ |
| 文字编辑 | 所有文字 `contenteditable`，点击直接改 |
| 图片替换 | 图片区有明确的"点击上传"视觉提示，点击可替换 |
| 导出按钮 | 单模块导出 / ZIP打包 / 合并长图 |
| 尺寸标注 | 每个模块右上角小字标注实际导出尺寸（如 1440×800） |
| 逐板块配置面板 | 选中模块弹出面板：板块高度/背景色/主文字色/字号缩放/对齐/图文比/镜像；直接改 DOM、实时生效、文字与图不丢、导出自动带上（见 code-patterns.md 第 5 节"配置面板"）|

---

## 第二部分：出图尺寸（按用户提供的尺寸）

> 不同平台/需求的图片尺寸不同——**第一步必须先问用户出图宽度**，再按它出图。内部统一在 **1440px 参考宽度**下设计，导出时缩放到用户的目标宽度（`BRAND.exportWidth`）。这样设计系统稳定、出图尺寸又能适配任意平台。

### 核心规格

| 参数 | 值 |
|---|---|
| **出图宽度** | **用户指定**（`BRAND.exportWidth`；用户不确定时默认 1440） |
| 内部设计参考宽度 | **1440px**（固定）；预览 360px（缩放 0.25）；导出时缩放到出图宽度 |
| **单模块高度范围** | **200–2000px**（按平台；用户给了上限就遵守，超出拆成多张上下拼接） |
| 单张文件大小 | 默认 **≤ 2MB**（按平台；用户给了上限就用它） |
| 图片格式 | JPG / PNG |

### 常见平台出图宽度参考（仅参考，以用户实际平台为准）

| 场景 | 常见宽度 |
|---|---|
| 手机端店铺首页 | 750 / 1080 / 1242 |
| 高清 / 2x 出图 | 1440 / 1500 |
| PC 端首页通栏 | 1920 |

### 重要约束

- **出图宽度 = 用户指定**；内部按 1440 设计，导出时缩放到该宽度（见 code-patterns.md `renderMod`）。
- 单模块高度超过平台上限（默认 2000px）必须**拆分为多张图上下拼接**。
- 文件超限需压缩（`compressCanvas` 默认压到 ≤2MB，可按用户上限调整）。

---

## 第三部分：组件 kit（按概念组合，不是固定菜单）

`references/code-patterns.md` 第 5 节内置一套**导出安全的组件 kit**：
`heroFull / heroMinimal / heroSplit / featureSplit / editorialAsym / grid / gridMasonry / statement / specBlock / swatchStrip / lookbookRow / transition / story`。

**用法**：由 concept 决定**选哪些 + 什么顺序 + 各自 cfg**，搭出品牌专属骨架。差异化来自"组合 × 参数化"，不是"换肤"。每个组件 = `fn(cfg)`，cfg 是单一数据源（文案/图片/版式都在内）。

**别把它当"构图菜单"四选一**——组合时在意这些**设计轴**（连续取值）：

| 设计轴 | 两端 |
|---|---|
| 图文权重 | 全图无字 ←→ 全字无图 |
| 信息密度 | 极限留白 ←→ 密集网格 |
| 对称 | 居中对称 ←→ 强不对称/重叠 |
| 边界 | 硬边切割 ←→ 柔和留白过渡 |
| 节奏 | 等距规整 ←→ 错落跳栏 |

**反收敛硬规则**：
- 结构（板块数量/顺序）由 concept 定，**不照搬第四部分范例**。
- **≥2 个板块**按概念定制（结构上不同于默认，不是改名）。
- 可扩充新组件，但必须过 `code-patterns.md` 第 6 节"导出安全清单"。
- **防脊柱趋同（窄语汇品类必看）**：有些品类有强默认母题（极简女装≈`heroMinimal`+不对称镜像；数码≈`heroSplit`+`specBlock`），同风格的两家极易塌成同一根**有序脊柱**。判据看**版块的有序序列**，不是"用了哪些组件"（零件重合 ≠ 模板雷同）。若你的序列与该品类明显默认骨架、或你给同类做过的上一版**有序重合度高**，必须主动打破其一：**换 hero 原型 / 重排顺序 / 替换 1-2 个承重组件 / 改签名母题**。目标——同品类同风格的两家也要不同脊柱。

---

## 第四部分：模块组合原则

### 节奏规则（必须遵守）

```
规则1：相邻模块不同构图类型
  ✓ 全出血 → 左右分栏 → 网格 → 全出血
  ✗ 全出血 → 全出血 → 全出血

规则2：背景色交替（2-3个值轮换）
  ✓ 浅色 → 深色 → 浅色 → 中间色
  ✗ 浅色 → 浅色 → 浅色

规则3：有过渡条分隔不同主题区块
  系列1 → [过渡条] → 系列2 → [过渡条] → 分类导航

规则4：首尾呼应
  - 首模块（Banner）视觉冲击力最强
  - 尾模块（品牌故事）用深色收尾，形成闭合感
```

### 多样化范例（⚠️ 示范"从概念推出结构"的过程，**禁止照抄**到真实品牌；你的结构必须从你自己的 concept 现推）

这 3 个骨架**刻意互不相同**——证明"Hero→过渡→系列→网格→故事"不是唯一解。已实测：独立盲评判为"3 套不同设计系统"，且同品类两品牌(珠宝)骨架差异最大。

**① 功能主义/非对称（工装·户外）**
`heroFull(全出血暗调)` → `transition` → `featureSplit(左图右字)` → **`specBlock(超大数字规格)`** → **`gridMasonry(错落)`** → **`statement(巨字)`** → `story(深色)`

**② editorial/极限留白（设计师珠宝·小众香氛）**
**`heroMinimal(居中浮图大留白)`** → **`editorialAsym×2(左右镜像·角落文字)`** → **`lookbookRow(横向)`** → `grid` → `story`　**（全程无过渡条）**

**③ 建筑/规整网格（几何珠宝·数码）**
**`heroSplit(左字右图)`** → **`swatchStrip(材质色卡条)`** → `grid(网格优先)` → `featureSplit(右图)` → `statement` → **`transition(大字水印)`** → `story`

> ②③ 同为珠宝，组件集合 Jaccard≈0.2。**换品牌就换骨架，不是换色。**

### 视觉串联（必须选一种贯穿全页）

| 手法 | 实现方式 | 适合风格 |
|---|---|---|
| 大字水印 | 品牌名超大字叠在背景，opacity 3-5% | 极简/高级 |
| 编号节奏 | 01 / 02 / 03 统一字体位置 | 现代/工装 |
| 细竖线 | 左侧固定位置 1px 虚线 + 节点装饰 | 日式/简约 |
| 品牌图案 | SVG 图案沿弧线分布，opacity 15-20% | 中式/复古 |

---

## 第五部分：色彩与字体

### 背景色策略（按风格选择）

| 风格 | 浅色背景 | 深色背景 | 备注 |
|---|---|---|---|
| 极简 / 现代 | `#F8F6F3` | `#EEEAE4` | 边界感消失 |
| 中式 / 手工艺 | `#EAE0CC` | `#DDD2BA` | 暖亚麻质感 |
| 高级 / 暗黑 | `#2A2520` | `#1C1814` | 深墨主色 |
| 活泼 / 清新 | 品牌色浅色 | 白色卡片 | 色彩饱和 |
| 科技 / 数码 | `#F5F5F7` | `#1A1A1A` | 冷灰对比 |
| 母婴 / 可爱 | `#FFFEF9` | `#FFE8D6` | 明亮暖白 |

> 更详细的行业配色方案见 `references/industry-*.md` 文件。

### 字体预设（根据品牌风格选择）

| 风格 | 中文字体 | 英文字体 | Google Fonts 参数（拼在 `…/css2?family=` 之后） |
|---|---|---|---|
| 高级 / 极简 | Noto Serif SC | Cormorant Garamond italic | `Noto+Serif+SC:wght@200;300;400&family=Cormorant+Garamond:ital,wght@0,300;1,300` |
| 活泼 / 国潮 | ZCOOL KuaiLe | DM Sans | `ZCOOL+KuaiLe&family=DM+Sans:wght@300;400` |
| 工装 / 运动 | Noto Sans SC | Space Grotesk | `Noto+Sans+SC:wght@300;400;500&family=Space+Grotesk:wght@300;400;500` |
| 复古 / 中式 | ZCOOL XiaoWei | Playfair Display | `ZCOOL+XiaoWei&family=Playfair+Display:ital,wght@0,400;1,400` |
| 现代 / 科技 | Noto Sans SC | Inter | `Noto+Sans+SC:wght@300;400&family=Inter:wght@300;400` |
| 手工 / 温暖 | Ma Shan Zheng | Lora | `Ma+Shan+Zheng&family=Lora:ital,wght@0,400;1,400` |

> ⚠️ **国内可用性**：URL host 必须用 `https://fonts.googleapis.cn/css2?family=...`（`.com` 国内被墙）；多个 family 之间用 `&family=` 连接（裸 `&` 会丢第二个 family）。每个 `font-family` 末位必须接同类系统字体兜底：serif CJK → `…,"Songti SC","SimSun",serif`；sans CJK → `…,"PingFang SC","Microsoft YaHei",sans-serif`；装饰体（ZCOOL/Ma Shan Zheng 无系统等价）→ 接 `"PingFang SC"` 或 `"Kaiti SC"`。详见 `references/code-patterns.md` §11。

### 字号规范（1440px 画布）

| 用途 | 字号 | weight | 备注 |
|---|---|---|---|
| 品牌名大标题 | 100–130px | 300 | 手机上约 25–33pt |
| 系列名 | 68–88px | 300 | |
| 段落正文 | 32–36px | 300, line-height 2.0–2.4 | 手机上约 8–9pt，保证可读 |
| 小标签/编号 | 24–28px | 400, letter-spacing 4–6px | |
| 功效数字/卖点大字 | 80–120px | 400–500 | 数码/美妆行业常用 |

---

## 第六部分：代码生成指引

> ⚠️ **生成代码前必须先读 `references/code-patterns.md`**，里面包含经过验证的完整代码模板。

### 核心架构要求

1. **BRAND 配置对象**：所有品牌信息（名称、色彩、字体）集中在一个 `BRAND` 对象中，CSS变量从中派生
2. **画布宽度固定 1440px**，预览缩放到 360px（缩放比 0.25）
3. **预览套手机外框**，让用户直观感受手机上的效果

### 必须遵守的技术规则

| 规则 | 原因 |
|---|---|
| 图片区用 `background-image`，不用 `<img>` | html2canvas 不支持 `object-fit: cover` |
| 每个图片区必须有占位背景色 + "点击上传"提示 | 否则未上传时页面空白 |
| 禁止 `innerHTML +=` | 会销毁已绑定的事件监听器 |
| 全部 `createElement` + `appendChild` | 保证事件不丢失 |
| `transform:scale()` 后必须手动设 wrapper 高度 | scale 不影响布局空间 |
| iframe 导出时字体用 `<link>` 写入 | 跨域 cssRules 提取会报错 |
| 字体 URL 从 `BRAND.googleFontsUrl` 读取 | 不硬编码，适配不同品牌 |
| 每个模块高度控制在 200–2000px（或平台上限）| 单模块图片高度限制（按平台） |
| 外部库用 BootCDN/staticfile，字体用 `fonts.googleapis.cn` | cdnjs / `fonts.googleapis.com` 国内被墙，导出会失败或字体错乱 |
| 每个 `font-family` 末位接同类系统字体兜底 | `.cn` 不可达时优雅降级，不崩成默认字体 |
| 导出前 `iDoc.fonts.check(BRAND.webFontCheck, 样本字)` 校验主 web 字体 | `fonts.ready` 失败也会 resolve；CJK 分片字体 check 须带模块真实字符，否则误判 |
| 复杂 inline SVG（mask/filter/foreignObject）慎用做装饰 | html2canvas 对其支持有限，可能渲染异常；优先简单 `path` |

### 图片上传区的视觉规范

**未上传状态**（必须同时具备）：
- 占位背景色（与模块风格协调）
- 居中虚线边框（`border: 2px dashed rgba(0,0,0,0.15)`）
- 居中图标 + "点击上传图片" 文字
- hover 时背景色加深

**已上传状态**：
- 隐藏提示文字和虚线
- 显示 `background-image: cover`
- 再次点击可重新上传

### 导出函数清单

导出相关的完整代码在 `references/code-patterns.md` 中，包括：

- `renderMod(id)` — 单模块渲染为 canvas（iframe隔离 + 字体等待 + 高度稳定检测）
- `compressCanvas(canvas, maxBytes)` — 压缩到 2MB 以内
- `expMod(id)` — 单模块导出
- `expZip()` — ZIP 打包（逐个渲染，释放内存）
- `expLong()` — 合并长图
- `safeFileName(str)` — 文件名安全处理
- `waitForStableHeight(el)` — 等待渲染高度稳定

### 必须引入的外部库

```html
<!-- 国内可达 CDN（cdnjs 国内常被墙/极慢）；备选 cdn.staticfile.org -->
<script src="https://cdn.bootcdn.net/ajax/libs/html2canvas/1.4.1/html2canvas.min.js"></script>
<script src="https://cdn.bootcdn.net/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
```

---

## 第七部分：质量检查清单

生成完 HTML 后，Claude 必须自查：

```
□ BRAND.canvasWidth === 1440
□ 预览缩放到 360px（scale 0.25），套手机外框
□ 所有图片区都有占位色 + 上传提示
□ makeUploadable 正确绑定在每个图片区
□ 所有文字节点有 contenteditable
□ 模块排序按钮（上移/下移）可用
□ 导出的三个按钮（单模块/ZIP/长图）都存在
□ renderMod 中字体 URL 从 BRAND 读取
□ 相邻模块使用了不同变体和背景色
□ 每个模块高度在 200–2000px 之间
□ 导出图片宽度 = 1440px
□ 导出图片压缩到 ≤ 2MB
□ 外部库用 BootCDN（非 cdnjs），字体 host 用 fonts.googleapis.cn
□ 每个 font-family 末位有同类系统字体兜底
□ renderMod 导出前用 iDoc.fonts.check(BRAND.webFontCheck, 模块真实字符) 校验主 web 字体
□【差异化】方案有"版块结构推导表"，每个板块回指 concept 某条（非通用词）
□【差异化】版块序列与第四部分任一范例【不相同】（非挑一个微调）
□【差异化】≥2 个板块结构按概念定制（真定制，非重命名）
□【差异化·换皮测试】把色/字/文案换成另一品牌，骨架是否依然成立？若"换皮即另一家店"=结构没差异化，打回
□【差异化·防脊柱趋同】版块**有序序列**不与同品类明显默认骨架雷同；窄语汇品类(极简女装/数码等)主动换 hero 原型或重排，别让同风格两家撞同一脊柱
□【配置】每个板块有配置面板：高度/背景/文字色/字号/对齐/图文比/镜像，改完实时生效且导出带上
□【导出安全】现做组件过 code-patterns.md 第6节清单；与差异化冲突时【可导出优先】
```
