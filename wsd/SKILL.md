---
name: workshop-scraping-docs
description: >
  Scrape an AWS Workshop Studio workshop and convert it into a bilingual (English/Vietnamese)
  Hugo documentation site. Handles authentication via headed Playwright, extracts full page content
  including tabbed panels, downloads signed-URL images, translates to Vietnamese, and verifies
  output against live workshop. Use when the user provides an AWS Workshop Studio URL and wants
  it converted to a Hugo static site with EN/VI bilingual content.
license: MIT
compatibility: >
  Requires Playwright MCP (@playwright/mcp, headed by default), Hugo extended, and subagent
  capability for parallel translation/verification. Windows PowerShell or cmd environment.
metadata:
  author: wsd-project
  version: "1.0"
---

# Workshop Scraping & Documentation Workflow

Complete workflow for a stateless agent to scrape an AWS Workshop Studio workshop
and convert it into a bilingual Hugo documentation site.

## Key Files

| File | Role |
|------|------|
| `.wsd/roadmap.md` | Generated during execution — tracks progress per page |
| `AGENTS.md` | Hugo repo conventions (front matter, shortcodes, structure) |

## Required Tools

- **Playwright MCP** — Browser automation, runs headed by default (DO NOT pass `--headless false`)
- **Hugo (extended)** — Static site generator
- **Subagent capability** — Delegate translation & verification (fallback: sequential)

## Playwright MCP Configuration

Playwright MCP runs in **headed** mode (visible browser window) by default.
**DO NOT** pass `--headless false` — the CLI treats `--headless` as a boolean flag and
`"false"` becomes a stray positional argument causing `too many arguments` error.

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"],
      "disabled": false,
      "autoApprove": [
        "browser_navigate",
        "browser_take_screenshot",
        "browser_snapshot",
        "browser_wait_for",
        "browser_click",
        "browser_evaluate",
        "browser_resize",
        "browser_close",
        "browser_fill_form",
        "browser_type"
      ]
    }
  }
}
```

> **Note:** Use `@playwright/mcp@latest` (not `@anthropic-ai/playwright-mcp`).
> Headed is the default — only add `--headless` (no value) if you want headless mode.

### `.playwright-mcp` Cleanup

Playwright MCP creates a `.playwright-mcp/` folder in the workspace during execution.
After first use, check `.gitignore` — if `.playwright-mcp` is not listed, add it:

```
# Playwright MCP temp data
.playwright-mcp/
```

## Execution Pipeline

```
Phase 0: Authentication
  → Navigate to workshop URL → Check access → Manual login if needed

Phase 1: TOC Discovery
  → Snapshot sidebar → Extract page list → Create .wsd/roadmap.md

Phase 2: Repo Restructure
  → Delete old content/ → Create directories from TOC → Verify Hugo build

Phase 3-5: Per-Page Pipeline (for each page in roadmap)
  → [Main] Extract content (EN) with all tabs
  → [Main] Download images via signed URL screenshot
  → [Subagent] Translate to Vietnamese
  → Update roadmap after each page

Phase 6: Verification
  → [Subagent] Compare Hugo markdown ↔ live workshop
  → Fix discrepancies → Final Hugo build
