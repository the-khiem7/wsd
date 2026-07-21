# Error Recovery

## Playwright Session Lost

If browser is closed or session expires:

1. Re-open Playwright: `playwright_browser_navigate("<workshop_url>")`
2. Check if re-login is needed (snapshot the page)
3. If login required: follow Phase 0 authentication procedure
4. Resume from last incomplete task in roadmap

## Signed URL Expired

If image downloads fail with 403:

1. Navigate to any workshop page that contains images
2. Re-extract signed URL params from `<img>` elements:

```javascript
() => {
  const imgs = document.querySelectorAll('article img');
  return Array.from(imgs).map(img => ({
    src: img.src.replace(/%7E/g, '~'),
    alt: img.alt,
    w: img.naturalWidth,
    h: img.naturalHeight
  }));
}
```

3. Update the signed params and retry download
4. If all params expired: use the lightbox fallback method (see image-download.md)

## Hugo Build Fails

1. Read error output carefully
2. Common issues:
   - **Unclosed shortcode:** check `{{< >}}` / `{{% %}}` balance
   - **Invalid front matter:** check YAML syntax
   - **Missing ref target:** ensure referenced page exists
3. Fix and re-run build before proceeding

## Context Window Running Low

1. Update `.wsd/roadmap.md` with current progress
2. Summarize what's been done in the roadmap
3. The next agent session can resume by reading `.wsd/roadmap.md`

## Page Content Too Large for Single Context

1. **Preferred:** Delegate to subagents — split by sections/headings, each subagent extracts
   a portion, main thread assembles (see content-extraction.md "Large Page Strategy")
2. If subagents unavailable: extract content in sections (by heading)
3. Write markdown incrementally using `fs_append`
4. Handle tabs in a separate pass after base content is written

## Workshop URL Patterns

- Standard: `https://catalog.us-east-1.prod.workshops.aws/workshops/<id>/...`
- Event-based: `https://catalog.us-east-1.prod.workshops.aws/event/dashboard/en-US/workshop/...`
