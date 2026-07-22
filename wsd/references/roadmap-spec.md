# Roadmap File Specification

## File: `.wsd/roadmap.md`

Created during Phase 1 and updated throughout execution. Single source of truth for progress.

The roadmap lives inside the `.wsd/` directory (hidden folder at project root).
Ensure `.wsd/` directory exists before writing the roadmap.

## Template

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
| Content (EN) | [ ] | |
| Multi-option blocks (all tabs) | [ ] | Only if page has tabbed panels |
| Images downloaded | [ ] | |
| Vietnamese translation | [ ] | |
| Verification | [ ] | |

(Repeat for each page)

> **Note:** The "Multi-option blocks" task is **conditional** — only include it for pages
> that actually contain tabbed panels. During TOC discovery or first page visit, check
> whether the page has tab/toggle UI elements. If not, omit this row from the page's
> task table to reduce noise.

---

## Issues & Decisions

| # | Issue | Decision | Date |
|---|-------|----------|------|
| 1 | ... | ... | ... |
```

## Task Definitions

| Task | Definition | Complete When |
|------|-----------|---------------|
| Content (EN) | Extract full text from page including all visible states | Full markdown written with all headings, text, code blocks |
| Multi-option blocks | Capture ALL tab/toggle options beyond the default | All tabs converted to `{{< tabs >}}` shortcodes with complete content. **Skip if page has no tabs.** |
| Images downloaded | All `<img>` from page saved to `static/images/workshop/` | Files exist on disk with descriptive names |
| Vietnamese translation | `_index.vi.md` created with full translation | File exists, proper Vietnamese, matches EN structure |
| Verification | Content compared against live workshop | All discrepancies resolved or documented |
| README generated | Workshop-focused README.md at repo root | File exists with workshop overview, lab summaries, learning objectives |

## Weight & Pre Convention

- Homepage: no `pre`, weight irrelevant
- Prerequisites section: weight=1, pre=" <b> 0. </b> "
- Lab N: weight=N+1, pre=" <b> N. </b> "
- Summary: weight=last, pre=" <b> last. </b> "
- Child pages: pre=" <b> N.M. </b> "
