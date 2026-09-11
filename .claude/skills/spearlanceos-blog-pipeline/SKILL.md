---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS (Supabase, project chikljxwgiskyjsnjelf) column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back tables (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees, task_columns). Read this during the Publish and Record-the-Run steps of routine.md so write-backs use real column names instead of guessing.
---

# SpearlanceOS Blog Pipeline — write-back reference

This file exists because `routine.md` repeatedly references it for exact schema and it has gone
missing from `main` multiple times: prior scheduled runs kept recreating it (verified fresh via
`information_schema.columns` each time) on their own disposable branch, but that branch was never
merged back, so the next run's fresh clone of `main` never had it. **If you are Garrett: merging any
one branch that carries this file into `main` is the actual fix — recreating it again is not.**

Schema below was verified directly against `information_schema.columns` for project
`chikljxwgiskyjsnjelf` on 2026-09-11. If it and the live schema ever disagree, the live schema wins —
re-verify with `information_schema.columns` and update this file.

## blog_topics

Columns: `id`, `client_id`, `topic_title`, `description`, `summary`, `keywords` (text[]), `avatar_id`,
`priority` (text, default `'medium'`), `status` (text, default `'idea'`), `blog_post_id`,
`ai_generated` (bool), `created_at`, `strategy_batch_id`, `suggested_publish_date` (date), `category`.

Status lifecycle: `idea` → `in_progress` (topic selected/generated, being drafted) → `published`
(turned into a post — NOT necessarily live; matches the table's existing convention) . A topic that
was drafted in Duda but blocked from getting a `blog_posts` row (e.g. by a same-day cap) should be
left at its prior status (or reverted to `idea`) for reuse, not marked `published` without a linked row.

```sql
-- 4b: generate + save a new topic
INSERT INTO public.blog_topics (client_id, topic_title, summary, keywords, category, priority, status, ai_generated, suggested_publish_date)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', 'medium', 'in_progress', true, current_date)
RETURNING id;

-- Publish step 1: mark done once a blog_posts row exists
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>' WHERE id = '<topic_id>';
```

`category` is freeform text; align it to a `blog_content_strategy.content_mix` key
(`how_to`, `case_studies`, `industry_news`, `best_practices`, `company_updates`) where the topic fits,
biasing toward under-represented categories per routine.md Step 0c.

## blog_posts

Columns: `id`, `client_id`, `title`, `slug`, `meta_description`, `content`, `excerpt`, `focus_keyword`,
`seo_score`, `readability_score`, `avatar_id`, `topic_category`, `target_keywords` (text[]),
`featured_image_url`, `featured_image_alt`, `body_images` (jsonb), `status` (text, default `'draft'`),
`scheduled_for`, `published_at`, `duda_page_id`, `duda_publish_url`, `ai_model`, `generation_prompt`,
`generation_metadata` (jsonb), `created_by`, `created_at`, `updated_at`, `last_edited_by`, `topic_id`,
`scheduled_publish_date` (date), `images_generated` (bool), `auto_run_id`, `quality_scores` (jsonb),
`rejection_reason`, `revision_count`.

**Always insert with `status = 'draft'`** — never `'scheduled'` or `'published'`; that's Mila's call.
`quality_scores` should carry the doctrine QA verdict (see routine.md's rule-by-rule check) as jsonb.

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt, focus_keyword,
  topic_category, target_keywords, featured_image_url, featured_image_alt,
  status, duda_page_id, duda_publish_url, topic_id, scheduled_publish_date,
  auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<duda-path-slug>', '<meta description>', $body$<html content>$body$,
  '<excerpt>', '<focus keyword>', '<category>', ARRAY['kw1','kw2'],
  '<featured image url>', '<alt text>', 'draft', '<duda post id>', '<duda public url>',
  '<topic_id>', current_date, '<auto_run_id>', '<jsonb QA verdict>'::jsonb
) RETURNING id;
```

**A DB trigger enforces one draft per client per day.** Confirmed live via `pg_trigger` on
2026-09-11: `trg_one_blog_draft_per_client_per_day` on `public.blog_posts` (enabled). A second insert
for a client that already has a `blog_posts` row dated today will be rejected. Before attempting to
draft for a due client, check whether it already has a row for `current_date`:

```sql
SELECT id FROM public.blog_posts WHERE client_id = '<client_id>' AND created_at::date = current_date;
```

If one exists, that client's slot for today is used — do not attempt a second `create_blog_post` +
`blog_posts` insert for it this run; pick another due client instead, per routine.md's
POSTS_PER_RUN scoping. This directly affects Step 0c/0d: routine.md's due-check text says not to look
at existing drafts when deciding who is *due*, but this trigger still caps who can *receive a new
blog_posts row* today regardless of due-status — check it once you've picked a candidate client to
actually draft for, even though it doesn't change the due list itself.

## blog_auto_runs

Columns: `id`, `client_id`, `triggered_at`, `trigger_type` (text: `'scheduled'` | `'manual'`),
`status` (text, default `'running'`), `topics_generated`, `articles_generated`, `articles_passed_gate`,
`articles_flagged` (all int, default 0), `research_summary` (jsonb), `completed_at`, `error_log`,
`created_at`.

Open one row per client at the start of processing that client; close the same row at the end
(`completed` with counts, or `failed` with `error_log` explaining why that branch stopped).

```sql
-- open (top of per-client processing)
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running') RETURNING id;

-- close: success
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(), topics_generated = 1, articles_generated = 1,
    articles_passed_gate = 1, research_summary = '<jsonb summary>'::jsonb
WHERE id = '<run_id>';

-- close: declined/blocked
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why this branch stopped>'
WHERE id = '<run_id>';
```

## tasks / task_columns / task_assignees

`tasks` columns (subset relevant here): `id`, `client_id`, `title`, `description`,
`status` (enum `task_status`, default `'to_do'`), `priority` (enum `task_priority`, default `'normal'`),
`assignee_user_id`, `column_id` (FK to `task_columns.id`), `related_asset_ids` (uuid[]), `created_at`.

`task_columns` columns: `id`, `client_id`, `name`, `key` (text, e.g. `'to_do'`), `display_order`,
`is_default`, `mapped_status` (enum `task_status`). Each client has its own `task_columns` rows — look
up the client's `to_do`-keyed column before inserting a task.

`task_assignees` columns: `id`, `task_id`, `user_id`, `created_at`. A task's `assignee_user_id` and a
`task_assignees` row are both expected (write both).

Look up Mila's user id once per run from `auth.users` (do not hardcode — verify it's current):

```sql
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';
```

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';

INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id, related_asset_ids)
VALUES ('<client_id>', 'Review blog draft: <topic>', '<description with Duda preview link + post id>',
        'to_do', 'normal', '<mila_user_id>', '<to_do_column_id>', ARRAY['<blog_posts.id>']::uuid[])
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_user_id>');
```

Only create the task (and its assignee row) when a `blog_posts` row was actually created this run —
never for a declined/blocked client.
