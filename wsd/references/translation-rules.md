# Vietnamese Translation Rules

## Overview

Create Vietnamese translation for every English page. This phase is delegated to a
subagent when possible.

## Task Brief Template

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

## Translation Matrix

| Element | Translate? | Notes |
|---------|-----------|-------|
| Body text, headings | ✅ Yes | Proper Vietnamese with diacritics |
| Image alt text | ✅ Yes | |
| `title` in front matter | ✅ Yes | |
| Code blocks | ❌ No | Keep English exactly |
| CLI commands | ❌ No | Keep English exactly |
| File/directory names | ❌ No | |
| Shortcode names/params | ❌ No | Translate content inside only |
| Tab names | ✅ Yes | "macOS/Linux" stays, descriptive names translate |
| Notice type keywords | ❌ No | `info`, `warning`, `tip` stay English |

## Quality Standards

- Use proper Vietnamese diacritics (ă, â, đ, ê, ô, ơ, ư, etc.)
- Never use ASCII transliteration
- Technical terms may remain in English if no standard Vietnamese equivalent exists
- Keep the same paragraph structure as English version

## Front Matter Rules

Vietnamese file keeps identical front matter except `title` is translated:

**English:**
```yaml
---
title: "Prerequisites"
date: 2026-01-01
weight: 1
chapter: false
pre: " <b> 0. </b> "
---
```

**Vietnamese:**
```yaml
---
title: "Điều kiện tiên quyết"
date: 2026-01-01
weight: 1
chapter: false
pre: " <b> 0. </b> "
---
```

## Subagent Instructions

Full instructions for the translator subagent:

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

## Fallback

If the platform does not support subagents, the main thread executes translation
directly after completing content + images for the page.
