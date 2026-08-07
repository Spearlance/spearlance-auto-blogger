---
name: spearlanceos-blog-pipeline
description: SpearlanceOS Supabase schema + copy-pasteable SQL for the Auto Blogger's write-back tables (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read this before writing to any of them so column names, enum values, and FK relationships are correct on the first try.
---

# SpearlanceOS Blog Pipeline — Write-Back Reference

`routine.md` writes to five tables in the `chikljxwgiskyjsnjelf` Supabase project. Every other table
is read-only. This file exists because the schema isn't guessable from column names alone (several
enum types, one FK chain, one client-scoped lookup table) — read it once per run instead of
re-discovering the schema via `information_schema` every tick.

> Provenance: reconstructed from live `information_schema` introspection on 2026-08-07 (multiple
> independent ticks had been doing this same introspection by hand because this file didn't exist —
> see `blog_auto_runs.error_log` entries from that day for the paper trail). If a column listed here
> ever doesn't exist, re-introspect and fix this file rather than guessing.

## Run lifecycle: `blog_auto_runs`

One row per client per tick. Open it **before** any research/drafting; close it once, at the end.

```sql
-- Open (call this first, per client, before Step 1)
INSERT INTO public.blog_auto_runs (client_id, trigger_type)
VALUES ('<client_id>', 'scheduled')  -- 'manual' only for an ad-hoc single-client_id invocation
RETURNING id;  -- keep as <run_id>; column default status='running'
```

```sql
-- Close: success (a draft was created)
UPDATE public.blog_auto_runs SET
  status = 'completed',
  topics_generated = 1, articles_generated = 1, articles_passed_gate = 1, articles_flagged = 0,
  completed_at = now(),
  research_summary = '{"topic":"...", "duda_post_id":"...", "duda_url":"...", "sources":[...]}'::jsonb
WHERE id = '<run_id>';
```

```sql
-- Close: declined (no draft — QA fail, no fuel, saturation, no strategy row, doctrine halt, escalation)
UPDATE public.blog_auto_runs SET
  status = 'failed',
  completed_at = now(),
  error_log = 'Free-text reason. Prior ticks write a detailed narrative here (what was checked,
    what was found, why nothing shipped, standing asks) — this is the only durable record of a
    declined tick, since there is no markdown ledger.'
WHERE id = '<run_id>';
```

`status` is free text in practice, but only `'running'` (default), `'completed'`, `'failed'` are used —
don't invent a third value. `topics_generated` / `articles_generated` / `articles_passed_gate` /
`articles_flagged` are integers, default 0.

## Topic queue: `blog_topics`

Selection query is in `routine.md` Step 4a. Columns actually present (routine.md's own SQL snippets
predate a few of these — trust this list over that one):

`id, client_id, topic_title, description, summary, keywords[], category, priority` (free text, this
repo's convention is `'low'|'medium'|'high'`), `status` (free text; this repo's convention is
`'idea'|'in_progress'|'published'`), `blog_post_id` (FK → `blog_posts.id`, nullable), `ai_generated`
(bool), `suggested_publish_date`, `strategy_batch_id`, `avatar_id`, `created_at`.

```sql
-- 4b: generating a new topic (mark in_progress immediately so the same tick / a racing tick can't reselect it)
INSERT INTO public.blog_topics (client_id, topic_title, summary, keywords, category, priority, status, ai_generated, suggested_publish_date)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', 'medium', 'in_progress', true, current_date)
RETURNING id;  -- keep as <topic_id>
```

```sql
-- After the Duda draft is created and blog_posts row exists: mark the topic done and link it
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

Note the FK: `blog_topics.blog_post_id` points at `blog_posts.id`, not at the Duda post ID. Step 4a's
filter (`blog_post_id IS NULL AND status NOT IN ('published','in_progress')`) is what keeps a topic
from being reselected once it's in flight or done — always set both fields together at the end.

## The article record: `blog_posts`

One row per drafted article — Mila's actual review/schedule record (separate from the Duda post
itself, which holds the live HTML). `status` is always `'draft'` when the routine writes it; scheduling
or publishing is Mila's action, not the routine's.

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, focus_keyword, topic_category,
  target_keywords, featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
  ai_model, topic_id, auto_run_id, images_generated, quality_scores
) VALUES (
  '<client_id>', '<title>', '<duda-path-slug>', '<meta description>', '<full article HTML>',
  '<focus keyword>', '<category>', ARRAY['kw1','kw2'], '<hero image url>', '<alt text>',
  'draft', '<duda post id>', '<duda public/preview url>', 'claude-sonnet-5',
  '<topic_id>', '<run_id>', true,
  '{"verdict":"SHIP","word_count":1700,"checks":{...},"sources":[...]}'::jsonb
) RETURNING id;  -- keep as <blog_posts_id>, used above to close out blog_topics
```

`content` should hold the actual article HTML (not a pointer/placeholder) — this is the record Mila
and the SpearlanceOS UI read without needing to open Duda. `slug` should match the Duda post's `path`
(fetch it via `get_blog_post` after creation if `create_blog_post`'s response doesn't include it).
`seo_score` / `readability_score` are nullable numeric columns — leave them null rather than inventing
a precision the routine doesn't actually measure.

## Review task: `tasks` + `task_assignees`

Only insert when a draft was actually created. `task_columns` is **client-scoped** — the `'to_do'`
column's `id` differs per client, so look it up per client, not once globally:

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

Find Mila once per run (or cache it — her `profiles.id` doesn't change):

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

`tasks.status` is enum `task_status`: `to_do | in_progress | done`. `tasks.priority` is enum
`task_priority`: `low | normal | high | urgent`.

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES (
  '<client_id>', 'Review draft: <article title> (<client name>)',
  'Auto Blogger created a new draft... Preview/edit: <duda editor url>. Doctrine QA verdict: SHIP (...). Follow-up: <e.g. FAQ schema needs manual Duda Custom HTML add>.',
  'to_do', 'normal', '<mila profile id>', '<client''s to_do task_columns id>'
) RETURNING id;  -- keep as <task_id>
```

```sql
INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila profile id>');
```

## Quick sanity queries

```sql
-- Confirm a client's due-check inputs in one shot
SELECT c.name, c.site_id, s.selected_days, s.content_mix
FROM public.clients c
LEFT JOIN public.blog_content_strategy s ON s.client_id = c.id AND s.is_global = false
WHERE c.id = '<client_id>';

-- Today's already-processed clients (avoid re-deriving "who ran this hour" from scratch)
SELECT client_id, status, created_at, completed_at, error_log
FROM public.blog_auto_runs WHERE created_at >= current_date ORDER BY created_at DESC;
```

## If a column here turns out wrong

Re-run:
```sql
SELECT table_name, column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_schema = 'public' AND table_name IN ('blog_topics','blog_posts','blog_auto_runs','tasks','task_assignees','task_columns')
ORDER BY table_name, ordinal_position;
```
and fix this file in the same tick — don't just work around it silently, or the next tick pays the
same discovery cost again.
