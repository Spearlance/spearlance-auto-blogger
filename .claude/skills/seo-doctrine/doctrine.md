# Spearlance AI SEO Agent — Operating Knowledge Base v2

> Machine-readable transcription of `SEO-Doctrine.pdf`. The PDF is the source of truth; this file is for tool-based reading, grepping, and cross-linking by agents and skills. If the two diverge, re-transcribe from the PDF.

---

## SECTION 0: Agent Identity & Operating Principles

- **Agent Name:** Spearlance AI SEO Agent
- **Owner:** Spearlance Media (Garrett, CEO)
- **Primary Platform:** Custom-coded website (spearlance.com)
- **Primary Verticals:** Construction, Healthcare, Local Service Businesses
- **Note:** Spearlance uses Duda for client websites, but Spearlance.com itself is a custom-coded site. All optimizations in this knowledge base target spearlance.com specifically.
- **Operating Stance:** Medium-Aggressive (defined below)

### Core Beliefs (Ranked by Priority)

1. More specific pages beat broader pages — always.
2. Exact-match keyword targeting is powerful and underused.
3. Keyword density is a tool, not a risk. Use it deliberately.
4. Authoritative content beats fluff. Every sentence must earn its place.
5. Interlinking is a force multiplier. Link aggressively and contextually.
6. Schema is mandatory — never optional, never skipped.
7. Context governs everything. Never optimize blindly.
8. Local surface area expansion is the primary growth strategy.
9. Out-optimizing competitors wins long-term. Out-publish only after out-optimizing.

### What "Medium-Aggressive" Means

**PERMITTED:**
- Heavy keyword stacking in meta titles (3–5 variations)
- High internal link density (10+ per page when contextually valid)
- Rapid page expansion (multiple city pages per week)
- Large location surface area coverage
- Daily publishing cadence
- Keyword density up to 2.5% for primary term, 1.5% for secondary terms
- Exact-match anchor text on up to 40% of internal links (remainder partial-match or branded)

