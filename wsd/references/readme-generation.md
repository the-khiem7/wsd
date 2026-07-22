# README Generation — Detailed Procedure

## Goal

Create a polished `README.md` at repository root that introduces the **workshop content**
to visitors browsing the repo. This is a content-focused document — it tells people what
the workshop is about, what they'll learn, and what each lab covers.

## What the README Is NOT

- NOT a repo setup guide (no `hugo server`, no `npm install`)
- NOT a project structure overview (no directory trees)
- NOT a contribution guide
- NOT deployment instructions

## What the README IS

A self-contained workshop introduction that answers:
1. What is this workshop about?
2. What will I learn?
3. What technologies/services are covered?
4. What does each lab teach?
5. What are the prerequisites (knowledge-wise, not tooling)?
6. What does the workshop look like? (architecture diagrams, key screenshots)

## Procedure

### Step 1: Gather Source Material

Read all extracted English content files:

```
content/<path>/_index.md (for every page in roadmap)
```

Focus on:
- Workshop homepage/introduction (usually the first page)
- Prerequisites page (knowledge requirements, not setup)
- Each lab's introductory paragraphs
- Learning objectives mentioned anywhere

### Step 2: Identify Key Information

Extract from the content:
- **Workshop title** — from homepage or TOC
- **Workshop purpose** — the "why" (1-2 sentences)
- **Target audience** — who benefits from this workshop
- **Learning objectives** — what skills/knowledge you'll gain
- **Technology stack** — AWS services, tools, frameworks covered
- **Lab summaries** — what each lab teaches (2-4 sentences each)
- **Prerequisites (knowledge)** — what you should already know (NOT tooling setup)
- **Key images** — architecture diagrams, workflow diagrams, key UI screenshots that
  help visitors visually understand the workshop scope

### Step 3: Write README.md

Use this template structure:

```markdown
# <Workshop Title>

<1-2 paragraph overview: what this workshop is about and why it matters>

![Architecture Overview](static/images/workshop/<architecture_diagram>.png)

## What You'll Learn

- <learning objective 1>
- <learning objective 2>
- ...

## Technologies & Services

- <service/tool 1> — brief role description
- <service/tool 2> — brief role description
- ...

## Prerequisites

Before starting, you should be familiar with:
- <knowledge prerequisite 1>
- <knowledge prerequisite 2>
- ...

## Workshop Content

### <Lab 0 / Prerequisites title>

<Brief description of what this section covers>

### <Lab 1 title>

<2-4 sentences: what you'll build/do, key concepts introduced>

![<descriptive alt text>](static/images/workshop/<relevant_image>.png)

### <Lab 2 title>

<2-4 sentences: what you'll build/do, key concepts introduced>

...

### <Summary / Cleanup title>

<Brief description of wrap-up and cleanup>

## Additional Information

- **Duration:** <estimated time if mentioned in workshop>
- **Difficulty:** <level if mentioned>
- **Original workshop:** [AWS Workshop Studio](<workshop_url>)
```

### Image Reference Rules for README

- Use **relative paths** from repo root: `static/images/workshop/<filename>.png`
- Prioritize images that communicate the most at a glance:
  1. **Architecture diagrams** — overall system/service diagram (place near top, after overview)
  2. **Workflow/sequence diagrams** — if they summarize the lab flow
  3. **Key result screenshots** — final output of a lab (e.g., working dashboard, deployed app)
- Do NOT dump every image — pick **1-3 images per lab max**, only those that add understanding
- Every image MUST have a descriptive alt text (accessibility)
- If a lab has no visually meaningful image, skip it — text summary is enough
- The architecture/overview diagram (if one exists) should appear right after the opening
  paragraph, before "What You'll Learn"

### Step 4: Quality Check

- [ ] Reads naturally for someone who has never seen the workshop
- [ ] No repo-specific instructions (build, deploy, run)
- [ ] No directory structure or file references
- [ ] All lab summaries are substantive (not just titles repeated)
- [ ] Technologies section is accurate (only what's actually used)
- [ ] Prerequisites focus on knowledge, not tooling
- [ ] Workshop URL is included for reference
- [ ] Language is inviting and clear
- [ ] Architecture/overview image included near the top (if one exists)
- [ ] Key images per lab are relevant and add visual understanding
- [ ] All image paths are correct relative paths (`static/images/workshop/...`)
- [ ] All images have descriptive alt text
- [ ] Not over-loaded with images (1-3 per lab max)

### Step 5: Update Roadmap

Add to roadmap:

```markdown
### README

| Task | Status | Notes |
|------|--------|-------|
| README.md generated | [x] | Workshop-focused intro |
```

---

## Writing Style Guidelines

- **Tone:** Professional but approachable — like a good conference talk abstract
- **Length:** 150-400 words for overview sections; 2-4 sentences per lab summary
- **Audience:** Developers/architects browsing GitHub who want to know if this workshop
  is worth their time
- **Language:** English (this is the repo README, not a translation target)
- **Verb tense:** Present tense for descriptions ("In this lab, you build..."),
  future tense for objectives ("You will learn how to...")

## Edge Cases

- **Workshop has no clear "learning objectives":** Derive them from what the labs teach
- **Workshop is very short (1-2 pages):** Combine into a single overview paragraph + bullet list
- **Workshop title is generic:** Add a subtitle or clarifying sentence
- **Multiple tracks/paths:** Describe each track briefly, note which one this repo covers

---

## Delegation

This phase runs on the **main thread** — it requires reading multiple content files and
synthesizing them, which is a light task after all extraction is complete. No subagent needed.
