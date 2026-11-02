---
name: email-template-design
description: "Use this skill whenever the user wants to convert a Figma design into an HTML email template, build a transactional/marketing email from Figma, or asks to 'implement this email design', 'turn this Figma into an email', 'build an HTML email from this mockup', or shares a figma.com URL alongside a request for an email deliverable. Specializes in pixel-accurate, table-based, inline-CSS HTML emails that render consistently across Gmail, classic Outlook, Apple Mail, and mobile clients, with no external libraries, frameworks, or build tooling. Also covers consolidating multiple Figma 'step' frames (e.g. status variants of one notification) into a single non-conditional email, and extracting Figma assets as email-safe PNG/JPEG. Do NOT use for plain web pages/landing pages (not email) or for editing existing production ESP templates in Mailchimp/Klaviyo/Braze syntax without being told to."
---

# Email template design (Figma → production HTML email)

This skill turns a Figma design into **one standalone `.html` file** (plus an `img/` asset folder) that is safe to send through a real ESP. It is a practical playbook, not a style guide — follow the steps in order, run the actual tool calls shown, and use the snippets as a starting skeleton.

## 0. Non-negotiable constraints

- Plain HTML + inline CSS only. No React/Vue, no Tailwind/Bootstrap, no icon libraries, no JS, no build step the final file depends on.
- Table-based layout (`role="presentation"` tables), not flexbox/grid/absolute positioning.
- No inline `<svg>` and no base64 images in the shipped HTML — Gmail and classic Outlook are unreliable with both. Every logo/icon is a downloaded PNG (or JPEG for photos) referenced by URL/relative path.
- Reuse the project's existing assets/conventions before fetching anything new (check for an `img/` folder, existing logo files, prior templates).
- Never guess at content, colors, or copy — pull every value from Figma via the tools below.

## 1. Inspect the Figma file before writing any code

**Load the Figma design-to-code prerequisite first.** Before any `get_design_context` call, read the skill resource (or call the `/figma-design-to-code` skill if the client offers it):
```
ReadMcpResourceTool(server: "claude_ai_Figma", uri: "skill://figma/figma-design-to-code/SKILL.md")
```
Pass `skillNames: "resource:figma-design-to-code"` on every subsequent `get_design_context` call. Skipping this step produces code that ignores real asset downloads and hallucinates structure.

**Open a session** as soon as you know the file key (from the URL `figma.com/design/:fileKey/...`):
```
create_session(fileKey, description: "<short task description>")
```
Reuse the returned `sessionId` on every later call, including calls against other files.

**Find every frame, not just the one node-id in the URL.** A single Figma file commonly contains many "step" frames that are really variations of one email (e.g. `Confirmation-Departure`, `Assigned-Departure`, `Completed-Departure`, `Cancelled-Departure`, `Chauffeur Changed`, `Vehicle Changed`, and their `-Arrival` mirrors). To discover them:
```
get_metadata(fileKey)                       # no nodeId → lists top-level pages
get_metadata(fileKey, nodeId: "<page-id>")   # lists every top-level frame on that page with x/y/width/height
```
Grep/scan the returned XML for lines matching `<frame id="..." name="..."` at the top indentation level — that is your list of candidate "steps." Frame `x` position in Figma canvases is usually left-to-right in the order they were designed; use it as a tiebreaker for section ordering later.

**Screenshot every candidate frame before deciding anything.** Screenshots are cheap and visual — use them to triage before spending tokens on full `get_design_context` dumps:
```
get_screenshot(fileKey, nodeId, maxDimension: 1400)
```
Download each with `curl -f -L -o name.png "<image_url>"` (the URL is short-lived) and `Read` it to actually look at it. Do this for **every** frame before consolidating — do not assume two similarly-named frames are identical.

**Pull exact design tokens from the most complete/representative frame** with a full `get_design_context` call (not just metadata) — this is the only way to get real hex colors, font family/weight/size/line-height, border-radius, shadows, and the Figma asset URLs:
```
get_design_context(fileKey, nodeId, clientFrameworks: "unknown", clientLanguages: "html,css", skillNames: "resource:figma-design-to-code")
```
The response is React+Tailwind — **do not use that syntax**. Treat it purely as a spec: copy the hex values, px sizes, and the "These styles are contained in the design" token list into your own table-based HTML/inline-CSS. If the response is very large, call `get_metadata` on the frame first to find the specific sub-frame node-ids you still need (banner, one card, one list) and call `get_design_context` on just those — far cheaper than re-pulling the whole frame.

## 2. Consolidate steps into ONE template — do this correctly

If the file contains multiple status/step frames for what is logically one email:

