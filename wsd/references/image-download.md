# Image Download — Detailed Procedure

## Background

Workshop images are hosted on CloudFront CDN (`static.us-east-1.prod.workshops.aws`) with
signed URLs. Standard methods (`fetch()`, `canvas.toDataURL()`) fail due to CORS/tainted
canvas restrictions.

**Proven method:** Navigate directly to the signed image URL, then screenshot the rendered
`<img>` element.

## Per-Page Procedure

### Step 1: Navigate to Workshop Page

```
playwright_browser_navigate("<workshop_base_url>/<page_path>")
playwright_browser_wait_for(time: 4)
```

### Step 2: Extract Image URLs

```javascript
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
```

⚠️ **CRITICAL:** Must replace `%7E` → `~` in URLs.
Browser encodes tilde as `%7E` in `img.src` but CloudFront signed URLs require literal `~`.

### Step 3: Extract Signed URL Parameters (first time only)

From any image URL, extract:
- `Key-Pair-Id=...`
- `Policy=...`
- `Signature=...`

These params typically use a wildcard policy covering ALL images in the workshop.
Save them for reuse across all image downloads.

### Step 4: Download Each Image

For EACH image:

**a. Determine descriptive filename:**
- Pattern: `<page-slug>_<description>.png`
- Example: `lab1_architecture_diagram.png`, `lab3_gateway_sequence.png`
- Name describes WHAT the image shows, not its position

**b. Navigate directly to signed image URL (with ~ decoded):**
```
playwright_browser_navigate("<image_url_with_tildes>")
```
- Console may show 403 ERROR — IGNORE IT (from subsidiary resources, not the image)
- Page title will show "filename.png (WxH)" confirming image loaded successfully

**c. Screenshot the image element:**
```
playwright_browser_take_screenshot({
  element: "<descriptive name>",
  ref: "e2",
  filename: "<descriptive_filename>.png"
})
```
- When navigating directly to an image URL, the page contains only 1 img element
- ref is typically "e2" but verify via snapshot if needed

**d. Copy file to project:**
```powershell
Copy-Item "<playwright_temp_output_path>/<filename>.png" "static/images/workshop/<filename>.png"
```

### Step 5: Update Markdown

Replace image placeholder comments:
```
<!-- IMAGE: description --> → ![description](/images/workshop/<filename>.png)
```

### Step 6: Update Roadmap

Mark "images downloaded" and "images referenced" tasks as complete.

---

## Image Naming Convention

| Pattern | Example |
|---------|---------|
| Architecture diagrams | `<page>_architecture_diagram.png` |
| UI screenshots | `<page>_<ui_element>_screenshot.png` |
| Terminal output | `<page>_terminal_<action>.png` |
| Config files | `<page>_<config_name>_config.png` |
| IDE/editor views | `<page>_<tool>_<context>.png` |

---

## Signed URL Technical Details

Base URL pattern:
```
https://static.us-east-1.prod.workshops.aws/<workshop-uuid>/static/<section-folder>/<image-name>.png
```

Signed URL components as query params:
```
?Key-Pair-Id=<id>&Policy=<base64>&Signature=<sig>
```

Key facts:
- Policy typically contains a wildcard resource (`/*`) covering all images
- All images in the same workshop share the same signed params
- Policy has a `DateLessThan` expiry — check the epoch timestamp
- Signature contains `~` characters — must NOT be URL-encoded

---

## Methods That Do NOT Work

| Method | Error |
|--------|-------|
| `fetch()` in page context | CORS block (cross-origin) |
| `canvas.toDataURL()` | Tainted canvas (cross-origin image) |
| Navigate without signed params | 403 Missing Key-Pair-Id |
| Navigate with `%7E` (not `~`) | 403 Invalid signature |
| `Invoke-WebRequest` / `curl` | May work but signed URLs from browser more reliable |

---

## Fallback Method: Lightbox Screenshot

If signed URL approach fails (expired, changed):

1. Resize browser: `playwright_browser_resize({ width: 2560, height: 1440 })`
2. Navigate to the workshop page containing the image
3. Click the image → opens lightbox/fullscreen dialog
4. Screenshot the img element inside the lightbox dialog
5. Close lightbox: click "Minimize image" or press Escape
6. Copy file to `static/images/workshop/`

Quality is lower than direct signed URL method but text remains readable.
