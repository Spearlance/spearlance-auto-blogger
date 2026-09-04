---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back to SpearlanceOS (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read this during the Publish and Record-the-Run steps of routine.md instead of re-deriving the schema from information_schema every tick.
---

# SpearlanceOS blog pipeline: write-back reference

Project id: `chikljxwgiskyjsnjelf` (production — see `supabase-prod-guard` skill before any write).
Schema introspected live via `information_schema` on 2026-09-04; re-verify if a write ever fails on an
unrecognized column, since this file is not auto-synced with the live DB.

The routine reads many tables (see CLAUDE.md) but writes only to the five below. Nothing else, ever.

## `blog_topics`

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, `gen_random_uuid()` |
| `client_id` | uuid | FK → `clients.id` |
| `topic_title` | text | |
| `description` | text | long-form description (routine.md's `summary` maps to the `summary` column, not this one — both exist) |
| `summary` | text | short summary used in the Step 4 SELECT/INSERT in routine.md |
| `keywords` | text[] | |
| `avatar_id` | uuid | optional, rarely populated |
| `priority` | text | free text, e.g. `low`/`medium`/`high` — no enum constraint |
| `status` | text | free text; observed values: `idea`, `in_progress`, `published` |
| `blog_post_id` | uuid | FK → `blog_posts.id`, set once a draft exists for this topic |
| `ai_generated` | boolean | |
| `strategy_batch_id` | uuid | optional |
| `suggested_publish_date` | date | |
| `category` | text | free text, should line up with `blog_content_strategy.content_mix` keys |
| `created_at` | timestamptz | |

**Take a calendar topic (Step 4a):**
```sql
SELECT id, topic_title, summary, keywords, category, priority, suggested_publish_date
FROM public.blog_topics
WHERE client_id = '<client_id>' AND blog_post_id IS NULL
  AND status NOT IN ('published', 'in_progress')
  AND suggested_publish_date <= current_date
ORDER BY priority DESC, suggested_publish_date ASC;
```
Mark it in progress before drafting:
```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**Generate a new topic (Step 4b):**
```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>', true, 'in_progress', current_date)
RETURNING id;
```

**After a draft is created (Publish step):**
```sql
UPDATE public.blog_topics
SET status = 'published', blog_post_id = '<blog_post_id>'
WHERE id = '<topic_id>';
```
("published" here means "turned into a post," matching this table's existing convention — not that the
Duda post itself is live.)

## `blog_posts`

The team's draft/schedule/publish queue. One row per drafted article. **Always insert `status = 'draft'`
— never `'scheduled'` or `'published'`; that's Mila's call.**

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | FK → `clients.id` |
| `title` | text | **NOT NULL** |
| `slug` | text | **NOT NULL** — use the Duda post's `path` |
| `meta_description` | text | |
| `content` | text | full HTML body, same as what was sent to Duda |
| `excerpt` | text | |
| `focus_keyword` | text | |
| `seo_score` / `readability_score` | numeric | optional, leave null unless you compute one |
| `avatar_id` | uuid | optional |
| `topic_category` | text | |
| `target_keywords` | text[] | |
| `featured_image_url` / `featured_image_alt` | text | mirror what was sent to Duda as thumbnail/main image |
| `body_images` | jsonb | optional |
| `status` | text | default `'draft'` — **always leave as `'draft'`** |
| `scheduled_for`, `published_at` | timestamptz | leave null — Mila sets these |
| `duda_page_id` | text | the Duda post id returned by `create_blog_post` |
| `duda_publish_url` | text | the preview/editor URL from Duda |
| `ai_model` | text | defaults to `'claude-sonnet-4-5'` — leave default unless you know otherwise |
| `generation_prompt`, `generation_metadata` | text / jsonb | optional provenance |
| `created_by` | uuid | optional |
| `topic_id` | uuid | FK → `blog_topics.id` — the topic this draft came from |
| `scheduled_publish_date` | date | optional, informational only |
| `images_generated` | boolean | default false (images are deferred per CLAUDE.md) |
| `auto_run_id` | uuid | FK → `blog_auto_runs.id` opened for this client this tick |
| `quality_scores` | jsonb | the doctrine QA verdict — put the rule-by-rule pass/fail summary here |
| `rejection_reason` | text | only set if you're recording a non-published QA outcome via this table (rare — usually a fail means no row is inserted at all) |
| `revision_count` | integer | default 0 |

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt, focus_keyword,
  topic_category, target_keywords, featured_image_url, featured_image_alt,
  status, duda_page_id, duda_publish_url, topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<duda_path>', '<meta_description>', '<html_content>', '<excerpt>',
  '<focus_keyword>', '<category>', ARRAY['<kw1>','<kw2>'], '<image_url>', '<image_alt>',
  'draft', '<duda_post_id>', '<duda_preview_url>', '<topic_id>', '<auto_run_id>',
  '{"word_count": 1650, "h2_count": 5, "internal_links": 7, "verdict": "SHIP", "sourced_stats": 4}'::jsonb
)
RETURNING id;
```

**Important operational note (confirmed live, 2026-09-04):** a DB trigger
`trg_one_blog_draft_per_client_per_day` (`BEFORE INSERT ON public.blog_posts`) rejects a second insert
for the same `client_id` on the same UTC calendar day. routine.md's Step 0c says same-day drafts should
not affect due-ness — that guidance and this trigger disagree. Check same-day drafts before attempting a
full research→draft cycle to avoid burning Tavily/DataForSEO spend on work the insert will then reject:
```sql
SELECT id FROM public.blog_posts WHERE client_id = '<client_id>' AND created_at::date = current_date;
```
If a row already exists, treat the client as not eligible this tick regardless of what Step 0c implies,
and move to the next due client. Flag this contradiction in the report; it needs a decision from Garrett
(either drop the trigger, or change routine.md/Step 0c to check same-day drafts, since the two currently
fight each other and burn most of the day's hourly ticks after the first draft lands).

## `blog_auto_runs`

One row per client actually processed each tick (opened before research/drafting, closed once, at the
end of that client's branch).

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | **NOT NULL**, no enum — observed value in production: `'scheduled'`. Use `'manual'` only for an ad hoc single-`client_id` invocation per Step 0b. |
| `status` | text | **NOT NULL**, default `'running'` — observed terminal values: `'completed'`, `'failed'` |
| `topics_generated` | integer | default 0 |
| `articles_generated` | integer | default 0 |
| `articles_passed_gate` | integer | default 0 |
| `articles_flagged` | integer | default 0 |
| `research_summary` | jsonb | optional — sources found, keyword data, etc. |
| `completed_at` | timestamptz | set when you close the row |
| `error_log` | text | **the report field** — free text; put the full per-tick narrative here (what was checked, what was found, why a draft was or wasn't created) since this table has no other narrative column |
| `created_at` | timestamptz | default `now()` |

**Open (per routine.md, before research):**
```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```
**Close — success:**
```sql
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = <n>, articles_generated = 1, articles_passed_gate = 1,
    error_log = '<full narrative of what was researched, sourced, drafted, and why it passed QA>'
WHERE id = '<run_id>';
```
**Close — stopped branch (no draft):**
```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(),
    error_log = '<why: no fuel / no strategy row / QA fail / topic-space exhausted / doctrine halt / escalation>'
WHERE id = '<run_id>';
```
In practice, doing the INSERT and final UPDATE as a single INSERT with the terminal `status` and
`completed_at` already set (as shown in recent run rows) is also fine and saves a round trip — the two-step
open/close pattern matters more for a long-running interactive process than for one atomic tool call.

## `tasks` + `task_assignees`

Only written when a draft was actually created. One task per article, assigned to Mila, in the client's
`to_do` column (looked up via `task_columns`, not hardcoded — `task_columns.key = 'to_do'` per client).

Mila's `profiles.id`: look up fresh, don't hardcode — `SELECT id FROM public.profiles WHERE email =
'mila@spearlance.com'`.

`tasks.status` is the enum `task_status`: `to_do` | `in_progress` | `done`.
`tasks.priority` is the enum `task_priority`: `low` | `normal` | `high` | `urgent`.

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | |
| `title` | text | **NOT NULL** — e.g. `"Review blog draft: <post title>"` |
| `description` | text | link the Duda preview URL + `blog_posts.id` here |
| `status` | task_status | `'to_do'` |
| `priority` | task_priority | `'normal'` unless there's a reason to escalate |
| `assignee_user_id` | uuid | Mila's `profiles.id` |
| `creator_user_id` | uuid | leave null (system-generated) unless a routine service-account id exists |
| `due_date` | date | optional |
| `column_id` | uuid | the client's `task_columns.id` where `key = 'to_do'` |
| `created_at` / `updated_at` | timestamptz | default `now()` |

```sql
WITH col AS (
  SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do' LIMIT 1
), mila AS (
  SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com' LIMIT 1
)
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
SELECT '<client_id>', 'Review blog draft: <post title>',
       'Duda draft: <duda_preview_url>  |  blog_posts.id: <blog_post_id>',
       'to_do', 'normal', mila.id, col.id
FROM mila, col
RETURNING id;
```
Then link the assignee (some setups treat `assignee_user_id` on `tasks` as sufficient; `task_assignees`
is a separate join table — populate both so the task shows up in Mila's assigned view regardless of which
one the UI reads):
```sql
INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_profile_id>');
```
(`task_assignees` columns: `id`, `task_id`, `user_id`, `created_at` — no status/role column.)

## Known standing issues (confirmed live, 2026-09-04 — check whether still true before relying on them)

- Only 3 of 12 enrolled clients (Sunshine State, CLE Remodeling, Invictus Snowfighters) have ever had a
  `blog_content_strategy` row, so the other 9 are permanently "not due" per Step 0c. This is a content
  and revenue gap, not a routine bug — flag it every run until strategy rows exist for the rest.
- `trg_one_blog_draft_per_client_per_day` on `blog_posts` vs. routine.md Step 0c's "don't check same-day
  drafts" — see the note under `blog_posts` above.
- Sunshine State's Duda inventory (~220 posts) is heavily saturated for its Jacksonville/Duval/Nassau/St
  Johns construction/renovation/ADU vertical; expect repeated topic-space-exhaustion declines until the
  client gets a new city, vertical, or content pillar to write about.
- One broken Sunshine State Duda draft (post id `f80da16b6e694dc3857a3caae3c837ab`, titled
  `[DO NOT PUBLISH - BROKEN DRAFT, DELETE]`) needs manual deletion — the routine cannot self-delete.