```

## Phase 0: Authentication

1. Navigate to workshop URL, wait 5 seconds
2. Snapshot to check state
3. If login/sign-in form visible:
   - STOP and inform user: "Workshop requires authentication. Please complete login manually. Reply 'continue' when done."
   - Wait for user response, then re-snapshot to confirm
4. If workshop content visible → proceed to Phase 1

**CRITICAL:** NEVER close the Playwright browser after login — session will be lost.

## Phase 1: TOC Discovery

1. Navigate to workshop homepage
2. Snapshot and identify sidebar navigation
3. Extract page titles, URL paths, nesting hierarchy
4. Use JavaScript evaluation for extraction:

```javascript
() => {
  const links = document.querySelectorAll('nav a, [class*="sidebar"] a, [class*="navigation"] a');
  return Array.from(links).map(a => ({
    title: a.textContent.trim(),
    href: a.getAttribute('href'),
    depth: /* calculate from nesting */
  }));
}
```

5. Create `.wsd/roadmap.md` — see [references/roadmap-spec.md](references/roadmap-spec.md)

## Phase 2: Repo Restructure

1. DELETE all files under `content/` (will be recreated)
2. KEEP: `config/`, `layouts/`, `themes/`, `static/`, `archetypes/`, `.github/`, `AGENTS.md`
3. CREATE directory structure from TOC: `content/<slug>/_index.md` + `_index.vi.md`
   - `_index.md`: placeholder with front matter only (body will be filled in Phase 3)
   - `_index.vi.md`: placeholder with front matter only (body will be filled in Phase 5)
   - Hugo requires `.vi.md` files to exist for Vietnamese pages to appear in navigation;
     creating them early ensures Hugo build passes at every phase
4. CREATE `static/images/workshop/`
5. CREATE `.wsd/` directory (for roadmap file)
6. CHECK `.gitignore` — ensure `.playwright-mcp/` is listed; add if missing
7. Verify Hugo build — use the project's actual config path (check `config/` directory structure):

```powershell
# Common patterns — pick the one that matches the project:
hugo --config config/_default/hugo.toml --destination tmp-hugo-build
# OR: hugo --config config/development/hugo.toml --destination tmp-hugo-build
Remove-Item -Recurse -Force tmp-hugo-build
```

Directory naming: lowercase, hyphenated, match workshop URL slug when possible.

## Phase 3: Content Extraction

For each page — see full procedure in [references/content-extraction.md](references/content-extraction.md).

Key rules:
- **NEVER** truncate code blocks
- **NEVER** summarize instructional text
- **NEVER** add page title as H1/H2 in body (front matter handles it)
- **ALWAYS** capture ALL tab options in tabbed panels
- **ALWAYS** preserve exact CLI commands with all flags
- **ALWAYS** convert callout boxes to `{{% notice %}}` shortcodes

### Large Page Handling

When a workshop page contains too much content for a single agent context (many tabs,
long code blocks, extensive instructions), **delegate content extraction to a subagent**:

- Split the page into logical sections (by heading or tab group)
- Each subagent extracts one section completely, writing to a temp file
- Main thread assembles sections into the final `_index.md`
- This ensures NO content is dropped or truncated due to context limits

Signs a page needs delegation:
- More than 5 tabbed panels
- Multiple nested tab groups
- Page snapshot exceeds ~8000 words of visible text
- Code blocks totaling more than 200 lines

### Context Budget Strategy

A typical workshop has 15-25 pages. The main thread cannot hold all pages in context
simultaneously. Follow this batching strategy:

- **Extract pages sequentially** on the main thread, writing each file immediately after extraction
- **After ~6-8 pages extracted**, delegate the next batch to a subagent if context is getting heavy
- **Translation and verification** should ALWAYS be delegated in batches (5-8 pages per subagent)
- **Update roadmap after each page** so progress survives context compaction

### Playwright `evaluate` for Large Pages

When extracting content from pages with >5000 characters of article text, use the
`filename` parameter on `browser_evaluate` to save output directly to a file instead
of returning it into agent context:

```javascript
playwright_browser_evaluate({
  filename: ".playwright-mcp/page-content.txt",
  function: `() => document.querySelector('article').innerText`
})
```

Then read the file with `read_file`. This avoids bloating the agent context with raw
page text that will be reformatted anyway.

### Workshop Element → Hugo Shortcode Mapping

| Workshop Element | Hugo Shortcode |
|-----------------|----------------|
| Tabbed panel | `{{< tabs >}}` / `{{< tab name="..." >}}` |
| Callout/alert | `{{% notice info/warning/tip %}}` |
| Expandable section | `{{% expand "Title" %}}` |
| Internal link | `{{< ref "path" >}}` |
| Code block | Fenced markdown code block |
| Mermaid diagram | `{{< mermaid >}}` (NEVER fenced code block) |

## Phase 4: Image Download

See full procedure in [references/image-download.md](references/image-download.md).

Key facts:
- Images hosted on CloudFront with signed URLs — standard fetch/canvas methods fail
- **Primary method:** `Invoke-WebRequest`/`curl` with signed params — fastest for batch download
- **Fallback method:** Navigate directly to signed URL via Playwright, then screenshot `<img>` element
- CRITICAL: Replace `%7E` → `~` in extracted URLs
- Signed params (Key-Pair-Id, Policy, Signature) are reusable across all images
- Naming: prefer original CDN filename when descriptive; rename only if opaque

## Phase 5: Vietnamese Translation

Delegated to subagent when possible. See [references/translation-rules.md](references/translation-rules.md).

Task brief for subagent:
```
TASK: Translate content/<path>/_index.md to Vietnamese
OUTPUT: content/<path>/_index.vi.md
RULES:
- Translate: title, headings, body text, image alt text
- Keep unchanged: code blocks, CLI commands, file names, shortcode syntax, image paths
- Use proper Vietnamese diacritics (ă, â, đ, ê, ô, ơ, ư)
```

## Phase 6: Verification

Delegated to subagent. See [references/verification.md](references/verification.md).

Compare Hugo markdown against live workshop for each page. Check for missing sections,
truncated code blocks, missing tabs, missing images. Fix discrepancies and re-verify build.

## Task Delegation Strategy

| Role | Thread | Notes |
|------|--------|-------|
| Orchestrator | Main | Auth, TOC, restructure, content, images, roadmap |
| Translator | Subagent | Isolated — only needs EN file |
| Verifier | Subagent | Independent per page after content+images done |

Platform delegation:
- **Kiro**: `invoke_sub_agent({ name: "general-task-execution", prompt: "<brief>" })`
- **Claude Code**: Use `Task` tool or run sequentially
- **Fallback**: Main thread executes all tasks sequentially

## Error Recovery

See [references/error-recovery.md](references/error-recovery.md) for detailed procedures.

- **Session lost**: Re-navigate, re-login if needed, resume from roadmap
- **Signed URL expired**: Re-extract params or use lightbox fallback
- **Hugo build fails**: Check shortcode balance, YAML syntax, missing refs
- **Context window low**: Update roadmap with progress, next session reads roadmap to resume

## Resumability

If context is lost:
1. Read `.wsd/roadmap.md` for current progress
2. Read this skill for workflow rules
3. Read `AGENTS.md` for Hugo conventions
4. Resume from first incomplete task in roadmap

The roadmap is the **single source of truth** for progress.
