# WSD — Workshop Scraping & Documentation Workflow

> God-rules document. Defines the complete workflow for any agent (stateless, no memory)
> to scrape an AWS Workshop Studio workshop and convert it into a bilingual Hugo documentation site.
>
> This file is self-contained. An agent reading only this file should be able to execute the full workflow.

---

## Table of Contents

1. [Prerequisites & Environment](#1-prerequisites--environment)
2. [Phase 0: Authentication](#2-phase-0-authentication)
3. [Phase 1: Table of Contents Discovery](#3-phase-1-table-of-contents-discovery)
4. [Phase 2: Repo Restructure](#4-phase-2-repo-restructure)
5. [Phase 3: Content Extraction](#5-phase-3-content-extraction)
6. [Phase 4: Image Download](#6-phase-4-image-download)
7. [Phase 5: Vietnamese Translation](#7-phase-5-vietnamese-translation)
8. [Phase 6: Verification](#8-phase-6-verification)
9. [Task Delegation Strategy](#9-task-delegation-strategy)
10. [Roadmap File Specification](#10-roadmap-file-specification)
11. [Hugo Conventions Reference](#11-hugo-conventions-reference)
12. [Error Recovery](#12-error-recovery)

---

## 1. Prerequisites & Environment

### Required Tools

| Tool | Purpose | Config Requirement |
|------|---------|-------------------|
| Playwright MCP | Browser automation for scraping | **Must run in `headed` mode** (`"headless": false` in mcp.json) |
| Hugo (extended) | Static site generator | Version specified in `AGENTS.md` |
| Subagent capability | Delegate translation & verification to parallel workers | Platform-dependent — use whatever sub-task/parallel mechanism is available. If unsupported, execute sequentially on main thread. |

### Playwright MCP Configuration

The Playwright MCP server **MUST** be configured in headed mode so the browser window is visible to the user for manual login:

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@anthropic-ai/playwright-mcp@latest", "--headless", "false"],
      "disabled": false
    }
  }
}
```

### Key Files

| File | Role |
|------|------|
| `WSD.md` | This file — god rules, never modified during execution |
| `WSD.roadmap.md` | Generated during execution — tracks progress per page |
| `AGENTS.md` | Hugo repo conventions (front matter, shortcodes, structure) |

---

## 2. Phase 0: Authentication

### Goal

Access the workshop content via Playwright. Handle login if required.

### Procedure

```
1. Navigate to workshop URL:
   playwright_browser_navigate("<workshop_url>")
   playwright_browser_wait_for(time: 5)

2. Take snapshot to check state:
   playwright_browser_snapshot()

3. Evaluate access:
   - IF page shows workshop content (sidebar, article text) → SKIP to Phase 1
   - IF page shows login/sign-in form → Continue to step 4

4. Login required:
   a. STOP processing
   b. Inform user:
      "Workshop requires authentication. The browser window is open in headed mode.
       Please complete the login manually (AWS Builder ID / Event credentials).
       Reply 'continue' when you are logged in and can see the workshop content."
   c. WAIT for user response

5. After user says "continue":
   playwright_browser_snapshot()
   - IF workshop content is visible → Proceed to Phase 1
   - IF still on login page → Inform user login appears incomplete, ask to retry
```

### Important Notes

- **NEVER close the Playwright browser** after login — session is stateless, login will be lost
- Workshop Studio URLs typically follow: `https://catalog.us-east-1.prod.workshops.aws/workshops/<id>/...`
- Event-based URLs: `https://catalog.us-east-1.prod.workshops.aws/event/dashboard/en-US/workshop/...`

---

## 3. Phase 1: Table of Contents Discovery

### Goal

Extract the complete workshop page structure (sidebar navigation) to build the roadmap.

### Procedure

```
1. Navigate to workshop homepage (root page)
2. playwright_browser_snapshot()
3. Identify the sidebar/navigation element containing all page links
4. Extract:
   - Page titles (in order)
   - Page URL paths (relative)
   - Nesting hierarchy (parent/child relationships)
   - Section vs leaf page distinction
```

### Extraction Script

```javascript
playwright_browser_evaluate({
  function: `() => {
    // Adapt selector to workshop platform's sidebar structure
    const links = document.querySelectorAll('nav a, [class*="sidebar"] a, [class*="navigation"] a');
    return Array.from(links).map(a => ({
      title: a.textContent.trim(),
      href: a.getAttribute('href'),
      depth: /* calculate from nesting/indentation */
    }));
  }`
})
```

### Output

After extracting TOC, immediately create `WSD.roadmap.md` following the specification in [Section 10](#10-roadmap-file-specification).

### Delegation

This phase MAY be delegated to a subagent (explorer role) if the sidebar structure is complex.

---

## 4. Phase 2: Repo Restructure

### Goal

Clean existing template content and create Hugo directory structure matching the discovered TOC.

### Procedure

```
1. DELETE all files under content/ EXCEPT:
   - content/_index.md (will be overwritten with workshop homepage)
   - content/_index.vi.md (will be overwritten)

2. KEEP intact:
   - config/ (all Hugo configuration)
   - layouts/ (all template overrides)
   - themes/ (Git submodule)
   - static/ (shared assets — favicon, fonts, etc.)
   - archetypes/
   - .github/
   - AGENTS.md, WSD.md

3. CREATE directory structure from TOC map:
   For each page discovered in Phase 1:
   - Create folder: content/<slug>/
   - Create empty: content/<slug>/_index.md
   - Create empty: content/<slug>/_index.vi.md
   For nested pages:
   - Create: content/<parent-slug>/<child-slug>/_index.md
   - Create: content/<parent-slug>/<child-slug>/_index.vi.md

4. CREATE static/images/workshop/ directory (for workshop images)

5. VERIFY Hugo build:
   hugo --config config/development/hugo.toml --destination tmp-hugo-build
   Remove-Item -Recurse -Force tmp-hugo-build
```

### Directory Naming Rules

- Lowercase, hyphenated: `lab1-agent-prototype`, `prerequisites`
- Match workshop URL slug when possible
- No numeric prefixes unless needed for disambiguation (use `weight` in front matter for ordering)

---

## 5. Phase 3: Content Extraction

### Goal

For each workshop page, extract content and write into the corresponding Hugo markdown file (English).

### Procedure (per page)

```
1. NAVIGATE to the workshop page:
   playwright_browser_navigate("<workshop_base_url>/<page_path>")
   playwright_browser_wait_for(time: 4)

2. CAPTURE default state:
   playwright_browser_snapshot()
   — Record all text content, headings, code blocks, callouts, tables

3. IDENTIFY multi-option blocks (tabbed panels):
   — Look for tab/toggle UI elements (e.g. "AgentCore CLI | Interactive", "macOS/Linux | Windows")
   — Record which tab is currently active (this is "Option 1")
   — Record the content of Option 1

4. CAPTURE remaining tab options:
   For EACH inactive tab:
     a. playwright_browser_click({ element: "<tab label>", ref: "<ref>" })
     b. playwright_browser_snapshot()
     c. Record content of this tab option
   — Note: Tabs can be NESTED (outer tab → inner sub-tabs). Capture ALL combinations.

5. WRITE English markdown file:
   — Write content/_index.md with proper front matter
   — Convert workshop elements to Hugo shortcodes (see mapping below)
   — Preserve ALL code blocks exactly as shown (never truncate)
   — Preserve ALL CLI commands with full flags
   — Reference images with placeholder comments: <!-- IMAGE: description -->

6. UPDATE roadmap:
   — Mark "content" task as complete for this page
   — Mark "multi-option blocks" task as complete
   — Note any code blocks found
```

### Workshop Element → Hugo Shortcode Mapping

| Workshop Element | Hugo Shortcode |
|-----------------|----------------|
| Tabbed panel | `{{</* tabs */>}}` / `{{</* tab name="..." */>}}` |
| Callout/alert box | `{{% notice info/warning/tip %}}` |
| Expandable/collapsible section | `{{% expand "Title" %}}` |
| Internal page link | `{{</* ref "path" */>}}` |
| Code block | Standard markdown fenced code block |
| Mermaid diagram | `{{</* mermaid */>}}` (NEVER fenced code block) |

### Multi-Option Block Rules

Workshop tabbed panels contain different content per tab. ALL tabs must be captured:

```markdown
{{< tabs >}}
{{< tab name="macOS/Linux" >}}
... content for macOS/Linux ...
{{< /tab >}}
{{< tab name="Windows" >}}
... content for Windows ...
{{< /tab >}}
{{< /tabs >}}
```

For nested tabs (tab within tab), use unique groupId:

```markdown
{{< tabs groupId="method" >}}
{{< tab name="AgentCore CLI" >}}

Choose your OS:

{{< tabs groupId="os-cli" >}}
{{< tab name="macOS/Linux" >}}
... CLI commands for macOS ...
{{< /tab >}}
{{< tab name="Windows" >}}
... CLI commands for Windows ...
{{< /tab >}}
{{< /tabs >}}

{{< /tab >}}
{{< tab name="Interactive" >}}
... interactive method content ...
{{< /tab >}}
{{< /tabs >}}
```

### Content Fidelity Rules

- **NEVER** truncate code blocks with `...` or `# ...rest`
- **NEVER** summarize — preserve full instructional text
- **NEVER** add the page title as H1/H2 in body (front matter `title` renders it)
- **ALWAYS** preserve exact CLI commands with all flags and arguments
- **ALWAYS** preserve table data completely
- **ALWAYS** convert callout/alert boxes to `{{% notice %}}` shortcodes

---

## 6. Phase 4: Image Download

### Goal

Download all images from the workshop page and reference them in the markdown content.

### Background

Workshop images are hosted on CloudFront CDN (`static.us-east-1.prod.workshops.aws`) with signed URLs. Standard methods (`fetch()`, `canvas.toDataURL()`) fail due to CORS/tainted canvas restrictions. The proven method is: navigate directly to the signed image URL, then screenshot the rendered `<img>` element.

### Procedure (per page)

```
1. NAVIGATE to the workshop page (if not already there):
   playwright_browser_navigate("<workshop_base_url>/<page_path>")
   playwright_browser_wait_for(time: 4)

2. EXTRACT all image URLs from the page:
   playwright_browser_evaluate({
     function: `() => {
       const imgs = document.querySelectorAll('article img');
       return Array.from(imgs).map(img => ({
         src: img.src.replace(/%7E/g, '~'),
         alt: img.alt,
         w: img.naturalWidth,
         h: img.naturalHeight
       }));
     }`
   })

   ⚠️ CRITICAL: Must replace %7E → ~ in URLs.
   Browser encodes tilde as %7E in img.src but CloudFront signed URLs require literal ~

3. EXTRACT signed URL parameters (first time only):
   From any image URL, extract:
   - Key-Pair-Id=...
   - Policy=...
   - Signature=...
   These params typically use a wildcard policy covering ALL images in the workshop.
   Save them for reuse across all image downloads.

4. For EACH image:
   a. Determine descriptive filename based on image context:
      - Pattern: <page-slug>_<description>.png
      - Example: lab1_architecture_diagram.png, lab3_gateway_sequence.png
      - Name describes WHAT the image shows, not its position

   b. Navigate directly to signed image URL (with ~ decoded):
      playwright_browser_navigate("<image_url_with_tildes>")
      — Console may show 403 ERROR — IGNORE IT (403 is from subsidiary resources, not the image itself)
      — Page title will show "filename.png (WxH)" confirming image loaded successfully

   c. Screenshot the image element:
      playwright_browser_take_screenshot({
        element: "<descriptive name>",
        ref: "e2",
        filename: "<descriptive_filename>.png"
      })
      — When navigating directly to an image URL, the page contains only 1 img element
      — ref is typically "e2" but verify via snapshot if needed

   d. Copy file from Playwright temp directory to project:
      Copy-Item "<playwright_temp_output_path>/<filename>.png" "static/images/workshop/<filename>.png"
      — Playwright saves screenshots to a temp directory (check output path from screenshot result)

5. REPLACE image placeholder comments in _index.md:
   <!-- IMAGE: description --> → ![description](/images/workshop/<filename>.png)

6. UPDATE roadmap:
   — Mark "images" task as complete for this page
   — Mark "image references" task as complete
```

### Image Naming Convention

| Pattern | Example |
|---------|---------|
| Architecture diagrams | `<page>_architecture_diagram.png` |
| UI screenshots | `<page>_<ui_element>_screenshot.png` |
| Terminal output | `<page>_terminal_<action>.png` |
| Config files | `<page>_<config_name>_config.png` |
| IDE/editor views | `<page>_<tool>_<context>.png` |

### Signed URL Technical Details

Workshop images follow this base URL pattern:
```
https://static.us-east-1.prod.workshops.aws/<workshop-uuid>/static/<section-folder>/<image-name>.png
```

Signed URL components appended as query params:
```
?Key-Pair-Id=<id>&Policy=<base64>&Signature=<sig>
```

Key facts:
- Policy typically contains a wildcard resource (`/*`) covering all images in the workshop
- All images in the same workshop share the same signed params
- Policy has a `DateLessThan` expiry — check the epoch timestamp
- Signature contains `~` characters — these must NOT be URL-encoded

### Methods That Do NOT Work

| Method | Error |
|--------|-------|
| `fetch()` in page context | CORS block (cross-origin) |
| `canvas.toDataURL()` | Tainted canvas (cross-origin image) |
| Navigate without signed params | 403 Missing Key-Pair-Id |
| Navigate with `%7E` (not decoded to `~`) | 403 Invalid signature |
| `Invoke-WebRequest` / `curl` | May work if cookies/auth not required, but signed URLs from browser are more reliable |

### Fallback Method: Lightbox Screenshot

If signed URL approach fails (expired, changed), use the lightbox method:

```
1. Resize browser for maximum clarity:
   playwright_browser_resize({ width: 2560, height: 1440 })

2. Navigate to the workshop page containing the image

3. Click the image → opens lightbox/fullscreen dialog

4. Screenshot the img element inside the lightbox dialog:
   playwright_browser_take_screenshot({
     element: "<description>",
     ref: "<img ref inside lightbox>",
     filename: "<filename>.png"
   })

5. Close lightbox: click "Minimize image" or press Escape

6. Copy file to static/images/workshop/
```

Quality is lower than direct signed URL method but text remains readable.

---

## 7. Phase 5: Vietnamese Translation

### Goal

Create Vietnamese translation for every English page. This phase is **delegated to a subagent** when possible.

### Task Brief: Translation

**Input:** `content/<path>/_index.md` (completed English page)
**Output:** `content/<path>/_index.vi.md`

**Instructions for the translator (subagent or main thread):**

```
Translate the English Hugo page at content/<path>/_index.md into Vietnamese.
Create content/<path>/_index.vi.md with:
- Identical front matter (same weight, pre, date, chapter)
- Title translated to Vietnamese
- All body text translated to proper Vietnamese with diacritics
- All headings translated
- All image alt text translated
- Code blocks and CLI commands kept in English exactly as-is
- Shortcode structure preserved (only translate text content inside)
- Image paths unchanged
Read the English file first, then create the Vietnamese version.
```

> **Fallback:** If the platform does not support subagents, the main thread executes this task directly after completing content + images for the page.

### Translation Rules

| Element | Translate? |
|---------|-----------|
| Body text, headings | ✅ Yes — proper Vietnamese with diacritics |
| Image alt text | ✅ Yes |
| `title` in front matter | ✅ Yes |
| Code blocks | ❌ No — keep English exactly |
| CLI commands | ❌ No — keep English exactly |
| File/directory names | ❌ No |
| Shortcode names/params | ❌ No (translate content inside only) |
| Tab names | ✅ Yes (e.g. "macOS/Linux" stays, but descriptive names translate) |
| Notice type keywords | ❌ No (`info`, `warning`, `tip` stay English) |

### Quality Standards

- Use proper Vietnamese diacritics (ă, â, đ, ê, ô, ơ, ư, etc.)
- Never use ASCII transliteration
- Technical terms may remain in English if no standard Vietnamese equivalent exists
- Keep the same paragraph structure as English version

---

## 8. Phase 6: Verification

### Goal

Verify that Hugo content matches the original workshop content via text comparison.

### Procedure (per page)

```
1. NAVIGATE to the workshop page:
   playwright_browser_navigate("<workshop_url>/<page_path>")
   playwright_browser_wait_for(time: 4)

2. CAPTURE full text content:
   playwright_browser_snapshot()
   — Focus on article/main content area
   — Also click through ALL tabs to capture their content

3. READ the local Hugo markdown:
   read_file("content/<path>/_index.md")

4. COMPARE — check for:
   - Missing sections or headings
   - Missing or truncated code blocks
   - Missing CLI commands
   - Missing table rows/columns
   - Missing callout/notice content
   - Missing tab/option content (ALL tabs must be present)
   - Incorrect ordering of steps
   - Missing image references

5. FIX any discrepancies:
   - Update _index.md (EN) with missing content
   - Delegate VI update to subagent (re-translate affected sections)

6. VERIFY Hugo build:
   hugo --config config/development/hugo.toml --destination tmp-hugo-build
   Remove-Item -Recurse -Force tmp-hugo-build

7. UPDATE roadmap:
   - Mark "verified" for this page
   - Note what was fixed (or "OK — no issues")
```

### Verification Checklist (per page)

- [ ] All headings present and correctly ordered
- [ ] All code blocks complete (not truncated)
- [ ] All CLI commands with full flags
- [ ] All tables complete with all rows
- [ ] All notice/callout boxes converted
- [ ] All tabbed panels with ALL options captured
- [ ] All images referenced
- [ ] No duplicate H1/H2 title (front matter handles it)
- [ ] Hugo build passes without errors

---

## 9. Task Delegation Strategy

### Purpose

Optimize main thread context window by delegating repetitive or context-heavy tasks to subagents (parallel workers).

### Role Assignment

| Role | Responsibility | Delegation? |
|------|---------------|-------------|
| **Orchestrator** | Authentication, TOC discovery, repo restructure, content extraction (EN), image download, roadmap management | Main thread — NEVER delegate |
| **Translator** | Vietnamese translation of each page | DELEGATE — isolated task, only needs the EN file |
| **Verifier** | Per-page verification against live workshop | DELEGATE — independent per page after content + images done |
| **Explorer** | Initial sidebar/TOC analysis if structure is complex | MAY delegate for complex workshops |

### Delegation Rules

1. **Main thread writes English content** — requires Playwright interaction, context about page structure
2. **Subagent translates to Vietnamese** — isolated task, only needs the English file as input
3. **Subagent verifies** — can be done independently per page after content + images are complete
4. **Never delegate image download** — requires active Playwright session with auth cookies

### Context Window Management

- Main thread handles ONE page at a time through the full pipeline (content → images → delegate translation)
- After completing a page, update roadmap before moving to next
- If context is getting large, summarize completed work in roadmap and continue
- Subagent results are trusted — do not re-read files they've already processed

### Task Briefs

#### Translation Brief

```
TASK: Translate content/<path>/_index.md to Vietnamese
INPUT: content/<path>/_index.md
OUTPUT: content/<path>/_index.vi.md
RULES:
- Translate: title, headings, body text, image alt text
- Keep unchanged: code blocks, CLI commands, file names, shortcode syntax, image paths
- Use proper Vietnamese diacritics
- Match paragraph structure of English version
```

#### Verification Brief

```
TASK: Verify content/<path>/_index.md against live workshop
INPUT: content/<path>/_index.md + workshop URL
STEPS:
1. Navigate Playwright to: <workshop_url>/<page_path>
2. Wait 4 seconds, then snapshot
3. Click through ALL tabs/options and snapshot each
4. Compare snapshot text with the local markdown file
5. Report: missing sections, truncated code blocks, missing tabs, any discrepancy
6. If issues found, fix the _index.md file directly
```

### Platform Compatibility

| Platform | How to delegate |
|----------|----------------|
| Kiro | `invoke_sub_agent({ name: "general-task-execution", prompt: "<task brief>" })` |
| Claude Code | Use `Task` tool or run sequentially |
| Cursor/Windsurf | Use composer agent or run sequentially |
| Any other | If subagent/parallel is available, use it. Otherwise, main thread executes task briefs sequentially. |

> **Fallback:** If the platform does not support subagents, the main thread executes all tasks sequentially. Priority order: all EN content first → all images → all translations → all verifications.

---

## 10. Roadmap File Specification

### File: `WSD.roadmap.md`

This file is CREATED during Phase 1 and UPDATED throughout execution. It is the single source of truth for progress tracking.

### Structure

```markdown
# WSD Roadmap — <Workshop Title>

> Generated: <timestamp>
> Workshop URL: <base_url>
> Status: IN PROGRESS | COMPLETE

## Workshop Info

- Title: <workshop title>
- Total pages: <N>
- Base URL: <url>

## Progress Summary

- Pages discovered: <N>
- Content extracted: <X>/<N>
- Images downloaded: <X>/<N>
- Translated (VI): <X>/<N>
- Verified: <X>/<N>

---

## Page Map

| # | Title | URL Path | Hugo Path | Weight | Pre |
|---|-------|----------|-----------|--------|-----|
| 0 | Prerequisites | /10-intro | content/prerequisites/ | 1 | " <b> 0. </b> " |
| 1 | Lab 1: ... | /20-lab1 | content/lab1-.../ | 2 | " <b> 1. </b> " |
| ... | ... | ... | ... | ... | ... |

---

## Tasks Per Page

### Page: <title> — <hugo_path>

| Task | Status | Notes |
|------|--------|-------|
| Content (default tab) | [ ] | |
| Multi-option blocks (all tabs) | [ ] | |
| Code blocks | [ ] | |
| Images downloaded | [ ] | |
| Images referenced in content | [ ] | |
| Vietnamese translation | [ ] | |
| Verification | [ ] | |

(Repeat for each page)

---

## Issues & Decisions

| # | Issue | Decision | Date |
|---|-------|----------|------|
| 1 | ... | ... | ... |
```

### Task Definitions

| Task | Definition | Complete When |
|------|-----------|---------------|
| Content (default tab) | Extract text from default/first visible state of page | Full markdown written with all headings, text, code blocks from default view |
| Multi-option blocks | Capture ALL tab/toggle options beyond the default | All tabs converted to `{{</* tabs */>}}` shortcodes with complete content |
| Code blocks | All code blocks preserved exactly | Every code block in workshop appears in markdown, untruncated |
| Images downloaded | All `<img>` from page saved to `static/images/workshop/` | Files exist on disk with descriptive names |
| Images referenced | Each downloaded image linked in the correct position in markdown | `![alt](/images/workshop/name.png)` at correct location |
| Vietnamese translation | `_index.vi.md` created with full translation | File exists, proper Vietnamese, matches EN structure |
| Verification | Content compared against live workshop | All discrepancies resolved or documented |

---

## 11. Hugo Conventions Reference

> Quick reference extracted from `AGENTS.md`. For full details, read that file.

### Front Matter Template

```yaml
---
title: "Page Title"
date: 2026-01-01
weight: 1
chapter: false
pre: " <b> 1. </b> "
---
```

### Weight & Pre Convention

- Homepage: no `pre`, weight irrelevant
- Prerequisites section: weight=1, pre=" <b> 0. </b> "
- Lab N: weight=N+1, pre=" <b> N. </b> "
- Summary: weight=last, pre=" <b> last. </b> "
- Child pages: pre=" <b> N.M. </b> "

### Image Paths

```markdown
![Description](/images/workshop/filename.png)
```

All workshop images stored in `static/images/workshop/` and referenced with root-relative paths.

### Internal Links

```markdown
[Link text]({{< ref "path/to/page" >}})
```

### Shortcode Quick Reference

```markdown
{{% notice info %}}Important text{{% /notice %}}
{{% notice warning %}}Warning text{{% /notice %}}
{{% notice tip %}}Tip text{{% /notice %}}

{{% expand "Click to expand" %}}Hidden content{{% /expand %}}

{{< tabs >}}
{{< tab name="Option A" >}}Content A{{< /tab >}}
{{< tab name="Option B" >}}Content B{{< /tab >}}
{{< /tabs >}}

{{< mermaid >}}
graph LR; A-->B;
{{< /mermaid >}}
```

### Build Verification Command

```powershell
hugo --config config/development/hugo.toml --destination tmp-hugo-build
Remove-Item -Recurse -Force tmp-hugo-build
```

---

## 12. Error Recovery

### Playwright Session Lost

If browser is closed or session expires:
1. Re-open Playwright: `playwright_browser_navigate("<workshop_url>")`
2. Check if re-login is needed (snapshot the page)
3. If login required: follow Phase 0 procedure
4. Resume from last incomplete task in roadmap

### Signed URL Expired

If image downloads fail with 403:
1. Navigate to any workshop page that contains images
2. Re-extract signed URL params from `<img>` elements using the evaluate script in Phase 4
3. Update the signed params and retry download
4. If all params expired: use the lightbox fallback method described in Phase 4

### Hugo Build Fails

1. Read error output carefully
2. Common issues:
   - Unclosed shortcode: check `{{</* */>}}` / `{{% %}}` balance
   - Invalid front matter: check YAML syntax
   - Missing ref target: ensure referenced page exists
3. Fix and re-run build before proceeding

### Context Window Running Low

1. Update `WSD.roadmap.md` with current progress
2. Summarize what's been done in the roadmap
3. The next agent session can resume by reading `WSD.roadmap.md`

### Page Content Too Large for Single Context

1. Extract content in sections (by heading)
2. Write markdown incrementally using `fs_append`
3. Handle tabs in a separate pass after base content is written

---

## Execution Order Summary

```
┌─────────────────────────────────────────────────────────┐
│ PHASE 0: Authentication                                 │
│   Navigate → Check access → Login if needed → Confirm   │
└─────────────────────┬───────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────┐
│ PHASE 1: TOC Discovery (PRE-PROCESSING)                 │
│   Snapshot sidebar → Extract page list → Create roadmap │
└─────────────────────┬───────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────┐
│ PHASE 2: Repo Restructure (PRE-PROCESSING)              │
│   Delete old content → Create directories → Verify build│
└─────────────────────┬───────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────┐
│ PHASE 3-5: Per-Page Pipeline (MAIN PROCESSING)          │
│   For each page in roadmap:                             │
│     3. Extract content (EN) ─── main thread             │
│     4. Download images ──────── main thread             │
│     5. Translate (VI) ───────── subagent                │
│   Update roadmap after each page                        │
└─────────────────────┬───────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────┐
│ PHASE 6: Verification (POST-PROCESSING)                 │
│   For each page:                                        │
│     Compare Hugo md ↔ Workshop live ─── subagent        │
│     Fix discrepancies                                   │
│     Final Hugo build                                    │
└─────────────────────────────────────────────────────────┘
```

### Per-Page Pipeline Detail

```
Page N:
  ├── [Main] Navigate to page
  ├── [Main] Snapshot default state → write EN content
  ├── [Main] Click all tabs → write tab content
  ├── [Main] Extract image URLs
  ├── [Main] Download each image → copy to static/
  ├── [Main] Insert image references in EN markdown
  ├── [Subagent] Translate → create VI markdown
  ├── [Main] Update roadmap: mark page complete
  └── Next page...
```

---

## Resumability

This workflow is designed for stateless agents. If context is lost:

1. Read `WSD.roadmap.md` to understand current progress
2. Read `WSD.md` (this file) to understand the workflow rules
3. Read `AGENTS.md` for Hugo conventions
4. Resume from the first incomplete task in the roadmap

The roadmap is the **single source of truth** for progress. Always update it after completing any task.
