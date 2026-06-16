# Spearlance Auto Blogger

Self-contained daily blogging routine for **www.spearlance.com**, run inside Claude Code.
It reads the doctrine, audits existing content, checks reactive feeds, researches keywords,
drafts source-backed posts, QAs them against the SEO Doctrine, and publishes through Duda —
all in one routine, with no external feed reader, scheduler service, or aggregator.

**The bar is authority, not volume.** Every post earns its place or it does not ship.
Missing a day is acceptable. Shipping weak content is not.

## Business

- **Brand:** Spearlance Media — web design & digital marketing agency
- **Location:** 244-246 North Main Street, STE 101, Concord, NH 03301 · (603) 821-9533 · hello@spearlance.com
- **Audience:** Local Concord / NH small business owners, plus the verticals below
- **Verticals:** Construction & home services · Medical & clinics · General local service businesses · Agencies / web professionals

## Duda

- **Site name (ID):** `e8531a5e`
- **Live domain:** https://www.spearlance.com
- **Blog name:** "Resources" (post paths live under `/blog/...`, plus `/guides`, `/resources`)
- **Author byline used historically:** Garrett Handley

## Key service / landing pages (internal-link targets)

Pull the live sitemap (`https://www.spearlance.com/sitemap.xml`) each run for the current set. Baseline:

- Search Engine Marketing — `/search-engine-marketing`
- Local Lead Generation — `/local-lead-generation`
- Fractional Marketing — `/fractional-marketing`
- AI for Local Business — `/ai-for-local-business`
- Duda Agency White Label — `/duda-agency-white-label`
- Our Work — `/our-work` · About — `/about-us` · Contact — `/contact` · Consultation — `/consultation`

> ⚠️ The original spec referenced "Web Design / SEO / Google Ads / Custom Development" service pages.
> The live site does **not** have those exact pages — SEO + paid search are combined under
> `/search-engine-marketing`, and web design is sold via per-city location pages. Link to what
> actually exists; never invent a service URL.

## Project layout

- `routine.md` — the daily run playbook. **This is what the scheduled routine executes.**
- `feeds.yaml` — monitored reactive sources (verified URLs).
- `.claude/skills/seo-doctrine/` — the governing quality rules, read fresh every run:
  - `SKILL.md` — blog-application layer (which doctrine rules bind a blog post).
  - `doctrine.md` — full vendored doctrine (18 sections). `SEO-Doctrine.pdf` — source of truth.
  - Vendored 2026-06-15 from the separate `SEO Doctrine` workspace so this project is
    self-contained for cloud runs. To update, re-copy both files; the PDF wins on any conflict.
- `ledger/content-log.md` — every post produced (seeded with the existing inventory).
- `ledger/processed-feeds.jsonl` — feed items already evaluated, for reactive dedup.

> Doctrine note: the doctrine names spearlance.com a "custom-coded site." It is in fact the Duda
> site `e8531a5e`; blog posts publish through the Duda connector. This doesn't affect the blog
> rules, but don't act on the "custom-coded" framing.

## Non-negotiable rules (full text in `routine.md`)

1. No claim without a real, fetched source.
2. Never invent citations — cite only URLs actually fetched and read.
3. No fluff, no filler.
4. Every draft is checked against the SEO Doctrine before it is eligible to publish.
5. Runs daily; publishes only when the bar is cleared.

## Current go-live posture

**HOLD FOR REVIEW.** Until Garrett flips the switch, the routine saves finished posts to Duda as
**UNPUBLISHED (draft)** instead of publishing live, so output can be sanity-checked first.
See `routine.md` → "Publishing mode". Flip to live by changing that one flag.
