# Auto Blogger

Multi-client blogging routine, run unattended on an **hourly** schedule. For each **enrolled client**
that is *due* per its strategy, it reads the doctrine, loads the brand story, selects or generates the
topic, **researches and sources it**, QAs against the SEO Doctrine, and creates a draft on that client's
**Duda** site — no external feed reader, scheduler service, or aggregator.

**The bar is authority, not volume.** Every post earns its place or it does not ship. Missing a slot is
acceptable. Shipping weak content is not.

**It never publishes live.** Every post lands as an unpublished Duda draft, for every client, always —
a teammate (Mila) reviews and publishes it herself. This is a standing part of the workflow, not a
temporary safety valve; see Go-live posture below.

## Architecture: brain + body

- **Brain (this repo).** The agentic playbook (`routine.md`) + the `seo-doctrine` skill +
  `spearlanceos-blog-pipeline` skill. This is the thinking layer — research, judgment, doctrine QA —
  that the dumb single-call alternatives can't do.
- **Body (SpearlanceOS).** The agency's Supabase app holds client identity, Duda `site_id`, brand
  stories, the **content strategy/calendar**, and enrollment. The routine **reads** it via the Supabase
  MCP and writes to five tables: `blog_topics` (the topic used/created), `blog_posts` (one row per
  drafted article — Mila's review/schedule queue), `blog_auto_runs` (one row per client processed, the
  run log), and `tasks` + `task_assignees` (one review task per article, assigned to Mila). It touches
  nothing else and never invokes SpearlanceOS's `blog-*` edge functions. Exact columns/SQL/status values
  are in `.claude/skills/spearlanceos-blog-pipeline/SKILL.md`.
- **Publishing (Duda MCP).** Draft posts are created on each client's Duda site, scoped by `site_id`.
  Never published through the connector — that's Mila's call.

> Why not use SpearlanceOS's own `blog-auto-*` functions? They're single LLM API calls (one prompt →
> article), not research + sourcing + doctrine QA. The whole point of this routine is the brain.

## Where client data lives (SpearlanceOS Supabase)

Reach it with the Supabase MCP `execute_sql`, project id **`chikljxwgiskyjsnjelf`** (read-mostly — the
routine writes only `blog_topics`, `blog_posts`, `blog_auto_runs`, `tasks`, `task_assignees`; everything
else is read-only). The connector must therefore **not** use `read_only=true`; keep it
`project_ref`-scoped. This is production — see the `supabase-prod-guard` skill; these five tables are
the routine's designed, standing, scheduled write surface, not an ad hoc write, but nothing beyond them.

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
  only reads it. `auto_blog_mode` (`queue` vs `auto_publish`) is enrollment-only now — it no longer
  changes publish behavior (every post is always a draft; see Go-live posture below).
- **Scheduling is calendar-driven:** a client is *due* when today's weekday ∈ its
  `blog_content_strategy.selected_days`. A client with no strategy row is not due — the routine doesn't
  invent a cadence for it.
- **Guard, not a DB constraint:** the routine caps itself to one `blog_posts` draft per client per day
  by checking `blog_posts` before drafting (there is no unique index enforcing this). If you see many
  same-day `blog_auto_runs` rows logging the guard blocking the same due clients, the cron is firing
  more often than the intended hourly cadence — that's an infra issue to fix in the schedule, not a
  doctrine problem.

## Autonomous execution (scheduled Claude Routine)

`routine.md` is run unattended on a schedule — no human confirms steps. Design implications baked into it:
- **Preflight gate:** the run aborts cleanly if a required connector is missing. The biggest cloud risk
  is that MCP servers authenticated interactively (Supabase, Duda, DataforSEO, **Tavily** for web
  sourcing) **don't attach in a headless run** — so the schedule **must** have those connectors enabled,
  and the routine fails loudly (creates no drafts) rather than half-running if they're absent.
  WebFetch/WebSearch are **blocked** in the cloud Routine (403); hosted MCPs (Tavily/DataforSEO) are how
  it reaches the web.
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
- `.claude/skills/spearlanceos-blog-pipeline/` — exact table schemas, status values, and
  copy-pasteable SQL for the `blog_topics`/`blog_posts`/`blog_auto_runs`/`tasks`/`task_assignees`
  write-back. Reconstructed 2026-09-09 from the live schema; keep it in sync with reality, not prose.

> The markdown ledger from the single-client version is retired. **`blog_posts`** (SpearlanceOS) is the
> system of record for a drafted article's review/schedule/publish state; **`blog_auto_runs`** is the
> system of record for what each tick did; Duda's post list is still used for dedup
> (`list_blog_posts` per site) and is the system of record for the post content itself. `blog_topics`
> tracks topic state. Doctrine note: it calls sites "custom-coded" — they're Duda sites published
> through the connector; ignore that framing.

## Non-negotiable rules (full text in `routine.md`)

1. No claim without a real, fetched source (brand-story claims must trace to the client's own story).
2. Never invent citations — cite only URLs actually fetched and read.
3. No fluff, no filler.
4. Every draft is checked against the SEO Doctrine before it is eligible to be created.
5. A draft is created only when the bar is cleared.

## Go-live posture

**Every post is always an unpublished Duda draft, for every client.** `auto_blog_mode` (`queue` vs
`auto_publish`) is enrollment-only — it does not change publish behavior; the routine never calls
`publish_blog_post`. Mila reviews each draft (surfaced via the `blog_posts` row and a `tasks` entry
assigned to her) and schedules or publishes it herself. This superseded the earlier per-client
`queue`/`auto_publish` + inline `HOLD_ALL` kill-switch design described in older commits — there is no
live-publish path left to hold.