1. Screenshot-compare every step frame against the "base" frame (usually the first/most complete one). Identify, section by section, what is **identical** (header, footer, order/service details card, support cards, legal/fraud notice) vs **unique per step** (the colored status banner, a status-specific detail card, that step's own "Good to know" bullet list).
2. Put every **identical** section in the template exactly once.
3. Put every **unique** section in the template exactly once, in the step's own place — do not merge differing copy into one generic paragraph, and do not drop a section because "it's similar enough" to another if the wording/data differs.
4. Order sections to match the real lifecycle/user journey (e.g. Confirmed → Assigned → Changed → Completed/Cancelled), not the arbitrary left-to-right canvas order, unless canvas order already reflects the journey.
5. **Never** implement this with `display:none`, JS toggles, conditional comments driving content choice, or query-param branches. Every section is simply present, stacked vertically, always visible. The "steps" become **one linear status timeline** in a single email.
6. If a status needs its own mini progress-tracker state (e.g. a red ✕ on step 2 for "Cancelled"), it's fine to re-render the tracker component once per terminal-state section it visually belongs to (e.g. once near the top in its fullest state, once again next to "Cancelled") — but don't repeat it for every single step if the state is visually identical to one already shown.

## 3. Asset extraction — PNG/JPEG only, never leave temp URLs in the HTML

**Rule of thumb for icon node ids:** `get_screenshot` only accepts plain Figma node ids (`123:456`), not instance-path ids like `I123:456;789:10`. When `get_design_context` shows an icon nested inside an instance (`data-node-id="I23:11422;1135:2525;..."`), you usually can't screenshot that exact nested node. Instead:
- Find a **sibling instance with a plain id** that renders the same glyph alone (e.g. the same check-icon used elsewhere as a direct instance), and screenshot that.
- Or check the `get_design_context` response for a **"Component descriptions"** block — it lists real, screenshot-able component node-ids (e.g. `check: 23:11294`, `alert-octagon: 23:9817`) with short usage descriptions. Try these; some will fail with "invalid node selection" if the component isn't placed on a visible page — that's expected, fall back to the next option.
- Or just screenshot the whole small instance that **is** a plain id one level up (e.g. the circular "Featured icon" wrapper, or the whole Badge) — simpler and usually looks identical since Figma already baked the background into the component.
- If no reliable node exists, implement that glyph with plain HTML/CSS instead (see below) rather than spending more calls chasing an asset.

**Download every successfully-screenshotted or exported asset immediately:**
```
curl -f -L -s -o img/icon-name.png "<image_url from get_screenshot or get_design_context>"
```
Figma asset URLs expire (~7 days) — the shipped HTML must reference local/hosted files, never the temporary URL.

**Reuse project assets first.** If the project already has a logo (e.g. `img/logo.svg`), use that over re-fetching from Figma, per the project's existing convention. If you need a PNG and only have the local SVG:
```bash
npm install --no-save sharp   # or: pip install cairosvg (needs system cairo; sharp is more portable on Windows)
node -e "require('sharp')('img/logo.svg',{density:600}).resize(W,H).png().toFile('img/logo.png')"
```
To make a **white silhouette** of a colored logo for a dark footer (common requirement), extract the alpha channel and recomposite it onto solid white — do not try to hand-edit the SVG fill colors if it uses gradients:
```js
const sharp = require('sharp');
const alpha = await sharp('logo.svg',{density:600}).resize(W,H).ensureAlpha().extractChannel('alpha').raw().toBuffer();
const whiteRgb = await sharp({create:{width:W,height:H,channels:3,background:{r:255,g:255,b:255}}}).raw().toBuffer();
await sharp(whiteRgb,{raw:{width:W,height:H,channels:3}})
  .joinChannel(alpha,{raw:{width:W,height:H,channels:1}})
  .png().toFile('logo-white.png');
```
(Always pass `.raw()` when extracting a channel for reuse as a buffer — omitting it causes a "memory area too small" error from a bare PNG buffer being misread as raw pixels.)

**When to use plain HTML/CSS instead of an image** ("simple icons"): dividers, bullet dots, numbered step circles, a generic "!" or text-glyph badge, pickup/drop-off route markers (small circle + 1px vertical line + circle, built from `<td>` backgrounds), and anything where an exact brand/icon shape isn't essential. Reserve PNG for: logos, app-store badges, social/brand glyphs (Facebook/YouTube/LinkedIn/WhatsApp — recognizability matters), and any detailed multi-color icon (alert triangle, info glyph, car/bank icons) that a CSS shape can't reasonably approximate.

**Never use SVG in the shipped email**, even if Figma exports it as SVG — rasterize to PNG first. Prefer JPEG only for actual photographs; PNG for everything else (keeps transparency).

## 4. HTML skeleton (start from this, don't reinvent it)

```html
<!DOCTYPE html>
<html lang="en" xmlns="http://www.w3.org/1999/xhtml" xmlns:v="urn:schemas-microsoft-com:vml" xmlns:o="urn:schemas-microsoft-com:office:office">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta http-equiv="X-UA-Compatible" content="IE=edge">
<meta name="color-scheme" content="light">
<meta name="supported-color-schemes" content="light">
<title>...</title>
<!--[if mso]>
<noscript><xml><o:OfficeDocumentSettings><o:PixelsPerInch>96</o:PixelsPerInch></o:OfficeDocumentSettings></xml></noscript>
<![endif]-->
<!-- Optional progressive-enhancement web font: Outlook skips it via the !mso guard and uses the fallback stack below. -->
<!--[if !mso]><!-->
<link href="https://fonts.googleapis.com/css2?family=<Font+Name>:wght@400;500;600;700&display=swap" rel="stylesheet" type="text/css">
<!--<![endif]-->
<style>
  body, table, td, p, a { -webkit-text-size-adjust: 100%; -ms-text-size-adjust: 100%; }
  table, td { mso-table-lspace: 0pt; mso-table-rspace: 0pt; }
  img { -ms-interpolation-mode: bicubic; border: 0; outline: none; text-decoration: none; display: block; }
  body { margin: 0; padding: 0; width: 100% !important; height: 100% !important; background-color: #f4f8fb; }
  a { color: inherit; }
  @media only screen and (max-width: 600px) {
    .full-w { width: 100% !important; }
    .fluid-pad { padding-left: 20px !important; padding-right: 20px !important; }
    .stack { display: block !important; width: 100% !important; }
  }
</style>
</head>
<body style="margin:0; padding:0; background-color:#f4f8fb; text-align:center;">
<div style="display:none; max-height:0; overflow:hidden; mso-hide:all;">Preheader text...</div>
<!--[if mso]>
<table role="presentation" width="600" align="center" cellpadding="0" cellspacing="0" border="0"><tr><td>
<![endif]-->
<table role="presentation" class="full-w" width="600" align="center" cellpadding="0" cellspacing="0" border="0"
       style="width:600px; max-width:600px; margin:0 auto; background-color:#f4f8fb; text-align:left;">
  <!-- sections go here -->
</table>
<!--[if mso]>
</td></tr></table>
<![endif]-->
</body>
</html>
```

Notes on this skeleton (each line earned its place from a real bug):
- **Centering without the deprecated `<center>` tag:** `text-align:center` on `<body>` + `align="center"` + `margin:0 auto` on the main table does the job in modern clients; the `<!--[if mso]-->` wrapper table with `align="center"` is what actually centers it in Outlook (Outlook ignores `margin:auto` on tables). Put `text-align:left` back on the main content table immediately, or the inherited `center` from `<body>` will wreck every unaligned paragraph/list inside it.
- **Every full-width table needs BOTH `width="100%"` (HTML attribute) AND `style="width:100%;..."` (CSS).** Relying on only the attribute causes some clients/preview tools to collapse the table to its content width once any other inline style is present — a real bug, not a hypothetical. Audit with:
  ```bash
  grep -c 'width="100%"' file.html
  grep -c 'width="100%".*style="width:100%' file.html   # should match the count above
  ```
- **Web fonts are progressive enhancement only.** Declare `font-family: '<Original Font>', Helvetica, Arial, sans-serif` everywhere. If the font is on Google Fonts, add the `<link>` (wrapped in `<!--[if !mso]><!-->...<!--<![endif]-->`) so supporting clients (Apple Mail, iOS Mail, some Android/Gmail, browser previews) render the real typeface — but never hand-write guessed `@font-face` `src:` URLs; Google's hashed `gstatic.com` filenames are not guessable and will 404. Let the `<link>` do it, or fetch `https://fonts.googleapis.com/css2?family=...` yourself first to confirm real URLs if you need an explicit `@font-face` block.
- **Buttons/CTAs:** never a styled `<a>` alone — wrap it in a table cell with background-color, padding, and border-radius, with the `<a>` just providing the click target and inheriting color/no-underline:
  ```html
  <table role="presentation" cellpadding="0" cellspacing="0" border="0"><tr>
    <td style="background-color:#603bb4; border-radius:8px; padding:8px 24px;">
      <a href="tel:..." style="color:#ffffff; text-decoration:none; font-weight:600;">Call</a>
    </td>
  </tr></table>
  ```
- **Two-column layouts that stack on mobile:** two `<td>`s with `class="stack"` and explicit `width="50%"`, each `padding-left`/`padding-right` adjusted via a mobile-only class, so they go full-width and vertical under the `max-width:600px` media query.
- **Images:** always explicit `width`/`height` attributes matching the displayed size (never rely on CSS alone to size an email image — many clients need the attributes), `alt=""` for decorative icons, descriptive `alt="..."` for meaningful images/logos, `style="display:block;"` to kill the default inline whitespace gap under images.

## 5. Validate before calling it done

Spin up a disposable headless browser to actually look at the output — don't ship on faith:
```bash
npm install --no-save playwright-chromium
npx playwright install chromium
node -e "
const { chromium } = require('playwright-chromium');
(async () => {
  const b = await chromium.launch();
  const p = await b.newPage({ viewport: { width: 680, height: 1200 } });
  await p.goto('file:///' + process.cwd().replace(/\\\\/g,'/') + '/email-template.html');
  await p.screenshot({ path: 'qa-desktop.png', fullPage: true });
  await b.close();
})();
"
```
Repeat at `width: 375` for a mobile check. Read both screenshots and crop/zoom into specific sections (`page.screenshot({clip:{x,y,width,height}})`) when checking small details like icon alignment or font rendering.

**Then delete the QA tooling** (`node_modules`, `package.json`, `package-lock.json`, any `qa-*.js`/`qa-*.png` scratch files) — the shipped deliverable is the `.html` file and `img/` folder only, nothing else. The instructions explicitly forbid build-time dependencies in the final template; this tooling is scaffolding for your own verification, not part of the product.

## 6. Final checklist (run through explicitly before reporting done)

**Visual accuracy**
- [ ] Every unique section from every inspected Figma frame is present and visible (no step was silently dropped or merged away because it "looked similar").
- [ ] Fonts, sizes, weights, line-heights, colors, border-radius, and spacing match the values pulled from `get_design_context`, not eyeballed.
- [ ] All copy/content is the exact Figma text — no placeholder/lorem ipsum, no paraphrasing.
- [ ] Desktop (≥600px) screenshot matches the Figma screenshot section-by-section.
- [ ] Mobile (~375px) screenshot shows full-width stacking, no horizontal scroll/overflow, no clipped text.

**Assets**
- [ ] No `<svg>` tags and no `data:image/...;base64` URIs anywhere in the shipped HTML.
- [ ] No leftover `figma.com/api/mcp/asset/...` temporary URLs — every image points at a local/hosted file.
- [ ] Every `<img>` has explicit `width`/`height` and an appropriate `alt` (empty for decorative).
- [ ] Logos/brand icons preserved at correct aspect ratio (no stretching).

**Email-client compatibility**
- [ ] Layout is 100% table-based (`role="presentation"`, `cellpadding="0" cellspacing="0" border="0"`), no flexbox/grid/absolute positioning.
- [ ] All critical styling (fonts, colors, padding, borders, widths) is inline, not solely in `<style>`.
- [ ] Every `width="100%"` table also has `style="width:100%;...` (checked via the grep above).
- [ ] MSO conditional wrapper present for Outlook centering; `mso-table-lspace/rspace: 0` reset in place.
- [ ] Buttons/links are table-cell based, large enough to tap on mobile (≥ ~36px touch target height).
- [ ] Web font (if any) is progressive enhancement only, wrapped out for Outlook, with a real email-safe fallback stack declared everywhere.
- [ ] No JavaScript, no `<form>`/interactive elements relying on scripting.

**Accessibility**
- [ ] Reasonable text/background contrast on every colored banner/badge.
- [ ] Semantic reading order: header → content → footer makes sense with images disabled (alt text alone conveys meaning).
- [ ] Links/buttons have descriptive text (not bare "click here").

**Structure/cleanup**
- [ ] HTML validated: table open/close tag counts match; no attribute with an unmatched quote (a real failure mode when scripting bulk edits — verify with a quick regex pass).
- [ ] No temporary QA/build tooling (`node_modules`, `package.json`, playwright, scratch screenshots) left in the deliverable directory.

## 7. Final report format

When done, report concisely:
1. Files created/modified (the `.html` path and every asset added under `img/`).
2. How Figma's step/variant frames were consolidated into one template (what was deduped vs. kept unique, and the section order chosen).
3. Which assets became PNG vs. which were done in plain HTML/CSS, and why.
4. Font choice and fallback behavior per client tier.
5. Any known differences from the Figma design caused by email-client limitations, and anything that couldn't be sourced/verified from Figma.
