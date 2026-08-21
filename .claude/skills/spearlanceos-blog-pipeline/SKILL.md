---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase write-back schema and SQL for the Auto Blogger routine (project chikljxwgiskyjsnjelf). Use whenever routine.md's Publish / Record the Run steps say to write to blog_topics, blog_posts, blog_auto_runs, tasks, or task_assignees. Reconstructed live from information_schema on 2026-08-21 after this file was found missing despite being referenced throughout routine.md.
---

# SpearlanceOS Blog Pipeline — write-back reference

Project id: `chikljxwgiskyjsnjelf`. The routine writes **only** to the five tables below; everything
else in SpearlanceOS is read-only. Never call `blog-*` edge functions.

## Guardrail you WILL hit: one `blog_posts` draft per client per day

A DB trigger `guard_one_blog_draft_per_client_per_day` (BEFORE INSERT on `blog_posts`) raises
`P0001` if a row already exists for that `client_id` where `created_at::date = now()::date`. This is
not documented anywhere else. Practical implications:

- Before attempting Step 4 research for a due client, check whether `blog_posts` already has a row
  for that `client_id` created today. If so, **do not** run the pipeline again for that client this
  due-cycle — there is nothing to write back, and the insert will fail.
- If a `create_blog_post` Duda call already happened this tick (draft exists live in Duda) and the
  guard then rejects the `blog_posts` insert, **do not retry inserting under a different date** — that
  falsifies `created_at`. Leave the Duda draft as an untracked orphan, note it in `blog_auto_runs.error_log`
  and the end-of-run report, and either let it reconcile naturally on that client's next due day, or
  flag for Garrett/Mila to add the `blog_posts`/`tasks` rows manually.

## 1. `blog_topics` — mark the topic used

```sql
-- 4a: using an existing calendar topic
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';

-- 4b: generating a new topic
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', true, 'in_progress', current_date)
RETURNING id;

-- After a draft is created (either branch):
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>' WHERE id = '<topic_id>';
```

Notes: `keywords` is `text[]`. `priority` defaults to `'medium'` (text, not numeric — no enforced enum
seen, treat as free text like `'high'`/`'medium'`/`'low'`). `status` values observed in use: `idea`,
`in_progress`, `published`. Never leave a topic `in_progress` if you decide not to draft it — revert to
`idea` (see prior-tick precedent) so a future run can reconsider it.

## 2. `blog_posts` — one row per drafted article (Mila's review/schedule queue)

Columns (types): `id` uuid pk, `client_id` uuid NOT NULL, `title` text NOT NULL, `slug` text NOT NULL,
`meta_description` text, `content` text, `excerpt` text, `focus_keyword` text, `seo_score` numeric,
`readability_score` numeric, `avatar_id` uuid, `topic_category` text, `target_keywords` text[],
`featured_image_url` text, `featured_image_alt` text, `body_images` jsonb, `status` text default
`'draft'`, `scheduled_for` timestamptz, `published_at` timestamptz, `duda_page_id` text,
`duda_publish_url` text, `ai_model` text default `'claude-sonnet-4-5'`, `generation_prompt` text,
`generation_metadata` jsonb, `created_by` uuid, `created_at`/`updated_at` timestamptz default now(),
`last_edited_by` uuid, `topic_id` uuid, `scheduled_publish_date` date, `images_generated` boolean
default false, `auto_run_id` uuid, `quality_scores` jsonb, `rejection_reason` text, `revision_count`
integer default 0.

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, target_keywords,
   featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
   topic_id, auto_run_id, quality_scores, topic_category)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta_description>', '<full html or markdown body>',
   '<primary keyword>', ARRAY['kw1','kw2'],
   '<thumbnail/main image url>', '<alt text>',
   'draft',                       -- ALWAYS 'draft'; never 'scheduled'/'published' — that is Mila's call
   '<duda post id>', '<duda editor/preview url>',
   '<blog_topics.id>', '<blog_auto_runs.id>',
   '{"doctrine_verdict":"SHIP","word_count":1650,"h2_count":7,"internal_links":9,"sources_cited":3}'::jsonb,
   '<category>')
RETURNING id;
```

`slug` is required NOT NULL — use the Duda post's `path`. Set `content` to the actual HTML/body you
sent to `create_blog_post` (needed for QA traceability; don't leave it null if you have it).

## 3. `blog_auto_runs` — one row per client processed per tick, system of record for what a tick did

Columns: `id` uuid pk, `client_id` uuid NOT NULL, `triggered_at` timestamptz default now(),
`trigger_type` text NOT NULL (`'scheduled'` for the hourly cron, `'manual'` only for an ad hoc
single-`client_id` invocation), `status` text default `'running'`, `topics_generated` integer default 0,
`articles_generated` integer default 0, `articles_passed_gate` integer default 0, `articles_flagged`
integer default 0, `research_summary` jsonb, `completed_at` timestamptz, `error_log` text,
`created_at` timestamptz default now().

```sql
-- Open, before any research/drafting for this client:
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;

-- Close, once this client's branch is fully resolved (drafted or stopped):
UPDATE public.blog_auto_runs
SET status = 'completed',              -- or 'failed' if the branch was stopped without a draft
    completed_at = now(),
    topics_generated = 1,              -- 0 if an existing calendar topic was reused
    articles_generated = 1,
    articles_passed_gate = 1,          -- 0 if QA killed it
    error_log = '<why, if stopped, or a short summary either way>'
WHERE id = '<run_id>';
```

Every opened row gets closed once, same tick, before moving to the next client.

## 4. `tasks` + `task_assignees` — Mila's review task (only when a draft was created)

```sql
-- Look up Mila's user id and the client's "to_do" task_columns.id once:
SELECT id FROM auth.users WHERE email = '<mila's email>';   -- or public.users if mirrored there
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
-- task_columns.mapped_status ('to_do' etc.) and task_status/task_priority are Postgres enums
-- (USER-DEFINED in information_schema) — use the same string values task_columns already uses.

INSERT INTO public.tasks
  (client_id, title, description, status, priority, assignee_user_id, column_id, related_asset_ids)
VALUES
  ('<client_id>', 'Review blog draft: <post title>',
   'Auto Blogger created a draft for review. Duda preview: <url>. Topic: <topic>. QA verdict: SHIP.',
   'to_do', 'normal', '<mila_user_id>', '<todo_column_id>', ARRAY['<blog_posts.id>']::uuid[])
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_user_id>');
```

`tasks.status`/`tasks.priority` are enum types (`task_status`, `task_priority`) — pass the same string
values `task_columns.mapped_status` already uses (`'to_do'` observed as a valid `task_status` value);
don't invent a new one.
