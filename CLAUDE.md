# Auto Blogger

Multi-client blogging routine, run unattended on an **hourly** schedule. For each **enrolled client**
that is *due* per its strategy, it reads the doctrine, loads the brand story, selects or generates the
topic, **researches and sources it**, QAs against the SEO Doctrine, and creates an unpublished draft on
that client's **Duda** site for teammate review — no external feed reader, scheduler service, or
aggregator.

**The bar is authority, not volume.** Every post earns its place or it does not ship. Missing a slot is
acceptable. Shipping weak content is not.

## Architecture: brain + body

- **Brain (this repo).** The agentic playbook (`routine.md`) + the `seo-doctrine` skill and the
  `spearlanceos-blog-pipeline` skill (write-back schema reference). This is the thinking layer —
  research, judgment, doctrine QA — that the dumb single-call alternatives can't do.
- **Body (SpearlanceOS).** The agency's Supabase app holds client identity, Duda `site_id`, brand
  stories, the **content strategy/calendar**, and enrollment. The routine **reads** it via the Supabase
  MCP and writes to four tables: `blog_topics` (the topic used/created), `blog_posts` (one row per
  drafted article — Mila's draft/schedule/publish queue), `blog_auto_runs` (one row per client
  processed, logging what happened), and `tasks` + `task_assignees` (one review task per article,
  assigned to Mila). It touches nothing else and never invokes SpearlanceOS's `blog-*` edge functions.
- **Publishing (Duda MCP).** Every post is created as an **unpublished draft** on the client's Duda
  site, scoped by `site_id`. The routine never calls `publish_blog_post` — Mila reviews each draft (in
  Duda and in the `blog_posts`/`tasks` queues) and schedules or publishes it herself. This is a
  standing part of the workflow, not a temporary safety valve — see "Go-live posture" below.

> Why not use SpearlanceOS's own `blog-auto-*` functions? They're single LLM API calls (one prompt →
> article), not research + sourcing + doctrine QA. The whole point of this routine is the brain.

## Where client data lives (SpearlanceOS Supabase)

Reach it with the Supabase MCP `execute_sql`, project id **`chikljxwgiskyjsnjelf`** (read-mostly — the
routine writes only `blog_topics`, `blog_posts`, `blog_auto_runs`, `tasks`, and `task_assignees`;
everything else is read-only). The connector must therefore **not** use `read_only=true`; keep it
`project_ref`-scoped. Exact columns/enums/copy-pasteable SQL for the five writable tables live in the
`spearlanceos-blog-pipeline` skill (`.claude/skills/spearlanceos-blog-pipeline/SKILL.md`) — read it
fresh each run rather than re-deriving schema from `information_schema`.

- **`clients`** — `name`, `site_id` (= Duda site/`site_name`), `website_url`, `status`, and the
  enrollment column **`auto_blog_mode`** (`off` | `queue` | `auto_publish`) + **`auto_blog_schedule`**
  (freeform cadence text, unused). `domain` is mostly null — use `website_url` or Duda `get_site_details`.
  `auto_blog_mode` only gates enrollment (`<> 'off'`) now — it no longer selects a publish posture; see
  "Go-live posture" below.
- **`client_brand_voice`** — the fuel. `story_completed` + **`story_summary`** (JSON: marketing_angles,
  value_propositions, pain_points, target_audience_insights, client_voice_samples, tone_indicators,
  key_themes) + `story_transcript`. Where present, this is the client's real, sourced positioning.
- **Content strategy / calendar (drives scheduling + topics):** `blog_content_strategy` (`selected_days`
  = posting calendar, `content_mix`, `posting_frequency`), `blog_topics` (the topic queue: `topic_title`,
  `summary`, `keywords[]`, `category`, `priority`, `suggested_publish_date`, `status`, `blog_post_id`),
  `blog_ai_preferences` (`custom_instructions`, `topics_to_avoid`). The team sets strategy; the routine
  takes the next due topic or generates + saves one into `blog_topics`.
- **`blog_posts`** — one row per drafted article, `status` always inserted as `'draft'`; this plus the
  `tasks` row below is what puts a draft in front of Mila. **`blog_auto_runs`** — one row per client the
  routine processes per tick, `status` ∈ `running`/`completed`/`failed`, with `error_log` free text.
  **`tasks` + `task_assignees`** — one review task per drafted article, assigned to Mila
  (`profiles.email = 'mila@spearlance.com'`; there is no `public.users` table).
- **Unreliable / not ready:** `client_business_model` is mostly empty (ignore). `avatars` is largely
  redundant with the story's audience insights. `brand_guide` **does not exist**, so SpearlanceOS's
  AI-image function can't run — images are deferred (use Duda stock for now).
