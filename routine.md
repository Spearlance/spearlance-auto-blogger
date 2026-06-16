# Spearlance Auto Blogger — Daily Run Playbook

You are running the Spearlance Auto Blogger. Produce authoritative, source-backed blog content
for **www.spearlance.com** (Duda site `e8531a5e`). Reinforce Spearlance's positioning as a
Concord, NH web design + digital marketing agency serving local businesses and the verticals in
`CLAUDE.md`. **The bar is authority, not volume.** Publishing nothing today is an acceptable
outcome. Publishing weak content is not.

Work the steps in order. Do not skip a gate. When a gate fails, stop the affected branch, log
why, and move on or halt as instructed.

---

## Publishing mode (read this first)

Check the go-live flag in `CLAUDE.md` → "Current go-live posture".

- **HOLD FOR REVIEW** (current default): create finished posts in Duda as **UNPUBLISHED / draft**.
  Do everything else identically (schema, meta, internal links, ledger). Do NOT call the publish
  endpoint. In your end-of-run report, give Garrett the Duda editor/preview link for each draft.
- **LIVE**: publish through Duda so the post goes live, then record the live URL.

Everything below that says "publish" obeys this flag.

---

## Step 1 — Read the doctrine (hard gate)

Read the doctrine skill in full, every run, before anything else:
- `.claude/skills/seo-doctrine/SKILL.md` — the blog-application layer (which rules bind a blog post).
- `.claude/skills/seo-doctrine/doctrine.md` — the full vendored doctrine (18 sections). Reread the
  sections relevant to the draft; never grade from memory.

- If either file is missing or unreadable, **HALT the entire run.** Publish nothing. Report:
  "Doctrine unavailable — halted per Step 1."
- Otherwise hold the doctrine in context. Every draft is checked against it in QA (Step 4).

## Step 2 — Audit existing content

Before writing anything:

1. Pull the live blog inventory from Duda: `list_blog_posts(site_name="e8531a5e", limit=100)`.
2. Read `ledger/content-log.md` for what the routine has already produced and the coverage notes.
3. Fetch `https://www.spearlance.com/sitemap.xml` to refresh the current service/landing pages.

Use all three to: (a) avoid duplicate topics and keyword cannibalization, and (b) build the
internal-linking shortlist — relevant existing posts + the service pages in `CLAUDE.md`. A dated
post (e.g. the 2022 Construction SEO guide) may be *refreshed* but never duplicated.

## Step 3 — Check the reactive feeds

Read `feeds.yaml`. For each source, fetch the feed (or poll the page for the `page-poll` source)
and collect current items. Compare each item URL against `ledger/processed-feeds.jsonl`
(skip `#` comment lines). For any **genuinely new** item, apply the newsworthiness filter:

> **Newsworthiness filter.** Does this materially change how a local business should run its
> website or marketing? If there is no real, concrete implication for our audience, it is NOT a
> post for us — record it as `skipped` and move on.

- If one or more items qualify → run **Branch B** (Step 4a) for the strongest one.
- If nothing qualifies → go to **Branch A** (Step 4b).
- Log every new item you evaluated (qualified or not) to `processed-feeds.jsonl` in Step 5.

---

## Step 4a — Branch B: Reactive post (only if a feed item qualifies)

1. **Verify the trigger (hard gate).** Fetch the actual primary source with WebFetch. Never write
   from a feed snippet, headline, rumor, or secondhand report. If the event cannot be confirmed
   against the primary source, **drop it, log it as skipped, and do not publish.** This gate is the
   entire safety net for auto-publishing — be strict.
2. **Translate, do not report.** The post answers *"what this means for you"* — a local Concord
   business owner — not "here's what happened in AI/SEO this week." No real implication → no post.
3. **Pace by urgency.**
   - Confirmed Google core update / ranking change → time-sensitive, ship **same day**.
   - Frontier model / product announcement → usually an evergreen "how a small business can
     actually use this" angle; it does not need to race out.
4. Go to **Shared drafting → QA → publish** below.

## Step 4b — Branch A: Routine post (if cadence allows and no reactive post took today's slot)

