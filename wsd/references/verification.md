# Verification — Detailed Procedure

## Goal

Verify that Hugo content matches the original workshop content via text comparison.

## Per-Page Procedure

### Step 1: Navigate

```
playwright_browser_navigate("<workshop_url>/<page_path>")
playwright_browser_wait_for(time: 4)
```

### Step 2: Capture Full Text

```
playwright_browser_snapshot()
```

Focus on article/main content area. Also click through ALL tabs to capture their content.

### Step 3: Read Local Markdown

```
read_file("content/<path>/_index.md")
```

### Step 4: Compare

Check for:
- Missing sections or headings
- Missing or truncated code blocks
- Missing CLI commands
- Missing table rows/columns
- Missing callout/notice content
- Missing tab/option content (ALL tabs must be present)
- Incorrect ordering of steps
- Missing image references

### Step 5: Fix Discrepancies

- Update `_index.md` (EN) with missing content
- Delegate VI update to subagent (re-translate affected sections)

### Step 6: Verify Hugo Build

```powershell
hugo --config config/development/hugo.toml --destination tmp-hugo-build
Remove-Item -Recurse -Force tmp-hugo-build
```

### Step 7: Update Roadmap

Mark "verified" for this page. Note what was fixed (or "OK — no issues").

---

## Verification Checklist

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

## Subagent Verification Brief

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