- **Known DB constraint:** `trg_one_blog_draft_per_client_per_day` (a `BEFORE INSERT` trigger on
  `blog_posts`) caps each client at one draft per calendar day, independent of what the due-check in
  `routine.md` Step 0c says. Reconciling the two is an open item — see "Standing issues" below.

## Enrollment & control surface

- A client is worked when `auto_blog_mode <> 'off'`. The working set is whatever is enrolled — turn
  clients on/off in SpearlanceOS; the routine picks up the change. **You** set the mode; the routine
  only reads it. `auto_blog_mode` (`queue` vs `auto_publish`) no longer selects a publish posture —
  every client's drafts go through Mila's review queue regardless (see "Go-live posture").
- **Scheduling is calendar-driven:** a client is *due* purely when today's weekday ∈ its
  `blog_content_strategy.selected_days` — the due-check does **not** look at whether the client already
  posted today (Mila's queue is allowed to have several drafts waiting; that's normal). In practice a DB
  trigger (`trg_one_blog_draft_per_client_per_day`) still caps one draft/client/day underneath this, so
  most hourly ticks after a due client's first draft of the day are dead by design — see "Standing
  issues."
- **Caveat (as of 2026-09-04):** only 3 of 12 enrolled clients (Invictus Snowfighters, CLE Remodeling,
  Sunshine State) have ever had a `blog_content_strategy` row — the other 9 are enrolled but never due.
  9 clients have a completed brand story (best fuel): Invictus, Trophy Electric, Grounded Columbus,
  Perennial Empire, The Estate at Oak Hollow, Kassthetics, MyHomeVineyard, National Signing Services,
  Game Gen.

## Standing issues (open, need Garrett's decision — routine surfaces these each run, doesn't act on them)

1. **Due-check vs. one-draft-per-day trigger.** `routine.md` Step 0c intentionally ignores same-day
   drafts, but `trg_one_blog_draft_per_client_per_day` enforces exactly one anyway. Fix by loosening the
   trigger, or by making 0c same-day-aware (cheap pre-check before spending research budget).
2. **9/12 enrolled clients have no `blog_content_strategy` row** and are therefore never due — they get
   zero auto-blog content until one is added.
3. **Orphaned Duda drafts from 2026-08-31** (draft created in Duda, matching `blog_posts` insert
   rejected by the one-per-day trigger) sit as unreconciled `to_do` tasks in Mila's queue: Invictus
   (`9b1f35b2`), CLE Remodeling (`55435c1e`), Sunshine State (`abe67c0b`, `4ff2f877`, `02a8dccf`).
4. **One broken Sunshine State Duda draft** — "[DO NOT PUBLISH - BROKEN DRAFT, DELETE] Content
   generation failed - duplicate of 2d22b04e55b84f158a4641d7fa2e0eab" (unpublished, created 2026-08-28)
   — needs manual deletion; the routine won't self-delete without explicit confirmation.
5. **Sunshine State's Jacksonville/Duval/Nassau/St Johns construction+ADU topic space is saturated** —
   repeated independent research passes find no non-duplicate, doctrine-safe angle. Needs a strategic
   call (new city/vertical expansion, a new content pillar, or a lower posting frequency), not continued
   hourly rechecks.

## Autonomous execution (scheduled Claude Routine)

`routine.md` is run unattended on a schedule — no human confirms steps. Design implications baked into it:
- **Preflight gate:** the run aborts cleanly if a required connector is missing. The biggest cloud risk
  is that MCP servers authenticated interactively (Supabase, Duda, DataforSEO, **Tavily** for web
  sourcing) **don't attach in a headless run** — so the schedule **must** have those connectors enabled,
  and the routine fails loudly (publishes nothing) rather than half-running if they're absent.
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
- `.claude/skills/spearlanceos-blog-pipeline/SKILL.md` — the write-back reference (columns, enums,
  copy-pasteable SQL) for `blog_topics`, `blog_posts`, `blog_auto_runs`, `tasks`, `task_assignees`.

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

**The routine never publishes live, for any client.** Every draft it creates in Duda is left
unpublished; `auto_blog_mode` (`queue` vs `auto_publish`) now only gates enrollment, not publish
behavior. Mila reviews every draft (via the Duda editor and the `blog_posts`/`tasks` queues in
SpearlanceOS) and schedules or publishes it herself — this replaced the old per-client `queue`/
`auto_publish` posture and the `HOLD_ALL` kill-switch entirely; there is no live-publish path in the
current routine to switch on.
