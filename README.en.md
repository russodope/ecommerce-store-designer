# ecommerce-store-designer

[中文](README.md) · **English**

A platform-agnostic **skill for AI coding agents** to design e-commerce store homepages — it asks your export size first, produces brand-specific (non-templated) designs, generates a single-file visual editor, and exports images at **your target platform's size**.

> Works with **any coding agent** — Claude Code, Cursor, Cline, Windsurf, Codex, etc. Not tied to a platform or a single agent.

---

## Where it applies / decoration models

This skill produces **a set of layout images rendered at a width you specify**, so it fits most directly with platforms whose decoration model is "**add an image / custom module → upload one finished design image**." Models differ by platform:

| Platform | Decoration model | Fit |
|---|---|---|
| **Taobao/Tmall, JD, Pinduoduo, Douyin Store** and similar CN marketplaces | Back-office "store decoration": add modules (carousel / custom image / image-text / hotspots) and **upload one full image at the platform's required size** | ✅ **Direct fit** |
| **Shopify, custom sites, WooCommerce** and theme/site builders | Theme **sections** (responsive, with text / buttons / products); images go into Image banner / image-text / custom-HTML sections | ⚠️ **Partial fit**: drop the design images in as **banner / promo / image-text section images**; but these platforms are section-based and responsive, not a single stack of full-bleed images |

> Sizes vary by platform (JD carousel ~1440 wide, Douyin 1125×633, Shopify banner ~1600×1050) — which is why this skill **asks your export width first**, then renders to it.

---

## What it does

It guides your coding agent through a workflow:

1. **Research the brief** — first asks your **export size** (platform / width, e.g. 750 / 1080 / 1242 / 1440), then brand info, style, and **what you're promoting / listing this time** (drives "what to sell and how to order it" on the homepage).
2. **Research-driven, differentiated design** — the brand's **art direction (concept)** drives palette/typography **and layout structure** at the same time; every section must trace back to a concept point; a built-in "anti-spine-convergence" self-check avoids same-category sameness.
3. **Generate a single-file HTML editor** — left "Add module" palette + **drag to any position** on the canvas, click-to-edit text, click-to-replace images, per-section ⚙ controls (size/leading/color/align/height/image-text ratio/mirror), and one-click export of the whole PNG set **at your specified width**.

---

## Key features

- 🎯 **Asks size first, renders per platform**: designs internally against a unified **1440px reference width**, then scales on export to your `BRAND.exportWidth` (any width; height / file-size caps follow the platform too).
- 🧩 **Component kit + concept composition**: 13 export-safe layout components (multiple heroes / split / equal-height grid / masonry / horizontal lookbook / swatch strip / spec big-number / asymmetric editorial / statement / transition / brand story…), **composed** by concept into a brand-specific skeleton — **not a fixed template reskin**.
- ✏️ **Visual editor**: **drag-insert** modules from the palette, per-section layout config, click-to-edit text and images; all editing affordances **auto-hide on export**.
- 🌏 **China-network reliable + fonts that bake**: scripts via BootCDN, fonts via `fonts.googleapis.cn`; uses **FontFace** to bake fonts into the exported image.
- 📐 **Export-safe**: image zones use `background-image`, no advanced SVG, module height 200–2000px (split if over), compressed to ≤2MB (tunable per platform).

---

## How it works

The output is **images**, not a web page. On CN marketplaces, mobile "store decoration" is module + image assembly and doesn't take custom HTML, so you upload the skill's exported images into the matching modules; on theme platforms like Shopify you drop those images into image banner / image-text sections.

What the skill generates is a **local design tool** (a single-file HTML): design it in the browser → export images → upload to your store.

---

## File structure

| File | Role |
|---|---|
| `SKILL.md` | **Decision layer**: workflow, research questions, concept→structure, component-kit usage, differentiation QA checklist. |
| `references/code-patterns.md` | **Code layer (verified implementation)**: `BRAND` config, fixed machinery (`makeUploadable` / `renderMod`+FontFace / export scaling / `expZip` / `expLong`), 13-component kit, editor shell (left add-module palette + drag + config panel). |
| `references/industry-*.md` | Industry palette / layout references (apparel, beauty & fragrance, home & food, electronics/sports/baby). |

---

## Usage

At its core this is a set of **markdown instructions + code templates**, usable by any coding agent that can read local files / context:

- **Claude Code / Claude Desktop**: download [`ecommerce-store-designer.skill`](./ecommerce-store-designer.skill) (or from [Releases](../../releases)) and import into skills; or drop this directory into your skills folder.
- **Cursor / Cline / Windsurf / Codex, etc.**: feed `SKILL.md` + `references/` as rules / context / attachments and have the agent follow the `SKILL.md` workflow.

> A `.skill` file is just a ZIP — rename to `.zip` and unzip to read the source.

---

## License

MIT
