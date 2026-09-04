---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase column names, status/enum values, and copy-pasteable SQL for the Auto Blogger's writes to blog_topics, blog_posts, blog_auto_runs, tasks, and task_assignees. Read this whenever routine.md says to write to one of those five tables.
---

# SpearlanceOS blog pipeline: write reference

Project id: `chikljxwgiskyjsnjelf` (production — confirm with `list_branches`; `is_default: true` /
`project_ref: chikljxwgiskyjsnjelf` is prod, `zlljsdaxsggkasvympku` is the persistent dev branch).
Schema below was introspected live from `information_schema.columns` / `pg_enum` on 2026-09-04;
re-verify if these queries ever error, since this file can drift from the live DB.

Every INSERT/UPDATE here is a production write. Per the `supabase-prod-guard` skill, ordinary ad hoc
writes need Garrett's explicit yes — but the Auto Blogger's writes to exactly these five tables are
pre-authorized as this routine's normal designed behavior (see `routine.md` and `CLAUDE.md`). Do not
extend that standing authorization to any other table or to `blog-*` edge functions.

## `blog_auto_runs` — one row per client processed, per tick

Columns: `id` (uuid, default), `client_id` (uuid, required), `triggered_at` (timestamptz, default now()),
`trigger_type` (text, required — use `'scheduled'` for the hourly cron, `'manual'` for a single
`client_id` test run), `status` (text, default `'running'`), `topics_generated` (int, default 0),
`articles_generated` (int, default 0), `articles_passed_gate` (int, default 0), `articles_flagged` (int,
default 0), `research_summary` (jsonb), `completed_at` (timestamptz), `error_log` (text), `created_at`
(timestamptz, default now()).

No formal enum on `status`; the routine's own convention (matches every row written so far) is
`'running'` → `'completed'` (a draft was created) or `'failed'` (branch stopped before a draft — QA
fail, no fuel, no strategy row, doctrine halt, escalation, topic-space exhausted, or nothing left to do
because every due client is already capped for the day). Use `'failed'` for any "no draft created"
outcome, not only genuine errors — that's the established convention in this table.

