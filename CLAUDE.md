# Auto Blogger

Multi-client blogging routine, run unattended on an **hourly** schedule. For each **enrolled client**
that is *due* per its strategy, it reads the doctrine, loads the brand story, selects or generates the
topic, **researches and sources it**, QAs against the SEO Doctrine, and publishes to that client's
**Duda** site — no external feed reader, scheduler service, or aggregator.

**The bar is authority, not volume.** Every post earns its place or it does not ship. Missing a slot is
acceptable. Shipping weak content is not.

## Architecture: brain + body

- **Brain (this repo).** The agentic playbook (`routine.md`) + the `seo-doctrine` skill. This is the
  thinking layer — research, judgment, doctrine QA — that the dumb single-call alternatives can't do.
- **Body (SpearlanceOS).** The agency's Supabase app holds client identity, Duda `site_id`, brand
  stories, the **content strategy/calendar**, and enrollment. The routine **reads** it via the Supabase
  MCP and writes back to **one table only** — `blog_topics` (the topic it used/created, marked done). It
  touches nothing else and never invokes SpearlanceOS's `blog-*` edge functions.
- **Publishing (Duda MCP).** Posts are created/published on each client's Duda site, scoped by `site_id`.

> Why not use SpearlanceOS's own `blog-auto-*` functions? They're single LLM API calls (one prompt →
> article), not research + sourcing + doctrine QA. The whole point of this routine is the brain.

## Where client data lives (SpearlanceOS Supabase)

Reach it with the Supabase MCP `execute_sql`, project id **`chikljxwgiskyjsnjelf`** (read-mostly — the
routine writes **only** `blog_topics`; everything else is read-only). The connector must therefore **not**
use `read_only=true`; keep it `project_ref`-scoped.

- **`clients`** — `name`, `site_id` (= Duda site/`site_name`), `website_url`, `status`, and the
  enrollment columns **`auto_blog_mode`** (`off` | `queue` | `auto_publish`) + **`auto_blog_schedule`**
  (freeform cadence text). `domain` is mostly null — use `website_url` or Duda `get_site_details`.
- **`client_brand_voice`** — the fuel. `story_completed` + **`story_summary`** (JSON: marketing_angles,
  value_propositions, pain_points, target_audience_insights, client_voice_samples, tone_indicators,
  key_themes) + `story_transcript`. Where present, this is the client's real, sourced positioning.
- **Content strategy / calendar (drives scheduling + topics):** `blog_content_strategy` (`selected_days`
  = posting calendar, `content_mix`, `posting_frequency`), `blog_topics` (the topic queue: `topic_title`,
  `summary`, `keywords[]`, `category`, `priority`, `suggested_publish_date`, `status`, `blog_post_id`),
  `blog_ai_preferences` (`custom_instructions`, `topics_to_avoid`). The team sets strategy; the routine
  takes the next due topic or generates + saves one into `blog_topics`.
- **Unreliable / not ready:** `client_business_model` is mostly empty (ignore). `avatars` is largely
  redundant with the story's audience insights. `brand_guide` **does not exist**, so SpearlanceOS's
  AI-image function can't run — images are deferred (use Duda stock for now).

## Enrollment & control surface

- A client is worked when `auto_blog_mode <> 'off'`. The working set is whatever is enrolled — turn
  clients on/off in SpearlanceOS; the routine picks up the change. **You** set the mode; the routine
  only reads it.
- `queue` = publish as Duda **draft (hold for review)**; `auto_publish` = **publish live**. This is the
  per-client version of the old single hold flag.
- **Scheduling is calendar-driven:** a client is *due* when today's weekday ∈ its
  `blog_content_strategy.selected_days` and it hasn't already posted today.
- **Caveat (as of 2026-06-16):** most enrolled clients have **no** `blog_topics` yet, and the few that do
  (Invictus, Sunshine State) are stale/off-season — so the routine mostly **generates** topics until
  calendars are populated/freshened. 9 clients have a completed brand story (best fuel): Invictus, Trophy
  Electric, Grounded Columbus, Perennial Empire, The Estate at Oak Hollow, Kassthetics, MyHomeVineyard,
  National Signing Services, Game Gen.

## Autonomous execution (scheduled Claude Routine)

`routine.md` is run unattended on a schedule — no human confirms steps. Design implications baked into it:
- **Preflight gate:** the run aborts cleanly if a required connector is missing. The biggest cloud risk
  is that MCP servers authenticated interactively (Supabase, Duda, DataforSEO) **don't attach in a
  headless run** — so the schedule **must** have those three connectors enabled, and the routine fails
  loudly (publishes nothing) rather than half-running if they're absent.
- **Hourly cron, calendar-driven:** most ticks do nothing — the routine runs the cheap due-check first
  and exits before reading the doctrine if no client is due. `POSTS_PER_RUN` (default 1) caps work per
  tick; the highest-priority / longest-waiting due client is served. Raise only to clear a backlog.
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

> The markdown ledger from the single-client version is retired: Duda's own post list is the system of
> record for published content (dedup uses `list_blog_posts` per site), and `blog_topics` tracks topic
> state. Doctrine note: it calls sites "custom-coded" — they're Duda sites published through the
> connector; ignore that framing.

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
