# Auto Blogger

Multi-client daily blogging routine, run inside Claude Code. For each **enrolled client**, it reads the
doctrine, loads the client's brand story + inventory, researches keywords, drafts source-backed posts,
QAs them against the SEO Doctrine, and publishes to that client's **Duda** site — no external feed
reader, scheduler service, or aggregator.

**The bar is authority, not volume.** Every post earns its place or it does not ship. Missing a slot is
acceptable. Shipping weak content is not.

## Architecture: brain + body

- **Brain (this repo).** The agentic playbook (`routine.md`) + the `seo-doctrine` skill. This is the
  thinking layer — research, judgment, doctrine QA — that the dumb single-call alternatives can't do.
- **Body (SpearlanceOS).** The agency's Supabase app holds client identity, Duda `site_id`, brand
  stories, and enrollment. The routine **reads** it via the Supabase MCP and is otherwise decoupled:
  it does **not** write to SpearlanceOS or invoke its `blog-*` edge functions.
- **Publishing (Duda MCP).** Posts are created/published on each client's Duda site, scoped by `site_id`.

> Why not use SpearlanceOS's own `blog-auto-*` functions? They're single LLM API calls (one prompt →
> article), not research + sourcing + doctrine QA. The whole point of this routine is the brain.

## Where client data lives (SpearlanceOS Supabase)

Reach it with the Supabase MCP `execute_sql`, project id **`chikljxwgiskyjsnjelf`** (read-only).

- **`clients`** — `name`, `site_id` (= Duda site/`site_name`), `website_url`, `status`, and the
  enrollment columns **`auto_blog_mode`** (`off` | `queue` | `auto_publish`) + **`auto_blog_schedule`**
  (freeform cadence text). `domain` is mostly null — use `website_url` or Duda `get_site_details`.
- **`client_brand_voice`** — the fuel. `story_completed` + **`story_summary`** (JSON: marketing_angles,
  value_propositions, pain_points, target_audience_insights, client_voice_samples, tone_indicators,
  key_themes) + `story_transcript`. Where present, this is the client's real, sourced positioning.
- **Unreliable / not ready:** `client_business_model` is mostly empty (ignore). `avatars` is largely
  redundant with the story's audience insights. `brand_guide` **does not exist**, so SpearlanceOS's
  AI-image function can't run — images are deferred (use Duda stock for now).

## Enrollment & control surface

- A client is worked when `auto_blog_mode <> 'off'`. The working set is whatever is enrolled — turn
  clients on/off in SpearlanceOS; the routine picks up the change. **You** set the mode; the routine
  only reads it.
- `queue` = publish as Duda **draft (hold for review)**; `auto_publish` = **publish live**. This is the
  per-client version of the old single hold flag.
- **As of setup:** 35 clients exist, all `off`; 28 have a `site_id`; **9** have a completed brand story
  (the quality-first pilot tier): Invictus Snowfighters, Trophy Electric, Grounded Columbus, Perennial
  Empire, The Estate at Oak Hollow, Kassthetics Studio, MyHomeVineyard, National Signing Services,
  Game Gen. Spearlance Media is a client (`site_id e8531a5e`) but is not yet story-complete.

## Autonomous execution (scheduled Claude Routine)

`routine.md` is run unattended on a schedule — no human confirms steps. Design implications baked into it:
- **Preflight gate:** the run aborts cleanly if a required connector is missing. The biggest cloud risk
  is that MCP servers authenticated interactively (Supabase, Duda, DataforSEO) **don't attach in a
  headless run** — so the schedule **must** have those three connectors enabled, and the routine fails
  loudly (publishes nothing) rather than half-running if they're absent.
- **Bounded scope:** `POSTS_PER_RUN` (default 2) caps work per run; the most-overdue enrolled client is
  served first. Scale throughput by raising that number and/or the cron frequency.
- **No blocking:** "escalate to Garrett" = flagged in the final report, never a halt. The run's final
  message is the report.

## Project layout

- `routine.md` — the per-client run playbook. **This is what the scheduled routine executes.**
- `feeds.yaml` — optional reactive sources (global SEO/marketing/AI news), verified URLs.
- `.claude/skills/seo-doctrine/` — the governing quality rules, read fresh every run:
  - `SKILL.md` — blog-application layer (which doctrine rules bind a blog post).
  - `doctrine.md` — full vendored doctrine (18 sections). `SEO-Doctrine.pdf` — source of truth.
  - Vendored 2026-06-15 from the separate `SEO Doctrine` workspace so this project is self-contained
    for cloud runs. To update, re-copy both files; the PDF wins on any conflict.

> The markdown ledger from the single-client version is retired: in read-only mode Duda's own post list
> is the system of record (dedup uses `list_blog_posts` per site). Doctrine note: it calls sites
> "custom-coded" — they're Duda sites published through the connector; ignore that framing.

## Non-negotiable rules (full text in `routine.md`)

1. No claim without a real, fetched source (brand-story claims must trace to the client's own story).
2. Never invent citations — cite only URLs actually fetched and read.
3. No fluff, no filler.
4. Every draft is checked against the SEO Doctrine before it is eligible to publish.
5. Publishes only when the bar is cleared.

## Go-live posture

Per-client, via `auto_blog_mode` (`queue` = draft/hold, `auto_publish` = live).

**Global kill-switch:** **HOLD ALL** (current default). While set, the routine forces every client to
draft (`queue`) behavior regardless of its mode, so output is sanity-checked first. Lift it by changing
this line to **LIVE** once quality is proven; individual clients are then governed by their own mode.