Open (start of a client's processing):
```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Close on success:
```sql
UPDATE public.blog_auto_runs
SET status = 'completed',
    topics_generated = <0 or 1>,
    articles_generated = 1,
    articles_passed_gate = 1,
    articles_flagged = 0,
    research_summary = '<jsonb: {"sources": [...], "keyword": "...", "internal_links": [...]}>'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';
```

Close on stopped branch:
```sql
UPDATE public.blog_auto_runs
SET status = 'failed',
    error_log = '<why: bar not cleared / no qualifying feed item / no fuel / no strategy configured / already capped for today>',
    completed_at = now()
WHERE id = '<run_id>';
```

## `blog_topics` — topic queue

Columns: `id` (uuid, default), `client_id` (uuid, required), `topic_title` (text, required), `description`
(text), `keywords` (text[]), `avatar_id` (uuid), `priority` (text, default `'medium'`), `status` (text,
default `'idea'`), `blog_post_id` (uuid — set once a post exists), `ai_generated` (boolean, default
false), `created_at` (timestamptz, default now()), `strategy_batch_id` (uuid), `summary` (text),
`suggested_publish_date` (date), `category` (text).

No formal enum on `status`; observed values: `'idea'`, `'in_progress'`, `'published'`. `'published'` here
means "turned into a post" (draft), not "live" — matches the table's existing convention per `routine.md`.

Generate + save a new topic (Step 4b):
```sql
INSERT INTO public.blog_topics (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>', true, 'in_progress', current_date)
RETURNING id;
```

Mark a used calendar topic in progress (Step 4a):
```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

Mark the topic done once a post is created (Publish step):
```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_post_id>' WHERE id = '<topic_id>';
```

## `blog_posts` — the draft/review/schedule record Mila works from

Columns: `id` (uuid, default), `client_id` (uuid, required), `title` (text, required), `slug` (text,
required), `meta_description` (text), `content` (text), `excerpt` (text), `focus_keyword` (text),
`seo_score` (numeric), `readability_score` (numeric), `avatar_id` (uuid), `topic_category` (text),
`target_keywords` (text[]), `featured_image_url` (text), `featured_image_alt` (text), `body_images`
(jsonb), `status` (text, default `'draft'`), `scheduled_for` (timestamptz), `published_at` (timestamptz),
`duda_page_id` (text), `duda_publish_url` (text), `ai_model` (text, default `'claude-sonnet-4-5'`),
`generation_prompt` (text), `generation_metadata` (jsonb), `created_by` (uuid), `created_at`
(timestamptz, default now()), `updated_at` (timestamptz, default now()), `last_edited_by` (uuid),
`topic_id` (uuid), `scheduled_publish_date` (date), `images_generated` (boolean, default false),
`auto_run_id` (uuid), `quality_scores` (jsonb), `rejection_reason` (text), `revision_count` (integer,
default 0).

**Note:** the column is `duda_page_id`, not `duda_post_id` — using the wrong name will error.

**Hard constraint:** a DB trigger (`trg_one_blog_draft_per_client_per_day`) rejects a second
`blog_posts` insert for the same `client_id` on the same day. Before attempting Draft → QA → Publish
for a due client, check whether it already has a `blog_posts` row with `created_at >= current_date`:
```sql
SELECT id, title, created_at FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at >= current_date;
```
If one exists, that client is done for today — do not attempt another insert (it will be rejected by
the trigger), and don't burn a research/QA cycle on it. This check is about the DB's hard cap, not
about second-guessing whether "enough" content went out — `routine.md` Step 0c's instruction not to
check existing drafts is about *due-ness*, not about whether an insert will physically succeed.

Insert after a Duda draft is created (`status` is always `'draft'` — never `'scheduled'`/`'published'`,
that's Mila's call):
```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, focus_keyword, target_keywords,
  topic_category, featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
  topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<slug>', '<meta_description>', '<html_content_or_summary>',
  '<focus_keyword>', ARRAY['<kw1>','<kw2>'], '<category>', '<thumbnail_url>', '<alt_text>',
  'draft', '<duda_page_id>', '<duda_publish_url>', '<topic_id>', '<auto_run_id>',
  '<jsonb QA verdict, e.g. {"doctrine_ship": true, "word_count": 1600, "h2_count": 5, "internal_links": 7, "sources": [...]}>'::jsonb
)
RETURNING id;
```

## `tasks` + `task_assignees` — one review task for Mila per draft created

`tasks` columns: `id` (uuid, default), `client_id` (uuid, required), `title` (text, required),
`description` (text), `status` (task_status enum, default `'to_do'`), `priority` (task_priority enum,
default `'normal'`), `assignee_user_id` (uuid), `creator_user_id` (uuid), `due_date` (date),
`related_asset_ids` (uuid[]), `related_meeting_ids` (uuid[]), `activity_log` (text), `created_at`/
`updated_at` (timestamptz), `linked_channel_id` (uuid), `parent_task_id` (uuid), `subtask_order`
(integer), `color` (varchar), `is_recurring` (boolean), `recurrence_pattern` (jsonb),
`parent_recurring_task_id` (uuid), `next_occurrence_date` (date), `is_recurring_instance` (boolean),
`original_due_date` (date), `column_id` (uuid — FK to `task_columns.id`, **client-scoped**, not global),
`started_at`, `completed_at` (timestamptz), `external_source`, `external_ref` (text).

`task_status` enum: `to_do`, `in_progress`, `done`. `task_priority` enum: `low`, `normal`, `high`,
`urgent`.

`task_assignees` columns: `id` (uuid, default), `task_id` (uuid, required), `user_id` (uuid, required),
`created_at` (timestamptz, default now()).

Mila's `profiles` row: `id = '4b9df73e-bc38-49ef-840e-75fdb0b29c29'`, `email = 'mila@spearlance.com'`.
Re-verify by email rather than hardcoding the id blindly, in case it ever changes:
```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Look up the client's `to_do` column (per-client, not global — every client observed so far has one with
`key = 'to_do'`, seed one via `is_default` rows if missing):
```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

Create the review task, then assign Mila:
```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES (
  '<client_id>', 'Review blog draft: <post title>',
  'Auto Blogger created a draft for review. Duda preview: <duda_publish_url>. Keyword: <focus_keyword>. Sources: <source urls>.',
  'to_do', 'normal', '<mila_profile_id>', '<to_do_column_id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_profile_id>');
```
