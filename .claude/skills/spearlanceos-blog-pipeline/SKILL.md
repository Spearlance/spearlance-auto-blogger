---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase schema and copy-pasteable SQL for the Auto Blogger's write-back tables (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Use this whenever routine.md's Publish or Record the Run steps say "see the spearlanceos-blog-pipeline skill" — it removes the need to re-discover column names via information_schema every run.
---

# SpearlanceOS Blog Pipeline — Write-Back Reference

Project id: `chikljxwgiskyjsnjelf`. All five tables below are the **only** tables the Auto Blogger
writes to; everything else in SpearlanceOS is read-only (see CLAUDE.md / routine.md). Schema captured
2026-08-07 via `information_schema.columns`; re-verify if these inserts start erroring (the app's own
migrations are the source of truth, not this file).

## 1. `blog_auto_runs` — one row per client per tick

Open at the very start of a client's processing, close once (completed or failed) at the end.

```sql
-- Open (Step 0, before any research)
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')  -- trigger_type = 'manual' for single-client test/dry-run
RETURNING id;  -- keep as <run_id>

-- Close: success
UPDATE public.blog_auto_runs
SET status = 'completed',
    topics_generated = 1,       -- 0 if an existing blog_topics row was reused instead of generated
    articles_generated = 1,
    articles_passed_gate = 1,   -- 0 if drafted but QA failed
    articles_flagged = 0,       -- count of doctrine ESCALATE flags raised
    research_summary = '{"topic": "...", "sources": ["..."], "duda_post_id": "...", "notes": "..."}'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';

-- Close: stopped without a draft (QA fail / no fuel / no strategy row / doctrine halt / escalation)
UPDATE public.blog_auto_runs
SET status = 'failed',
    error_log = 'Reason the branch was stopped, specific enough to act on.',
    completed_at = now()
WHERE id = '<run_id>';
```

Columns: `client_id`, `triggered_at` (default now()), `trigger_type` (text: `scheduled`|`manual`),
`status` (text, default `running`), `topics_generated`/`articles_generated`/`articles_passed_gate`/
`articles_flagged` (int, default 0), `research_summary` (jsonb), `completed_at`, `error_log`,
`created_at`.

## 2. `blog_topics` — the topic queue

```sql
-- 4a: pull the next due, usable topic
SELECT id, topic_title, summary, keywords, category, priority, suggested_publish_date, status
FROM public.blog_topics
WHERE client_id = '<client_id>' AND blog_post_id IS NULL
  AND status NOT IN ('published', 'in_progress')
  AND suggested_publish_date <= current_date
ORDER BY priority DESC, suggested_publish_date ASC;

-- If using an existing row, claim it immediately:
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';

-- 4b: no usable row -> generate one and save it back
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<1-3 sentence summary>', ARRAY['kw1','kw2'], '<category>',
   true, 'in_progress', current_date)
RETURNING id;  -- keep as <topic_id>

-- After the Duda draft is created (Publish step), mark the topic done — "turned into a post," not "live":
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>' WHERE id = '<topic_id>';
```

Columns: `id`, `client_id`, `topic_title` (NOT NULL), `description`, `keywords` (text[]), `avatar_id`,
`priority` (text, default `medium`), `status` (text, default `idea`; routine uses `idea`/`in_progress`/
`published`), `blog_post_id`, `ai_generated` (bool), `created_at`, `strategy_batch_id`, `summary` (text
— routine.md's Step 4a/4b query calls this `summary`, matches this column), `suggested_publish_date`
(date), `category` (text — matches `blog_content_strategy.content_mix` keys, e.g. `how_to`,
`case_studies`, `industry_news`, `best_practices`, `company_updates`).

## 3. `blog_posts` — one row per drafted article (Mila's review/schedule/publish queue)

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, focus_keyword, target_keywords,
  topic_category, featured_image_url, featured_image_alt, status, topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<duda-path-slug>', '<meta description>',
  'Full HTML lives in Duda; this column can hold the HTML too or a short pointer — Duda is the source of truth for the rendered post.',
  '<primary keyword>', ARRAY['kw1','kw2'], '<category>',
  '<main image url>', '<alt text>',
  'draft',                 -- ALWAYS 'draft'; never 'scheduled' or 'published' — that is Mila's call
  '<topic_id>', '<run_id>',
  '{"verdict":"SHIP","checks":{...}}'::jsonb
)
RETURNING id;  -- keep as <blog_post_row_id>, used above to close out blog_topics.blog_post_id
```

Columns worth knowing beyond the obvious: `slug` is `text NOT NULL` (use the Duda post's `path`),
`status` defaults to `'draft'`, `duda_page_id`/`duda_publish_url` exist if you want to store the Duda
post id/URL directly on this row too, `body_images` (jsonb), `seo_score`/`readability_score` (numeric,
optional), `revision_count` (int, default 0), `rejection_reason` (text, for QA-failed attempts you still
want to log — optional, most failed attempts only need a `blog_auto_runs.error_log` entry).

## 4. `tasks` + `task_assignees` — Mila's review ping (only when a draft was created)

```sql
-- Look up Mila once per run (cheap to just do it every time)
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';

-- Find the client's "to do" column
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';

INSERT INTO public.tasks (client_id, title, description, status, priority, column_id, creator_user_id)
VALUES (
  '<client_id>', 'Review draft: <post title>',
  'Duda draft (unpublished): <duda editor url>, post id <duda_post_id>. Doctrine QA verdict: <SHIP/etc>. <any follow-ups, e.g. FAQ schema needs manual add>.',
  'to_do', 'normal', '<to_do_column_id>', '<mila_user_id>'
)
RETURNING id;  -- keep as <task_id>

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_user_id>');
```

`tasks.status` is the enum `task_status` (`to_do`/`in_progress`/`done`, matches `task_columns.key`
values); `tasks.priority` is enum `task_priority` (`normal` is a safe default). `task_columns` is
per-client — always look it up rather than assuming an id.

## Full write-back order (Publish step)

1. `blog_topics` → `status = 'published'`, set `blog_post_id`
2. `blog_posts` → insert draft row, `status = 'draft'`
3. `blog_auto_runs` → close out (`completed` or `failed`)
4. `tasks` + `task_assignees` → only if a draft was created
