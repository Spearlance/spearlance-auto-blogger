---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase table schemas, status values, and copy-pasteable SQL for the four tables the Auto Blogger routine writes to (blog_topics, blog_posts, blog_auto_runs, tasks/task_assignees). Read whenever routine.md's Publish or run-log steps say to consult this skill.
---

# SpearlanceOS Blog Pipeline — write-back reference

Project id: **`chikljxwgiskyjsnjelf`**. Everything below is schema confirmed live via
`information_schema.columns` and enum introspection — re-verify if a query errors on an unknown
column, since this file can drift from the live schema.

The routine writes to exactly four tables: `blog_topics`, `blog_posts`, `blog_auto_runs`, and
`tasks` + `task_assignees`. Nothing else. Never call SpearlanceOS's `blog-*` edge functions.

## 1. `blog_auto_runs` — one row per client per tick

Open at the **start** of processing a client (before research/drafting), close it at the **end**.

Columns: `id` (uuid, default), `client_id` (uuid, required), `triggered_at` (timestamptz, default
now()), `trigger_type` (text, required — use `'scheduled'` for the hourly cron; `'manual'` only for
an ad hoc single-`client_id` invocation), `status` (text, default `'running'`), `topics_generated`
(int, default 0), `articles_generated` (int, default 0), `articles_passed_gate` (int, default 0),
`articles_flagged` (int, default 0), `research_summary` (jsonb, nullable), `completed_at`
(timestamptz, nullable), `error_log` (text, nullable), `created_at` (timestamptz, default now()).

**Open:**
```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```
Keep the returned `id` as `<run_id>` for `blog_posts.auto_run_id` below.

**Close — success (draft created):**
```sql
UPDATE public.blog_auto_runs
SET status = 'completed', topics_generated = <n>, articles_generated = 1,
    articles_passed_gate = 1, completed_at = now()
WHERE id = '<run_id>';
```

**Close — stopped branch (no draft):**
```sql
UPDATE public.blog_auto_runs
SET status = 'failed', error_log = '<why: QA fail / no fuel / no strategy row / doctrine halt / escalation / saturated topic space / etc.>',
    completed_at = now()
WHERE id = '<run_id>';
```

Note: in practice a single `INSERT ... RETURNING id` followed immediately by an `UPDATE` is fine, but
when the whole run resolves quickly (e.g. a same-tick skip with no research attempted) it's also
acceptable to combine into one INSERT with `status` and `completed_at` already set, as done for
already-decided skips (e.g. "already served today", "no strategy configured").

## 2. `blog_topics` — the topic queue

Columns: `id`, `client_id`, `topic_title` (required), `description`, `keywords` (text[]), `avatar_id`,
`priority` (text, default `'medium'`), `status` (text, default `'idea'`), `blog_post_id` (uuid,
nullable — set when a post is created from this topic), `ai_generated` (bool, default false),
`created_at`, `strategy_batch_id`, `summary`, `suggested_publish_date` (date), `category`.

Status values used by the routine: `'idea'` (untouched), `'in_progress'` (selected/generated this
run, being drafted), `'published'` (turned into a post — Duda draft created, not necessarily live).

**Select next due topic** (routine.md Step 4a):
```sql
SELECT id, topic_title, summary, keywords, category, priority, suggested_publish_date
FROM public.blog_topics
WHERE client_id = '<client_id>' AND blog_post_id IS NULL
  AND status NOT IN ('published', 'in_progress')
  AND suggested_publish_date <= current_date
ORDER BY priority DESC, suggested_publish_date ASC;
```

**Mark a selected calendar topic in progress:**
```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**Generate + save a new topic** (routine.md Step 4b — note the real column is `summary`, not
`description`, and there is no `ai_model` column on this table):
```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date, priority)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>']::text[], '<category>', true, 'in_progress', current_date, 'medium')
RETURNING id;
```

**Mark done once the Duda draft is created** (Publish step, item 1):
```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>' WHERE id = '<topic_id>';
```

## 3. `blog_posts` — Mila's draft/schedule/publish queue (one row per article)

Columns: `id`, `client_id` (required), `title` (required), `slug` (required), `meta_description`,
`content`, `excerpt`, `focus_keyword`, `seo_score` (numeric), `readability_score` (numeric),
`avatar_id`, `topic_category`, `target_keywords` (text[]), `featured_image_url`,
`featured_image_alt`, `body_images` (jsonb), `status` (text, default `'draft'` — **always insert as
`'draft'`, never `'scheduled'`/`'published'`, that's Mila's call**), `scheduled_for` (timestamptz),
`published_at`, `duda_page_id`, `duda_publish_url`, `ai_model` (text, default
`'claude-sonnet-4-5'`), `generation_prompt`, `generation_metadata` (jsonb), `created_by`,
`created_at`, `updated_at`, `last_edited_by`, `topic_id` (uuid, links back to `blog_topics.id`),
`scheduled_publish_date` (date), `images_generated` (bool, default false), `auto_run_id` (uuid,
links to `blog_auto_runs.id`), `quality_scores` (jsonb — put the doctrine compliance verdict here),
`rejection_reason`, `revision_count` (int, default 0).

**Insert after the Duda draft is created:**
```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, target_keywords,
   topic_category, featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
   topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta description>', '<full HTML/markdown body>',
   '<focus keyword>', ARRAY['<kw1>','<kw2>']::text[], '<category>', '<image url>', '<alt text>',
   'draft', '<duda post id>', '<duda preview/editor url>', '<topic_id>', '<run_id>',
   '{"verdict":"SHIP","title":"PASS","description":"PASS","headings":"PASS","content":"PASS","interlinking":"PASS","schema":"PASS (Article; FAQ needs manual Custom HTML)","intent":"PASS","escalation":"none","word_count":1650,"h2_count":7,"internal_links":9,"sources":["https://...","https://..."]}'::jsonb)
RETURNING id;
```

## 4. `tasks` + `task_assignees` — Mila's review ping (only when a draft was created)

`tasks.status`/`task_priority` are Postgres enums: `task_status` = `to_do | in_progress | done`;
`task_priority` = `low | normal | high | urgent`. Every client has default `task_columns` rows keyed
`to_do` / `in_progress` / `done` (`is_default = true`); use the client's `to_do` column's `id` for
`tasks.column_id`.

Mila's `profiles.id`: look up fresh each run (do not hardcode) —
```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Get the client's `to_do` column id:
```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

**Insert the task, then the assignee row:**
```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, column_id, assignee_user_id, creator_user_id)
VALUES ('<client_id>', 'Review blog draft: <title>',
        'Auto Blogger created a draft for review. Duda preview: <preview_url>. Keyword: <focus keyword>. Intent: <intent>.',
        'to_do', 'normal', '<to_do column id>', '<mila profile id>', '<mila profile id>')
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task id>', '<mila profile id>');
```
(`assignee_user_id` on `tasks` and the `task_assignees` join row are both used by the app; set both.)

## Write order for a client that produced a draft

1. Duda `create_blog_post` → capture `duda_page_id` + preview URL.
2. `blog_topics` → mark `status = 'published'`, set `blog_post_id`.
3. `blog_posts` → insert the article row (`status = 'draft'`, `topic_id`, `auto_run_id`, `quality_scores`).
4. `blog_auto_runs` → close the run row opened at the start (`status = 'completed'`).
5. `tasks` + `task_assignees` → create Mila's review task.

For a client whose branch was stopped instead (no draft): only step 4, with `status = 'failed'` and
a specific `error_log`.
