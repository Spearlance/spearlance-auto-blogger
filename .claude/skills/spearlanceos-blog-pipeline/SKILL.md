---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the five SpearlanceOS tables the Auto Blogger routine writes to (blog_auto_runs, blog_topics, blog_posts, tasks, task_assignees). Read this during Step 0 (open the run log) and the Publish step (write-back) of routine.md.
---

# SpearlanceOS blog pipeline: write-back reference

The routine (`routine.md`) reads most of SpearlanceOS but writes to exactly five tables, project
`chikljxwgiskyjsnjelf`. Everything here was derived by introspecting the live schema
(`information_schema.columns`) and by observing actual write patterns from prior runs. If this file and
the live schema ever disagree, the live schema wins — re-derive with the query at the bottom.

## 1. `blog_auto_runs` — one row per client per tick

Open this **first**, before any research, for every client you process this tick.

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, default `gen_random_uuid()` |
| `client_id` | uuid | NOT NULL |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | NOT NULL. Use `'scheduled'` for the hourly cron; `'manual'` only when invoked ad hoc for a single `client_id` |
| `status` | text | default `'running'` → update to `'completed'` or `'failed'` when you close it out |
| `topics_generated` | int | default 0 |
| `articles_generated` | int | default 0 |
| `articles_passed_gate` | int | default 0 |
| `articles_flagged` | int | default 0 |
| `research_summary` | jsonb | free-form; see shape below |
| `completed_at` | timestamptz | set when closing the row |
| `error_log` | text | set only when `status = 'failed'` |
| `created_at` | timestamptz | default `now()` |

**Open the log** (run this before Step 1):

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id, triggered_at;
```

Keep the returned `id` as `<run_id>` for the rest of this client's processing.

**Close it on success** (use `jsonb_build_object`, not a hand-built JSON string — free text in the
`notes`/`reason_selected` fields routinely contains apostrophes and quotes that break naive string
literals):

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',
    topics_generated = 1,
    articles_generated = 1,
    articles_passed_gate = 1,
    completed_at = now(),
    research_summary = jsonb_build_object(
      'topic', '<topic title>',
      'branch', 'routine (generated topic, Step 4b)',  -- or 'routine (calendar topic, Step 4a)' / 'reactive'
      'client', '<client name>',
      'reason_selected', '<why this topic, dedup rationale>',
      'duda_post_id', '<duda post id>',
      'duda_preview_url', '<full public URL>',
      'sources_fetched', ARRAY['<url1>','<url2>'],
      'qa_verdict', 'SHIP',
      'notes', '<anything else worth carrying into the end-of-run report>'
    )
WHERE id = '<run_id>'
RETURNING id, status;
```

**Close it on failure** (bar not cleared, no fuel, no strategy row, doctrine halt, escalation):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed',
    completed_at = now(),
    error_log = '<why this branch stopped, in plain text>'
