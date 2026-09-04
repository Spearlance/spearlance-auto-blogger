---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase schema and copy-pasteable SQL for the Auto Blogger's write-back tables (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read by routine.md's Publish step and Step 0 run-log step so the routine doesn't have to re-derive the schema from information_schema every tick.
---

# SpearlanceOS Blog Pipeline — write-back reference

Project id: `chikljxwgiskyjsnjelf` (production — see `supabase-prod-guard`; the routine's writes to
these five tables are pre-authorized by `routine.md`/`CLAUDE.md`, no per-run confirmation needed).

Schema below was introspected via `information_schema.columns` on 2026-09-04. If a query fails on a
missing/renamed column, re-derive from `information_schema` for the affected table and update this file.

## `blog_auto_runs` — one row per client the routine attempts, per tick

Columns: `id` (uuid, default), `client_id` (uuid, required), `triggered_at` (timestamptz, default now()),
`trigger_type` (text, required — `'scheduled'` for the hourly cron, `'manual'` for a single-client
test/dry-run), `status` (text, default `'running'` — set to `'completed'` or `'failed'`),
`topics_generated` (int, default 0), `articles_generated` (int, default 0), `articles_passed_gate` (int,
default 0), `articles_flagged` (int, default 0), `research_summary` (jsonb), `completed_at` (timestamptz),
`error_log` (text), `created_at` (timestamptz, default now()).

**Open at the start of a client's processing:**
```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

**Close on success** (after the Duda draft + `blog_posts` row exist):
```sql
UPDATE public.blog_auto_runs
SET status = 'completed', topics_generated = 1, articles_generated = 1, articles_passed_gate = 1,
    completed_at = now(), research_summary = $rs${...jsonb...}$rs$
WHERE id = '<run_id>';
```

**Close on a declined/failed branch** (QA fail, no fuel, no strategy row, doctrine halt, topic-space
saturation, escalation):
```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, in detail>'
WHERE id = '<run_id>';
```

## `blog_topics` — the topic queue

Columns: `id`, `client_id`, `topic_title` (text, required), `description` (text — NOTE: routine.md calls
this `summary`; the real column name is `description`), `keywords` (text[]), `avatar_id` (uuid),
`priority` (text, default `'medium'` — free text, e.g. `low`/`medium`/`high`), `status` (text, default
`'idea'` — use `'in_progress'` while drafting, `'published'` once turned into a `blog_posts` row),
`blog_post_id` (uuid, currently unused/nullable — the real link is `blog_posts.topic_id`, not this
column), `ai_generated` (bool), `created_at`, `strategy_batch_id` (uuid), `summary` (text — a second,
separate free-text field from `description`), `suggested_publish_date` (date), `category` (text — align
to `blog_content_strategy.content_mix` keys: `how_to`, `case_studies`, `industry_news`,
`best_practices`, `company_updates`).

**Generate + save a new topic (Step 4b):**
```sql
INSERT INTO public.blog_topics (client_id, topic_title, summary, keywords, category, priority, status, suggested_publish_date, ai_generated)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', 'high', 'in_progress', current_date, true)
RETURNING id;
```

**Mark done after a draft is created:**
```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

## `blog_posts` — one row per drafted article (Mila's review/schedule queue)

Columns: `id`, `client_id`, `title` (required), `slug` (required — use the Duda post's `path`),
`meta_description`, `content` (the full HTML body), `excerpt`, `focus_keyword`, `seo_score`,
`readability_score`, `avatar_id`, `topic_category`, `target_keywords` (text[]), `featured_image_url`,
`featured_image_alt`, `body_images` (jsonb), `status` (default `'draft'` — **always insert as
`'draft'`, never `'scheduled'`/`'published'`**), `scheduled_for`, `published_at`, `duda_page_id`,
`duda_publish_url`, `ai_model` (default `'claude-sonnet-4-5'` — leave to the column default), `topic_id`
(uuid, links to `blog_topics.id`), `auto_run_id` (uuid, links to `blog_auto_runs.id`), `quality_scores`
(jsonb — the doctrine QA verdict), `rejection_reason`, `revision_count` (default 0).

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt, focus_keyword,
  topic_category, target_keywords, featured_image_url, featured_image_alt,
  status, topic_id, auto_run_id, quality_scores, duda_page_id, duda_publish_url
) VALUES (
  '<client_id>', '<title>', '<slug>', '<meta_description>', $c$<html>$c$, '<excerpt>',
  '<focus_keyword>', '<category>', ARRAY['kw1','kw2'], '<image_url>', '<alt_text>',
  'draft', '<topic_id>', '<run_id>', $qa${"verdict": "SHIP", ...}$qa$, '<duda_post_id>', '<duda_url>'
)
RETURNING id;
```

Use dollar-quoting (`$c$...$c$`) for `content`/description fields to avoid manually escaping quotes
and apostrophes in prose.

## `tasks` + `task_assignees` — the review task for Mila

`tasks` columns of note: `id`, `client_id`, `title`, `description`, `status` (enum `task_status`, default
`'to_do'`), `priority` (enum `task_priority`, default `'normal'`), `assignee_user_id`, `column_id` (uuid
— the `task_columns` row whose `key = 'to_do'` for this client), `external_source`, `external_ref`.

Look up Mila and the client's To Do column once per run (both are stable, safe to cache within a tick):
```sql
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id, external_source, external_ref)
VALUES ('<client_id>', '<task title>', '<task description with Duda + preview links>', 'to_do', 'normal',
        '<mila_user_id>', '<to_do_column_id>', 'auto_blogger', '<blog_posts.id>')
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_user_id>')
RETURNING id;
```

## Known DB behavior not documented elsewhere in this repo

- **`trg_one_blog_draft_per_client_per_day`**: a `BEFORE INSERT` trigger on `public.blog_posts` that
  blocks a second draft for the same client on the same calendar day. `routine.md` Step 0c says
  same-day drafts should not affect due-ness — that's still correct for the *due-check*, but this
  trigger means a second attempt at drafting for an already-capped client will fail at insert time. In
  practice: before picking the highest-priority due client for the tick, check
  `SELECT 1 FROM blog_posts WHERE client_id = '<id>' AND created_at::date = current_date` and skip
  clients that already have one — they're due, but structurally can't take a second draft today.
