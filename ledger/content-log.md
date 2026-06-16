# Content Ledger — Spearlance Auto Blogger

The routine's memory across runs. Prevents duplicate topics, keyword cannibalization, and
re-reactions. Every run appends here after publishing. Newest entries go at the top of "Produced
by the routine".

Each routine entry records: date, branch (A routine / B reactive), topic, target keyword + intent,
published URL, sources cited, and any feed items processed.

---

## Produced by the routine

### 2026-06-16 · Branch A · Google Business Profile Optimization
- **Target keyword / intent:** "google business profile optimization" — 6,600/mo, KD 26, LOW competition (informational + commercial)
- **URL:** https://www.spearlance.com/google-business-profile-optimization-the-step-by-step-guide-for-local-service-businesses   [status: UNPUBLISHED — HOLD FOR REVIEW]
- **Duda post ID:** `9f116f8ff00e4f57a1e460f268c60013`
- **Internal links (10):**
  - `/blog/google-maps-pack` — "Google Maps Pack" (exact)
  - `/about-us` — "Spearlance" (branded)
  - `/local-landing-pages-how-to-increase-your-organic-leads-for-life` — "local landing page" (partial)
  - `/blog/customer-reviews` — "4 ways to get more customer reviews" (partial)
  - `/search-engine-marketing` — "search engine marketing" (exact)
  - `/local-lead-generation` — "local lead generation" (exact)
  - `/blog/how-to-automate-your-seo-to-get-more-leads` — "automating your local SEO" (partial)
  - `/consultation` — "schedule a free consultation" (natural)
  - `/blog/keyword-research` — "keyword research" (exact)
  - `/ai-for-local-business` — "AI for local business" (partial)
- **Sources cited:** DataforSEO keyword overview API (June 2026) — keyword volume, difficulty, and intent data for topic selection. All post content is expert guidance; no external statistical claims requiring citation.
- **Feed items processed this run:** All 6 feeds evaluated via WebSearch (WebFetch returned 403 for all URLs in this cloud environment). Google Search Central blog and Search Status feeds surfaced the May 2026 core update (completed June 2) and a Google Search Console AI performance reports announcement (early June 2026). Both dropped per Branch B hard gate — could not verify against primary sources since WebFetch is blocked. OpenAI/Anthropic feeds surfaced Claude Fable 5 and other model launches — no immediate local-business implication strong enough to justify a same-day post. All feed items logged in processed-feeds.jsonl.
- **Notes:** First run of the routine. Cadence was at 0/3 posts this week; this draft re-establishes it. Keyword gap confirmed — no existing GBP optimization content on site. Natural cluster with `/blog/google-maps-pack` (2022) and `/local-lead-generation`. WebFetch environment issue noted: all outbound HTTP fetches return 403 in this cloud run; reactive branch verification is impossible until resolved. FAQ schema included in post HTML as JSON-LD `<script>` block; Duda may strip `<script>` tags — manual verification recommended before publishing.

<!--
Template for a new entry:

### 2026-06-16 · Branch A · <topic>
- **Target keyword / intent:** <keyword> (informational | commercial | transactional | local)
- **URL:** https://www.spearlance.com/blog/<slug>   [status: UNPUBLISHED | PUBLISHED]
- **Internal links:** <pages linked>
- **Sources cited:** <list of fetched URLs>
- **Feed items processed this run:** <none | list — also logged in processed-feeds.jsonl>
- **Notes:** <why it cleared the bar / anything notable>
-->

---

## Pre-existing inventory (baseline as of 2026-06-15)

Imported from Duda site `e8531a5e`. Use to avoid duplication and to find internal-link targets.
Clusters: **PC** platform comparison · **LSEO** local SEO / lead gen · **WD** web design · **VG** vertical guide · **SB** small-business.