WHERE id = '<run_id>'
RETURNING id, status;
```

Every opened run log gets closed one way or the other before moving to the next client.

## 2. `blog_topics` — the topic queue

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | NOT NULL |
| `topic_title` | text | NOT NULL |
| `summary` | text | |
| `description` | text | separate from `summary`; both exist, `summary` is the one Step 4a/4b reads |
| `keywords` | text[] | |
| `category` | text | free text, aim to match `blog_content_strategy.content_mix` keys, e.g. `how_to`, `best_practices`, `case_studies`, `industry_news`, `company_updates` |
| `priority` | text | default `'medium'` |
| `status` | text | default `'idea'`. Routine lifecycle: `'idea'` → `'in_progress'` (selected/generated this run) → `'published'` (turned into a draft — NOT necessarily live) |
| `blog_post_id` | uuid | link once a `blog_posts` row exists for it (optional; many existing rows leave this null even when `status = 'published'` — don't rely on it for dedup, use `list_blog_posts` on Duda instead) |
| `ai_generated` | bool | true when the routine generated it (Step 4b) |
| `suggested_publish_date` | date | |
| `strategy_batch_id` | uuid | only populated for team-seeded calendar batches |

**Generate + save a new topic (Step 4b):**

```sql
INSERT INTO public.blog_topics (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], 'how_to', true, 'in_progress', current_date)
RETURNING id;
```

**Mark it done after the draft is created** (this means "turned into a post," not "live"):

```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>' RETURNING id, status;
```

## 3. `blog_posts` — the review/schedule queue Mila works from

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | NOT NULL |
| `title` | text | NOT NULL |
| `slug` | text | NOT NULL — use the Duda post's `path` |
| `meta_description` | text | |
| `content` | text | optional; the HTML lives in Duda, this is a convenience copy — fine to leave null if the body is long |
| `excerpt` | text | short summary, distinct from `meta_description` |
| `focus_keyword` | text | |
| `target_keywords` | text[] | |
| `topic_category` | text | matches `blog_topics.category` |
| `featured_image_url` | text | the resolved CDN URL Duda assigned, not the `image://` query string |
| `featured_image_alt` | text | |
| `status` | text | **always `'draft'`** — never `'scheduled'` or `'published'`, that's Mila's call |
| `topic_id` | uuid | FK to `blog_topics.id` |
| `auto_run_id` | uuid | FK to `blog_auto_runs.id` |
| `duda_page_id` | text | Duda's post id |
| `duda_publish_url` | text | full public URL (valid once both site and post are published) |
| `quality_scores` | jsonb | the doctrine QA verdict, see shape below |
| `ai_model` | text | default `'claude-sonnet-4-5'`; set to whatever model actually ran the session |
| `revision_count` | int | default 0 |

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, excerpt, focus_keyword, topic_category,
  target_keywords, featured_image_url, featured_image_alt, status, topic_id, auto_run_id,
  duda_page_id, duda_publish_url, quality_scores, ai_model
) VALUES (
  '<client_id>', '<title>', '<slug>', '<meta description>', '<excerpt>', '<focus keyword>', '<category>',
  ARRAY['kw1','kw2'], '<resolved image url>', '<alt text>', 'draft', '<topic_id>', '<run_id>',
  '<duda post id>', '<full public url>',
  jsonb_build_object(
    'verdict', 'SHIP',
    'word_count', 1723,
    'h2_count', 8,
    'internal_links', 11,
    'external_sources', 3,
    'checks', jsonb_build_object(
      'title','PASS','description','PASS','headings','PASS','content','PASS',
      'interlinking','PASS','schema','PASS - Article auto-applied by Duda; FAQ JSON-LD needs manual add via Duda Settings > Custom HTML',
      'intent','PASS','escalation','none'
    ),
    'sources', ARRAY['<url1>','<url2>']
  ),
  'claude-sonnet-5'
) RETURNING id;
```

## 4. `tasks` + `task_assignees` — Mila's review notification

Only insert these when a draft was actually created.

`tasks.status` is an enum (`task_status`): known values include `'to_do'`, `'in_progress'`, `'done'`.
`tasks.column_id` should point at the client's `'to_do'` row in `task_columns` (per-client, not global):

```sql
SELECT id, name, key FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

Look up Mila's user id once per run (id is stable, safe to cache within a session):

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Create the task, linking the new `blog_posts.id` via `related_asset_ids`:

```sql
INSERT INTO public.tasks (client_id, title, description, status, assignee_user_id, column_id, related_asset_ids)
VALUES (
  '<client_id>',
  'Review blog draft: <title>',
  '<preview URL, QA verdict summary, any manual follow-ups like FAQ schema>',
  'to_do',
  '<mila profile id>',
  '<to_do column id for this client>',
  ARRAY['<blog_posts.id>']::uuid[]
)
RETURNING id;
```

```sql
INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task id>', '<mila profile id>')
RETURNING id;
```

## Re-deriving this file if the schema drifts

```sql
SELECT table_name, column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema='public' AND table_name IN ('blog_auto_runs','blog_posts','blog_topics','tasks','task_assignees','task_columns')
ORDER BY table_name, ordinal_position;
```

## Why this file existed only intermittently

This repo runs in an ephemeral cloud container per session. A prior session created this file locally
but never committed it, so it was lost when the container was reclaimed and every subsequent tick had
to reconstruct the schema via `information_schema` introspection from scratch. Anything worth keeping
across ticks — including this file — has to actually be committed and pushed, not just written to disk.