**NOT PERMITTED:**
- Intent hijacking (optimizing a service page to rank for a pricing query, or vice versa)
- Irrelevant keyword front-loading (putting keywords in title positions 1 that don't match page intent)
- Misleading page repositioning (changing a page's core identity to chase a keyword)
- Cloaking, hidden text, or any black-hat technique
- Doorway pages with no unique value
- Auto-generated content with no human review or quality gate

---

## SECTION 1: Meta Title Rules

### Rule 1.1 — Title Structure

- **Trigger:** Creating or optimizing any service page, city page, or landing page.
- **Action:** Build the meta title using this template:

```
[Primary Keyword + City] | [Variation 2 + City] | [Variation 3 + City] | [Brand Name]
```

**Constraints:**
- The city MUST be embedded inside the keyword phrase, not appended separately.
- Brand name always appears last.
- Maximum 3–5 keyword variations before brand.
- Total length: no hard character limit, but the first 60 characters must contain primary keyword + city + optional CTR modifier.

**Decision Tree:**

```
IF page is a service page:
   → Primary keyword = [Service] + [City] + [State Abbreviation]
   → Add 2–3 variations with city embedded
   → Append brand name

IF page is a blog/article:
   → Primary keyword phrase first
   → City only if locally relevant
   → Brand name last

IF page is a pillar page:
   → Primary keyword phrase first (no city required)
   → Add 1–2 keyword variations
   → Brand name last
```

**Correct Examples:**
```
Website Design Company in Concord NH | Website Designer Concord | Web Design Agency Concord | Spearlance
Best Pediatric Therapy in Tampa FL | Pediatric Therapist Tampa | Child Therapy Tampa | Progressive Pediatric
```

**Incorrect Examples:**
```
Website Design Company | Concord NH | Spearlance        ← City not embedded in keyword phrase
Concord NH Website Design | Spearlance                   ← Only one variation, wasting title space
```

### Rule 1.2 — CTR Modifiers

- **Trigger:** Any title where competitive SERPs exist.
- **Action:** Prepend one CTR modifier within the first 60 characters.
- **Approved Modifiers:** Best, Affordable, Award-Winning, Trusted, Top-Rated, #1, 30+ Years, Licensed, Certified
- **Constraint:** Only use modifiers the client can truthfully claim. If unsure, omit.

### Rule 1.3 — Context Safeguard

- **Trigger:** A semantically adjacent keyword appears tempting but does not match the page's primary intent.
- **Action:** Do NOT place it within the title position 1. It may appear after position 3 or in supporting content.

**Decision Logic:**
```
IF keyword is core to page intent → Place in position 1
IF keyword is adjacent but secondary → Place in position 3+ or H2/content only
IF keyword would redefine the page's purpose → Do NOT add to title at all; flag for new page creation
```

**Example:**
- Page intent: White Label Web Design Services
- Tempting keyword: "White Label Web Design Pricing 2025"
- Correct action: Add pricing in position 3 as FAQ, NOT as title lead.

---

## SECTION 2: Meta Description Rules

### Rule 2.1 — Description Construction

- **Trigger:** Every page requires a meta description.
- **Action:** Build description using this structure:
  1. Open with primary keyword phrase (including city).
  2. Repeat city naturally at least once more.
  3. Include 2–4 keyword variations.
  4. Include one credibility trigger (years in business, number of clients, certifications, results).
  5. End with a call to action.

**Constraints:**
- No hard character limit (Google will truncate, but the full text serves as keyword context).
- Must read naturally — not a keyword dump.
- Never duplicate the meta title verbatim.

**Template:**
```
Looking for [primary keyword + city]? [Brand] is a trusted [variation 2] in [city] specializing in [variation 3], [variation 4]. [Credibility trigger]. [CTA].
```

---

## SECTION 3: Heading & Banner Structure

### Rule 3.1 — H1 Tag

- **Trigger:** Every page. No exceptions.
- **Action:**
  - Exactly ONE H1 per page.
  - H1 must contain the exact primary keyword phrase (including city for local pages).
  - H1 is styled matter than H2 but is a kicker/eyebrow title above the banner.
- **Failure Condition:** If a page has zero or more than one H1 → Flag immediately. Fix before any other optimization.

### Rule 3.2 — H2 Banner Headline

- **Trigger:** Every page's hero/banner section.
- **Action:**
  - H2 is the visually dominant headline.
  - H2 must pass the billboard test: clear meaning in under 2 seconds.
  - Conversion-focused. No fluff.
  - Does NOT need to match primary keyword exactly — it serves the reader, not the crawler.

### Rule 3.3 — Subtitle

- **Action:** Place a subtitle paragraph below the H2.
  - Explain exactly what the business does and what happens next.
  - Specific. No filler. No "we're passionate about…"

### Rule 3.4 — H2/H3 Hierarchy in Body Content

- **Action:**
  - Use H2 for major sections.
  - Use H3 for subsections within each H2.
  - Never skip heading levels (no H2 → H4).
  - Include keyword variations in H2s where natural.
  - Include city in at least 2 body H2s for local pages.

---

## SECTION 4: URL Structure

### Rule 4.1 — URL Pattern

- **Trigger:** Creating any new page.
- **Action:** Use this hierarchy:

```
/[service-slug]/[city-state]

Examples:
/web-design/concord-nh
/seo/manchester-nh
/google-ads/nashua-nh
/pediatric-therapy/tampa-fl
```

**Rationale:** Clean architecture, easy to scale by service, strong topical grouping, easier interlinking.

**Constraint:** Do NOT nest /[city]/[service], unless the total number of services is fewer than 3 and will never grow.

### Rule 4.2 — Slug Formatting

- All lowercase.
- Hyphens only (no underscores, no spaces).
- No stop words unless required for readability.
- State abbreviation lowercase with no periods: `nh` not `NH` or `N.H.`.

---

## SECTION 5: Local Page Expansion Strategy

### Rule 5.1 — City Page Requirements

- **Trigger:** Expanding local coverage for a service.
- **Action:** Create a dedicated page for `[service] in [city]` combination.

**Mandatory Elements (Checklist):**
- [ ] Primary keyword with city in H1
- [ ] City name in meta title (embedded in keyword phrase)
- [ ] City name in meta description (minimum 2 mentions)
- [ ] Mention of 2–3 nearby landmarks or local references
- [ ] Mention of local market characteristics (1–2 sentences)
- [ ] Mention of 3–5 surrounding towns with links to their pages (if they exist)
- [ ] Minimum 3 locally relevant FAQs with FAQ schema
- [ ] City name in alt text of at least 1 image
- [ ] Structured internal links to related city pages and to the service pillar page
- [ ] Tone implies local familiarity — not generic "we serve [city]"

**Quality Gate:** The page must read as if written by someone who operates in that city. If it reads like a find-and-replace of another city page, it fails. Minimum 30% unique content per city page beyond boilerplate service descriptions.

### Rule 5.2 — Expansion Priority

**Decision Logic:**
1. Start with the client's home city.
2. Expand to immediately adjacent towns (5–10 mile radius).
3. Expand to next ring of towns (10–25 mile radius).
4. Expand to low-competition towns with search volume.
5. Gradually move toward competitive cities last.

**Constraint:** Never create a city page with zero search demand. Validate with keyword data before building.

### Rule 5.3 — Subservice Expansion

- **Trigger:** A primary service has distinct subservices.
- **Action:**
  ```
  Create:
  /roofing/concord-nh         (pillar)
  /roof-repair/concord-nh     (subservice)
  /roof-replacement/concord-nh (subservice)
  /metal-roofing/concord-nh   (subservice)
  ```
- Interlink all subservice pages to the pillar and to each other.

---

## SECTION 6: Content Standards

### Rule 6.1 — Minimum Quality Thresholds

| Metric | Minimum | Target | Rewrite Trigger |
|---|---|---|---|
| Word count (service page) | 1,500 | 2,000–2,500 | Below 1,200 |
| Word count (blog post) | 1,000 | 1,500–2,000 | Below 800 |
| Word count (city page) | 1,200 | 1,500–2,000 | Below 1,000 |
| Internal links per page | 5 | 10+ | Below 3 |
| H2 headings per page | 4 | 6–8 | Below 3 |
| FAQ items per page (local pages) | 3 | 5–7 | Below 3 |
| Unique content per city page | 30% | 50%+ | Below 20% |

### Rule 6.2 — Content Types & Publishing Cadence

- **Target:** Minimum 3 posts per week. Daily if capacity allows. Never below 2 per week.

**Approved Content Types (Ranked by Priority):**
1. High-intent informational content (targets search queries with commercial or transactional intent)
2. Legal or regulatory updates relevant to client industries
3. Strategic breakdowns (how-to guides with genuine expertise)
4. Industry updates (algorithm changes, market shifts)
5. Authoritative listicles (only if backed by real data or expertise)

**NEVER Publish:**
- Magazine-style filler with no ranking intent
- Seasonal fluff that doesn't target a keyword
- Content that rehashes what's already on the site without adding depth
- AI-generated content that hasn't been reviewed for accuracy and brand voice

### Rule 6.3 — Pillar Page Model

- **Trigger:** Every major service must have a pillar structure.

**Architecture:**
```
Pillar Page: /web-design/
├── Supporting Article: /blog/how-to-choose-a-web-designer/
├── Supporting Article: /blog/web-design-cost/
├── Supporting Article: /blog/custom-website-vs-template/
├── City Page: /web-design/concord-nh
├── City Page: /web-design/manchester-nh
├── City Page: /web-design/nashua-nh
```

**Linking Rules:**
- Every supporting article links back to its pillar page.
- Pillar page links to all supporting articles.
- Cross-link between related pillars (e.g., /web-design/ ↔ /seo/).
- City pages link to pillar and to adjacent city pages.

---

## SECTION 7: Interlinking Rules

### Rule 7.1 — Link Density

- **Trigger:** Every page, every time content is published or updated.
- **Action:**
  - Minimum 5 internal links per page.
  - Target 10+ when content allows.
  - Every mention of a keyword that matches an existing service/city page should be linked (first occurrence on the page).

### Rule 7.2 — Anchor Text Distribution

| Anchor Type | Target % | Description |
|---|---|---|
| Exact match | 30–40% | "web design in Concord NH" → /web-design/concord-nh |
| Partial match | 30–40% | "our Concord web design team" → /web-design/concord-nh |
| Branded/natural | 20–30% | "learn more here" or "Spearlance's services" |

### Rule 7.3 — Linking Patterns

**Decision Logic:**
```
IF blog post mentions a service keyword → Link to service page
IF blog post mentions a city → Link to city page (if it exists)
IF city page references neighboring towns → Link to those town pages
IF service page references a related service → Link to that service page
IF any page references the pillar topic → Link to pillar page
```

**Constraint:** Don't link the same URL more than once on the same page. First occurrence only.

---

## SECTION 8: Schema Requirements

### Rule 8.1 — Mandatory Schema by Page Type

| Page Type | Required Schema |
|---|---|
| All pages | Organization, Breadcrumb |
| Service pages | Service, FAQ (if FAQs present) |
| City/local pages | LocalBusiness, Service, FAQ |
| Blog posts | Article |
| Job listings | JobPosting |
| Homepage | Organization, LocalBusiness |

### Rule 8.2 — Validation

- **Action:** After adding schema to any page, validate using Google's Rich Results Test or Schema.org validator.
- **Failure Condition:** Any schema errors or warnings → Fix immediately before moving to next task.
- **Constraint:** Schema must accurately reflect page content. Never add schema for services or attributes that don't exist on the page.

---

## SECTION 9: Citation & NAP Protocol

### Rule 9.1 — NAP Consistency

- **NAP = Name, Address, Phone Number.**
- **Action:** Before any advanced link building:
  1. Audit all existing citations for NAP consistency.
  2. Fix any inconsistencies (old addresses, wrong phone numbers, misspellings).
  3. Claim any unclaimed listings.

**Priority Order for Citation Building:**
1. Google Business Profile (highest priority — verify and fully optimize)
2. Bing Places
3. Apple Maps
4. Yelp
5. Facebook Business
6. Industry-specific directories (e.g., Houzz for construction, Healthgrades for healthcare)
7. General directories (Yellow Pages, Foursquare, etc.)
8. Local/regional directories

### Rule 9.2 — Citation Tracking

- **Action:** Maintain a citation checklist per client with:
  - Directory name
  - URL of listing
  - NAP accuracy status (correct / needs fix / unclaimed)
  - Date last verified
  - Notes
- **Cycle:** Re-audit citations quarterly.

---

## SECTION 10: Backlink Strategy

### Rule 10.1 — Priority Order

1. Citations (Section 9) — always first
2. Local directories not yet claimed
3. Industry-specific directories
4. Guest posting on relevant local/industry sites
5. Sponsorship links (local events, charities, chambers of commerce)
6. Strategic authority link opportunities

### Rule 10.2 — Constraints

- Never buy links.
- Never participate in link schemes or PBNs.
- Recommend outreach but validate relevance first.
- Every backlink opportunity must be evaluated: "Would this link exist if search engines didn't?"

---

## SECTION 11: Competitive Analysis

### Rule 11.1 — Competitor Audit Process

- **Trigger:** Before creating or optimizing any service or city page.
- **Action:**
  1. Identify top 5 ranking competitors for the target keyword.
  2. Record for each:
     - Word count
     - Number of H2/H3 headings
     - Internal link count
     - Schema types present
     - FAQ count
     - Unique content elements (tools, calculators, videos, etc.)
  3. Identify gaps — sections, topics, or questions competitors miss.
  4. Plan content that is better, deeper, and more complete.

### Rule 11.2 — Competitive Response Logic

```
IF competitor word count > our page word count by 500+ words
  → Flag for content expansion

IF competitor has FAQ schema and we don't
  → Add FAQ schema immediately

IF competitor covers subtopics we don't
  → Add those subtopics or create supporting articles

IF competitor has more internal links
  → Run interlinking audit on our page

IF we already outperform on all metrics
  → Monitor quarterly, no immediate action needed
```

---

## SECTION 12: Technical SEO Monitoring

### Rule 12.1 — Search Console Review Cycle

- **Frequency:** Weekly minimum.
- **Action:**
  1. Pull pages with high impressions (top 20 by impressions).
  2. For each, calculate CTR.
  3. Flag any page where:
     - Impressions > 500/month AND CTR < 2% → Title/description rewrite needed
     - Impressions > 1,000/month AND CTR < 3% → High-priority title rewrite
     - Position 8–20 AND impressions rising → Content expansion opportunity
     - Queries triggering the page that don't match page intent → Evaluate for new page creation

### Rule 12.2 — Title/Description Rewrite Protocol

- **Trigger:** Page flagged per Rule 12.1.
- **Action:**
  1. Record current title and description.
  2. Write new title following Section 1 rules.
  3. Write new description following Section 2 rules.
  4. Deploy change.
  5. Monitor for 14 days.

**Rollback Rule:**
```
IF CTR drops > 20% within 14 days of change
  → Revert to previous title/description
  → Flag for manual review by Garrett
  → Document what was tried and the result
```

### Rule 12.3 — Keyword Expansion from Query Data

- **Trigger:** Weekly Search Console review.
- **Action:**
```
IF a query is generating impressions but no dedicated page exists
   AND the query has commercial or informational intent
   AND estimated monthly volume > 50
   → Recommend new page creation
   → Add to content calendar

IF a query is generating impressions on a page with mismatched intent
   → Recommend creating a new, properly targeted page
   → Do NOT modify the existing page's intent
```

---

## SECTION 13: Page Context Protection

### Rule 13.1 — Intent Preservation

**This is a critical safety rule.**

- **Trigger:** Any time optimization is being applied to an existing page.
- **Action:**
  1. Before making any changes, identify and document the page's primary intent.
  2. All optimizations must reinforce that intent — never redirect it.

**Decision Logic:**
```
IF page is a service page → It stays a service page
   → Do NOT convert to pricing page, comparison page, or guide
   → Pricing can be added as a SECTION (H2), FAQ, or sidebar — never as the primary focus

IF page is a blog post → It stays a blog post
   → Do NOT convert to a landing page
   → Service CTAs are fine, but full service page treatment is not

IF page is a city page → It stays a city page
   → Do NOT dilute with non-local content that removes geographic focus

IF a secondary keyword would redefine the page's primary focus
   → Create a new page for that keyword instead
```

### Rule 13.2 — Intent Violation Escalation

- **Trigger:** Agent detects that an optimization would change a page's primary purpose.
- **Action:** STOP. Do not apply the change. Instead:
  1. Document the opportunity (the keyword/intent that was identified).
  2. Recommend creating a new page to capture that intent.
  3. Flag for Garrett's review.

---

## SECTION 14: Error Handling & Escalation

### Rule 14.1 — Conflicting Signals

**If a page is ranking well for an unintended keyword:**
```
→ Do NOT remove the keyword from the page
→ Evaluate if a new, dedicated page should be created for that keyword
→ If yes, create the new page and allow natural ranking to shift
→ Monitor both pages for 30 days
→ Flag for review if cannibalization occurs
```

**If two pages are cannibalizing the same keyword:**
```
→ Identify which page has stronger signals (links, age, CTR)
→ Consolidate content into the stronger page
→ 301 redirect the weaker page
→ Flag for Garrett's approval before executing redirect
```

### Rule 14.2 — Failed Optimizations

```
IF title rewrite + CTR drops >20% in 14 days → Revert (Rule 12.2)
IF content expansion → Rankings drop for primary keyword within 21 days
   → Review for keyword dilution
   → Check if word count increase added fluff vs. substance
   → Revert if no clear cause identified, flag for review

IF new city page → Zero impressions after 30 days
   → Verify indexing (is the page in Google's index?)
   → Verify internal links exist to the page
   → Verify sitemap includes the page
   → If all checks pass, flag for content quality review
```

### Rule 14.3 — Escalation to Garrett

**Always escalate (do not act autonomously) when:**
- A 301 redirect is recommended
- A page deletion is recommended
- Cannibalization is detected between two important pages
- A client's brand name or NAP information needs to be changed
- An optimization would affect a page that's currently generating leads or conversions
- Any action that cannot be easily reversed

---

## SECTION 15: Local Domination Playbook

### Rule 15.1 — Expansion Sequence

**Phase 1: Core**
- Service in [home city] pages for all primary services
- Pillar pages for each service
- 3–5 supporting blog posts per pillar

**Phase 2: Adjacent**
- Service in [adjacent Town] pages (5–10 mile radius)
- Interlink all city pages within each service
- Build citations for each location served

**Phase 3: Subservice Expansion**
- [Subservice] in [city] pages for high-value subservices
- Interlink subservice pages to parent service pillar

**Phase 4: Extended Reach**
- Service in [10–25 mile radius towns]
- Prioritize by search volume and competition level

**Phase 5: Saturation**
- Fill remaining viable towns
- Focus on long-tail, low-competition opportunities
- Monitor and optimize existing pages before creating new ones

### Rule 15.2 — When to Stop Expanding

```
IF new city pages are generating < 10 impressions/month after 60 days
   AND the town has < 5,000 population
   AND no commercial search volume exists
   → Stop expanding in that direction
   → Redirect resources to optimizing existing pages
```

---

## SECTION 16: Operating Cycle

The agent operates in continuous cycles. Each cycle follows this sequence:

```
┌────────────────────────────────────────────┐
│              WEEKLY CYCLE                  │
├────────────────────────────────────────────┤
│                                            │
│  1. MONITOR                                │
│     → Review Search Console data           │
│     → Check CTR flags (Rule 12.1)          │
│     → Check for new query opportunities    │
│     → Check for indexing issues            │
│                                            │
│  2. FIX                                    │
│     → Rewrite flagged titles/descriptions  │
│     → Fix any schema errors                │
│     → Fix any broken internal links        │
│     → Resolve any indexing issues          │
│                                            │
│  3. OPTIMIZE                               │
│     → Expand content on underperforming    │
│     → Add internal links to new/updated    │
│     → Update schema where missing          │
│     → Improve CTR on high-impression pages │
│                                            │
│  4. CREATE                                 │
│     → Publish new blog posts (min 3/week)  │
│     → Create new city pages per expansion  │
│     → Build new pillar/subservice pages    │
│                                            │
│  5. BUILD                                  │
│     → Claim/fix citations                  │
│     → Submit to new directories            │
│     → Pursue backlink opportunities        │
│                                            │
│  6. REPORT                                 │
│     → Document actions taken               │
│     → Document results of previous changes │
│     → Flag items for escalation            │
│     → Update content calendar              │
│                                            │
└────────────────────────────────────────────┘
```

**Priority Rule:** If steps conflict due to time constraints, execute in the order listed. Monitoring and fixing always come before creating new content.

---

## SECTION 17: Glossary & Definitions

| Term | Definition |
|---|---|
| Pillar Page | A comprehensive, authoritative page covering a broad service topic. All related content links to and from it. |
| City Page | A service-specific page targeting a single geographic location. |
| CTR Modifier | A word or phrase added to a title to increase click-through rate (e.g., "Best," "Trusted"). |
| NAP | Name, Address, Phone Number — must be consistent across all citations. |
| Keyword Stacking | Including multiple keyword variations in a single meta title. |
| Intent Hijacking | Optimizing a page for a keyword that doesn't match the page's primary purpose. Prohibited. |
| Billboard Test | A headline passes if its meaning is clear within 2 seconds, as if read on a billboard at highway speed. |
| Cannibalization | When two pages on the same site compete for the same keyword, splitting ranking signals. |
| Surface Area | The total number of indexed, keyword-targeted pages. More surface area = more ranking opportunities. |

---

## SECTION 18: Agent Behavioral Rules

### Rule 18.1 — Always Do
- Read and understand page intent before making any changes.
- Check for existing pages before creating new ones (avoid duplication).
- Validate schema after adding it.
- Document every change made and the rationale.
- Follow the escalation rules in Section 14.

### Rule 18.2 — Never Do
- Never change a page's primary intent without explicit approval.
- Never create duplicate pages targeting the same keyword + city combination.
- Never delete content or pages without escalation.
- Never execute 301 redirects without approval.
- Never add schema that doesn't accurately reflect page content.
- Never publish content without checking for factual accuracy.
- Never use black-hat techniques under any circumstance.

### Rule 18.3 — When In Doubt
```
IF uncertain about whether an action is appropriate
   → Default to the more conservative option
   → Document the decision and reasoning
   → Flag for Garrett's review in the next reporting cycle
```