| Publish date | Title | Path | Cluster | Status |
|---|---|---|---|---|
| 2025-07-19 | Duda vs. Squarespace: Which Platform Wins for Local Businesses? | `/duda-vs-squarespace-which-platform-wins-for-local-businesses` | PC | PUBLISHED |
| 2025-04-04 | Local Landing Pages: How to Increase Your Organic Leads for Life | `/local-landing-pages-how-to-increase-your-organic-leads-for-life` | LSEO | PUBLISHED |
| 2025-04-02 | How to Build a High-Quality Website in Under a Week Without Learning to Code | `/how-to-build-a-high-quality-website-in-under-a-week-without-learning-to-code` | WD | PUBLISHED |
| 2025-03-31 | How AI is Changing Website Design: Should You Use AI to Build Your Site? | `/how-ai-is-changing-website-design-should-you-use-ai-to-build-your-site` | WD | PUBLISHED |
| 2025-03-28 | What Makes a Website Effective? 7 Key Design Principles You Need | `/what-makes-a-website-effective-7-key-design-principles-you-need` | WD | PUBLISHED |
| 2025-03-26 | GoHighLevel vs. Duda: Which is Better to Start Your Web Design Agency? | `/gohighlevel-vs-duda-which-is-better-to-start-your-web-design-agency` | PC | PUBLISHED |
| 2025-03-21 | The Best All-in-One CRM for Freelancers: HighLevel vs. HoneyBook | `/the-best-all-in-one-crm-for-freelancers-highlevel-vs-honeybook` | PC | PUBLISHED |
| 2025-02-28 | Best Website Builder for Local Businesses: How to Pick One That Actually Gets You Leads | `/blog/best-website-builder-for-local-businesses-how-to-pick-one-that-actually-gets-you-leads` | PC | PUBLISHED |
| 2025-02-27 | Best AI Tools for Website Designers in 2025 | `/blog/best-ai-tools-for-website-designers-in-2025` | WD | PUBLISHED |
| 2025-02-25 | How to Automate Your SEO to Get More Leads | `/blog/how-to-automate-your-seo-to-get-more-leads` | LSEO | PUBLISHED |
| 2025-02-24 | WordPress to Duda Migration: How to Move Your Site Without Losing Rankings (2025 Guide) | `/blog/how-to-migrate-from-wordpress-to-duda-without-losing-seo-rankings` | PC | PUBLISHED |
| 2025-02-20 | Duda vs. Webflow: Which Website Builder is Right for Your Business? | `/blog/duda-vs-webflow-which-website-builder-is-right-for-your-business` | PC | PUBLISHED |
| 2025-02-20 | Why Duda is the Best Website Platform for Agencies | `/blog/why-duda-is-the-best-website-platform-for-agencies` | PC | PUBLISHED |
| 2025-02-18 | The Ultimate Guide to Pediatric Clinic Websites | `/blog/the-ultimate-guide-to-pediatric-clinic-websites` | VG | PUBLISHED |
| 2022-12-21 | 4 Ways to Get More Customer Reviews | `/blog/customer-reviews` | LSEO | PUBLISHED |
| 2022-12-21 | 4 Ways to Market Your Small Business for Free | `/blog/free-marketing` | SB | PUBLISHED |
| 2022-08-17 | Keyword Research for Local Business | `/blog/keyword-research` | LSEO | PUBLISHED |
| 2022-08-09 | Guide to Construction Company SEO 2022 | `/blog/construction-company-seo` | VG | PUBLISHED |
| 2022-07-27 | Local Lead Generation Strategies for Small Business | `/local-lead-generation` | LSEO | PUBLISHED |
| 2022-07-18 | How Artificial Intelligence is Helping Small Businesses Grow | `/blog/artificial-intelligence` | SB | PUBLISHED |
| 2022-07-11 | What is the Google Maps Pack? | `/blog/google-maps-pack` | LSEO | PUBLISHED |
| 2023-03-23 | How Do Lead Generation Companies Work? | `/how-do-lead-generation-companies-work` | SB | UNPUBLISHED |
| — | Demo | `/demo` | — | UNPUBLISHED (ignore) |

**Coverage notes for topic selection:**
- *Construction company SEO* exists but is dated **2022** — a refresh is fair game (mark as update, don't duplicate).
- Platform-comparison cluster is saturated (Duda vs everything). Raise the bar before adding another.
- No published post since **2025-07-19** — cadence has lapsed; re-establishing it is the immediate goal.
- Thin verticals: only pediatric clinics + construction have dedicated guides. Medical (non-pediatric) and other home-service trades are open.
