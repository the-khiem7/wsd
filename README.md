# WSD - Workshop Scraping & Documentation

## Installation

```bash
npx skills add the-khiem7/wsd
```

---

A skill that automatically scrapes **AWS Workshop Studio** workshops and converts them into bilingual (English/Vietnamese) Hugo documentation sites.

## Features

- Automated workshop authentication (headed browser — supports manual login)
- Full Table of Contents discovery from sidebar navigation
- Complete content extraction including **all tabbed panels** (nested tabs supported)
- Image download from CloudFront signed URLs (bypasses CORS restrictions)
- Vietnamese translation with proper diacritics
- Verification against live workshop content
- Workshop-focused README generation with key images
- Resumable workflow — progress tracked via roadmap file, can resume from any point

## Pipeline

```
Phase 0: Authentication       → Open browser, manual login if required
Phase 1: TOC Discovery        → Extract page list from sidebar
Phase 2: Repo Restructure     → Create Hugo directory structure from TOC
Phase 3: Content Extraction   → Extract EN content (all tabs, all code blocks)
Phase 4: Image Download       → Download images via signed URL / screenshot
Phase 5: Vietnamese Translation → Translate to Vietnamese (subagent)
Phase 6: Verification         → Compare Hugo output ↔ live workshop
Phase 7: README Generation    → Generate workshop-focused README with images
```

## Requirements

- **Playwright MCP** (`@playwright/mcp`) — headed mode (default)
- **Hugo extended** — static site generator
- **Subagent capability** — for parallel translation/verification (fallback: sequential)

## Usage

After installing the skill, provide the workshop URL to the agent:

> "Scrape this workshop into a Hugo site: https://catalog.us-east-1.prod.workshops.aws/workshops/..."

The agent will execute the full pipeline automatically. You only need to log in manually when prompted.

## Playwright MCP Configuration

Add to `.kiro/settings/mcp.json` (or equivalent on your platform):

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
        "browser_close"
      ]
    }
  }
}
```

> **Note:** DO NOT pass `--headless false` — headed is the default. Only add `--headless` (no value) if you want headless mode.

## Platform Compatibility

| Platform        | Delegation                                    |
| --------------- | --------------------------------------------- |
| Kiro            | `invoke_sub_agent` — full parallel support |
| Claude Code     | `Task` tool or sequential                   |
| Cursor/Windsurf | Composer agent or sequential                  |
| Other           | Sequential on main thread                     |

## License

MIT
