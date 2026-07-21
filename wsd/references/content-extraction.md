# Content Extraction — Detailed Procedure

## Per-Page Procedure

### Step 1: Navigate

```
playwright_browser_navigate("<workshop_base_url>/<page_path>")
playwright_browser_wait_for(time: 4)
```

### Step 2: Capture Default State

```
playwright_browser_snapshot()
```

Record all text content, headings, code blocks, callouts, tables.

**Tip for large pages:** When the article text exceeds ~5000 characters, use the `filename`
parameter on `browser_evaluate` to save content directly to a file instead of returning it
into agent context:

```javascript
playwright_browser_evaluate({
  filename: ".playwright-mcp/page-content.txt",
  function: `() => document.querySelector('article').innerText`
})
```

Then read the file separately. This avoids bloating agent context with raw text that will
be reformatted into markdown anyway.

### Step 3: Identify Multi-Option Blocks

Look for tab/toggle UI elements (e.g. "AgentCore CLI | Interactive", "macOS/Linux | Windows").
Record which tab is currently active (this is "Option 1") and its content.

### Step 4: Capture Remaining Tab Options

For EACH inactive tab:

```
a. playwright_browser_click({ element: "<tab label>", ref: "<ref>" })
b. playwright_browser_snapshot()
c. Record content of this tab option
```

Note: Tabs can be NESTED (outer tab → inner sub-tabs). Capture ALL combinations.

### Step 5: Write English Markdown

Write `content/<path>/_index.md` with:
- Proper front matter (title, weight, pre, date, chapter)
- Workshop elements converted to Hugo shortcodes
- ALL code blocks preserved exactly
- ALL CLI commands with full flags
- Image placeholders: `<!-- IMAGE: description -->`

### Step 6: Update Roadmap

Mark content, multi-option blocks, and code blocks tasks as complete.

---

## Large Page Strategy — Subagent Delegation

When a workshop page is too large for a single agent context to handle without
risk of dropping content, delegate extraction to subagents:

### When to Delegate

- Page has more than 5 tabbed panels
- Multiple nested tab groups (tabs within tabs)
- Page snapshot exceeds ~8000 words of visible text
- Code blocks totaling more than 200 lines
- Agent context is already heavy from prior pages

### How to Delegate

1. **Main thread** navigates to the page and takes initial snapshot
2. **Main thread** identifies section boundaries (headings, tab groups)
3. **Subagent A** extracts sections 1-N (e.g., intro + first 3 tab groups)
4. **Subagent B** extracts sections N+1-M (e.g., remaining tab groups)
5. **Main thread** assembles all sections into final `_index.md`
6. **Main thread** verifies no gaps between section boundaries

### Subagent Brief for Partial Extraction

```
TASK: Extract partial content from workshop page
URL: <workshop_url>/<page_path>
SCOPE: Extract content from heading "<start_heading>" through heading "<end_heading>"
       Include ALL tabs within this scope
OUTPUT: Write to content/<path>/_section_<N>.md (temp file)
RULES:
- Navigate to page, click relevant tabs, capture ALL content in scope
- Preserve code blocks exactly, never truncate
- Use Hugo shortcodes for tabs, notices, etc.
- Do NOT add front matter (main thread handles assembly)
```

### Assembly

After all subagents complete, main thread:
1. Creates `_index.md` with front matter
2. Concatenates section files in order
3. Removes temp `_section_*.md` files
4. Verifies no duplicate or missing content at section boundaries

---

## Multi-Option Block Rules

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

---

## Content Fidelity Rules

- **NEVER** truncate code blocks with `...` or `# ...rest`
- **NEVER** summarize — preserve full instructional text
- **NEVER** add the page title as H1/H2 in body (front matter `title` renders it)
- **ALWAYS** preserve exact CLI commands with all flags and arguments
- **ALWAYS** preserve table data completely
- **ALWAYS** convert callout/alert boxes to `{{% notice %}}` shortcodes

---

## Hugo Front Matter Template

```yaml
---
title: "Page Title"
date: 2026-01-01
weight: 1
chapter: false
pre: " <b> 1. </b> "
---
```

---

## Shortcode Quick Reference

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

---

## Image Paths

```markdown
![Description](/images/workshop/filename.png)
```

All workshop images stored in `static/images/workshop/` and referenced with root-relative paths.

---

## Internal Links

```markdown
[Link text]({{< ref "path/to/page" >}})
```
