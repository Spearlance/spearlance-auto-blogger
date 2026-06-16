---
name: seo-doctrine
description: The governing SEO and editorial quality rules for all Spearlance blog content. Read fresh on every Auto Blogger run before anything is drafted. Maps the full Spearlance SEO Doctrine to what binds blog posts specifically — structure, sourcing, internal linking, schema, intent, and the kill criteria.
---

# Spearlance SEO Doctrine — Blog Application Layer

This skill is the quality bar every Auto Blogger draft is checked against.

## How to read it

1. **Read `doctrine.md` in this folder IN FULL first** — it is the vendored, machine-readable
   doctrine (18 sections). Never grade a draft from memory; reread the relevant sections each run.
2. Then apply the **blog-specific mapping** below, which tells you which doctrine rules bind a
   *blog post* (the doctrine covers far more — city pages, pillars, citations, backlinks — that
   does not apply to a routine blog post).

## Provenance & sync

- **Source of truth:** `SEO-Doctrine.pdf` (in this folder) → transcribed to `doctrine.md`.
- **Origin:** vendored from the separate `SEO Doctrine` workspace on **2026-06-15** so this
  project is fully self-contained for cloud routine runs.
- **To update:** re-copy `doctrine/doctrine.md` and `doctrine/SEO-Doctrine.pdf` from the
  `SEO Doctrine` workspace. If the PDF and `doctrine.md` ever disagree, the **PDF wins** —
  re-transcribe. Keep `doctrine.md` byte-identical to its source to make diffs trivial.

---

## Blog-Post Application: which doctrine rules bind a blog post

A routine/reactive blog post must satisfy **all** of the following. Citations are to `doctrine.md`.

### Identity & stance (§0)
- Audience = local Concord/NH business owner in our verticals (Construction, Healthcare, Local
  Service). Every post answers **"what this means for you,"** not "what happened this week."
- Core beliefs apply: **every sentence must earn its place** (§0.4); interlink aggressively and
  contextually (§0.5); schema is mandatory (§0.6); never optimize blindly — context governs (§0.7).

### Meta title (§1, blog branch of Rule 1.1)
- **Primary keyword phrase first.** City only if locally relevant. **Brand name ("Spearlance" /
  "Spearlance Media") last.** Do not force the city-embedded service-page pattern onto a blog title.
- CTR modifier (§1.2) allowed in the first 60 chars **only if truthfully claimable**.
- Context safeguard (§1.3): never lead the title with a keyword that doesn't match the post's intent.

### Meta description (§2.1)
- Open with the primary keyword phrase; read naturally (not a keyword dump); include one
  credibility trigger; end with a CTA; never duplicate the title verbatim.

### Headings (§3)
- Exactly **one H1** containing the exact primary keyword phrase (§3.1).
- **H2 for major sections, H3 for subsections; never skip levels** (§3.4). Include keyword
  variations in H2s where natural.

### Content standards (§6) — the hard blog gates
- **Word count: minimum 1,000; target 1,500–2,000. Below 800 = do not ship / rewrite** (§6.1).
- **H2 headings: ≥4 (target 6–8); below 3 fails** (§6.1).
- Approved content types only, ranked (§6.2): high-intent informational, legal/regulatory updates,
  strategic how-to breakdowns, industry updates, data-backed listicles.
- **NEVER publish** (§6.2): magazine-style filler with no ranking intent; seasonal fluff with no
  keyword; content that rehashes existing site content without adding depth; unreviewed AI content.
- If the post is a refresh of an existing post, deepen it — do not duplicate (ties to §13, §18.1).

### Interlinking (§7) — the hard blog gates
- **Minimum 5 internal links per post; target 10+** (§7.1, §6.1). Link the first occurrence of any
  keyword that matches an existing service/city/post URL.
- Anchor distribution (§7.2): ~30–40% exact, ~30–40% partial, ~20–30% branded/natural.
- Linking patterns (§7.3): mention a service keyword → link the service page; mention a city →
  link its city page if it exists; reference a pillar topic → link the pillar. One link per URL,
  first occurrence only.

### Schema (§8)
- **Blog posts require Article schema** (§8.1). Add **FAQ schema** when the post contains a genuine
  FAQ block. Schema must accurately reflect the content (§8.2, §18.2) — never fabricate it.

### Intent preservation (§13) — safety
- A blog post **stays a blog post** (§13.1). Service CTAs are fine; full service-page treatment is
  not. If a keyword would redefine the post's purpose, it belongs on a new page — flag, don't force.

### Behavioral rules (§18) & escalation (§14.3)
- Always: check existing pages before creating (avoid duplication); validate schema; never publish
  without factual review (§18.1, §18.2).
- **Escalate to Garrett, do not act, when** (§14.3): a 301 redirect, a deletion, cannibalization
  between important pages, or any irreversible change is implied. A routine blog post should rarely
  trip these — if it does, stop and flag.
- When in doubt, choose the **more conservative** option and flag for review (§18.3).

## Doctrine-compliance check (do this in QA, every draft)

Produce a short rule-by-rule verdict before a post is eligible to publish:

```
§1 Title       — PASS/FAIL — note
§2 Description  — ...
§3 Headings     — ...
§6 Content (wordcount, ≥4 H2, not on NEVER list) — ...
§7 Interlinking (≥5 links, anchor mix) — ...
§8 Schema (Article [+FAQ]) — ...
§13 Intent (still a blog post) — ...
§14.3 Escalation triggers — none / ESCALATE
VERDICT: SHIP / FIX FIRST / ESCALATE
```

Any FAIL on a hard gate (§6 word count/H2, §7 link minimum, §8 schema, any unsourced claim) →
the post does **not** ship. A missed day is acceptable; weak content is not.