**Cadence:** target **2–3 posts/week**. Before drafting, check `ledger/content-log.md` for how many
routine posts published in the last 7 days. If already at 3, do not force another — stop and report
"cadence met." Reactive (Branch B) posts do **not** count against this cadence.

1. **Research.** Use DataforSEO to surface keyword opportunities filtered against Spearlance
   positioning and the verticals — **not raw volume**. Useful tools:
   - `dataforseo_labs_google_keyword_ideas` / `keyword_suggestions` — expand seed topics.
   - `dataforseo_labs_google_keyword_overview` / `bulk_keyword_difficulty` — volume + difficulty.
   - `dataforseo_labs_search_intent` — confirm intent.
   - `serp_organic_live_advanced` — see who ranks and whether we can credibly beat them.
   Seed from the verticals, the service pages, and gaps in the existing inventory. Default location
   = United States / Concord–NH context where local intent applies.
   Output a **ranked shortlist** with target keyword, intent, difficulty, and angle.
2. **Select and plan.** Pick ONE topic mapped to a target keyword + intent. Confirm against Step 2
   that it does not overlap or cannibalize an existing post. Build a working outline (H2/H3) that
   follows the doctrine's structure rules.
3. Go to **Shared drafting → QA → publish** below.

---

## Shared drafting → QA → publish (both branches)

### Draft
Write to the SEO Doctrine blog-application layer (the `seo-doctrine` skill): title/meta pattern
(§1–2 blog branch), one H1 + ≥4 H2s (§3), 1,000+ words / 1,500–2,000 target (§6.1), 5–10 internal
links with the §7.2 anchor mix, Article (+FAQ) schema (§8). Audience framing is always "what this
means for a local Concord/NH business owner." Internal-link to relevant existing posts (from Step 2)
and the right service page(s). Use the **Garrett Handley** byline unless the doctrine says otherwise.

### QA — every item is a kill gate
Run the doctrine-compliance check from the `seo-doctrine` skill (rule-by-rule verdict), plus the
five non-negotiables:
1. **No claim without a real source.** Every factual/statistical claim has a real, fetched,
   reputable source. If a claim can't be sourced, cut it — do not assert it.
2. **Never invent citations.** Cite only URLs you actually fetched and read this run. A
   plausible-but-fabricated source is a failure, not a near miss.
3. **No fluff, no filler.** Nothing on the doctrine's §6.2 "NEVER publish" list. If a section isn't
   useful and specific to our audience, cut it. Empty word count is worse than no post.
4. **Doctrine adherence.** The compliance check must come back SHIP (no FAIL on any hard gate:
   §6 word count / ≥4 H2, §7 ≥5 links, §8 schema, intent §13).
5. If the draft cannot clear all of the above → **do not publish.** Log the attempt as not-shipped
   with the reason. A missed day is acceptable. If a §14.3 trigger appears → ESCALATE to Garrett.

### Publish (obeying the Publishing-mode flag)
Create the post in Duda (`create_blog_post` for site `e8531a5e`) with:
- Optimized **meta title** and **meta description** per doctrine.
- Appropriate **schema** (Article, plus FAQ where applicable).
- **Internal links** to relevant existing posts and service pages.
- Thumbnail / main image and alt text.
- **HOLD mode:** leave status UNPUBLISHED (draft). **LIVE mode:** publish, then capture the live URL.

## Step 5 — Update the ledger

- Append a full entry to `ledger/content-log.md` (use the template there): date, branch, topic,
  target keyword + intent, URL (+ status), internal links, **every source URL cited**, and any feed
  items processed.
- Append one line per **new** feed item evaluated to `ledger/processed-feeds.jsonl` (qualified or
  skipped) so it never re-triggers.
- If nothing published today, still log the run outcome (cadence met / bar not cleared / doctrine
  halt / no qualifying feed item) so the next run has memory.

## End-of-run report

Summarize for Garrett: which branch ran, what published or why nothing did, draft/live links,
keyword + sources, and feed items processed. In HOLD mode, include Duda preview links for review.
