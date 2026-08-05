---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase write-back for the Auto Blogger routine — column names, status values, and copy-pasteable SQL for blog_auto_runs, blog_topics, blog_posts, tasks, and task_assignees. Read whenever routine.md's "Open the run log" or "Publish" steps say to consult this skill.
---

# SpearlanceOS Blog Pipeline — write-back reference

Project id: **`chikljxwgiskyjsnjelf`**. Use `mcp__Supabase__execute_sql` (never `read_only`). The
routine writes **only** to the five tables below; everything else in SpearlanceOS is read-only, and
`blog-*` edge functions are never called.

Schema below was captured by direct `information_schema.columns` introspection (source of truth if this
file and the live DB ever disagree — re-introspect, don't guess).

## 1. `blog_auto_runs` — one row per client per tick

Open **before** any research (Step 0d), close exactly once at the end of that client's processing.

```sql
-- Open (per client, at the very start of processing that client)
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')  -- trigger_type = 'manual' only for an ad hoc single-client test run
RETURNING id;  -- keep as <run_id>

-- Close: success path
UPDATE public.blog_auto_runs
SET status = 'completed',
    topics_generated = <0 or 1>,       -- 1 if Step 4b generated a new topic this run
    articles_generated = 1,
    articles_passed_gate = 1,
    articles_flagged = <0 or 1>,       -- 1 if QA flagged something non-blocking (e.g. FAQ schema follow-up)
    research_summary = '<jsonb of sources/keywords used>'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';

-- Close: stopped-branch path (QA fail, no fuel, no strategy row, doctrine halt, no usable topic/angle)
UPDATE public.blog_auto_runs
SET status = 'failed',
    completed_at = now(),
    error_log = '<why this client''s branch was stopped instead of producing a draft>'
WHERE id = '<run_id>';
```

Columns: `id` (uuid, pk), `client_id`, `triggered_at` (default now()), `trigger_type` (text —
`'scheduled'` | `'manual'`), `status` (text — `'running'` default | `'completed'` | `'failed'`),
`topics_generated` (int, default 0), `articles_generated` (int, default 0), `articles_passed_gate`
(int, default 0), `articles_flagged` (int, default 0), `research_summary` (jsonb, nullable),
`completed_at` (timestamptz, nullable), `error_log` (text, nullable), `created_at` (default now()).

## 2. `blog_topics` — mark the used/generated topic done

```sql
-- If a calendar topic (Step 4a) was used, mark it in_progress the moment you start drafting:
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';

-- If none was usable, generate + insert one (Step 4b):
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, priority, status, ai_generated, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>', 'medium',
   'in_progress', true, current_date)
RETURNING id;  -- keep as <topic_id>

-- After the draft is created in Duda (Publish step), mark it done — "done" means "turned into a
-- post," not "live":
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>' WHERE id = '<topic_id>';
```

Columns: `id`, `client_id`, `topic_title`, `description`, `summary`, `keywords` (text[]), `category`,
`priority` (text, default `'medium'`), `status` (text, default `'idea'` — observed values: `idea`,
`in_progress`, `published`), `blog_post_id` (uuid, nullable, fk-ish to `blog_posts.id`), `ai_generated`
(bool, default false), `suggested_publish_date` (date), `avatar_id`, `strategy_batch_id`, `created_at`.

Note: `blog_topics` has **no** `updated_at` column — don't try to write one.

## 3. `blog_posts` — the draft/review record Mila works from

Insert **after** the Duda draft is created, once you have the Duda post id.

```sql
INSERT INTO public.blog_posts
  (client_id, topic_id, auto_run_id, title, slug, meta_description, excerpt, focus_keyword,
   target_keywords, content, featured_image_url, featured_image_alt, status, duda_page_id,
   duda_publish_url, quality_scores)
VALUES
  ('<client_id>', '<topic_id>', '<run_id>', '<title>', '<duda-path-slug>', '<meta description>',
   '<short excerpt>', '<primary keyword>', ARRAY['<kw1>','<kw2>'], '<full HTML body actually published>',
   '<main image url>', '<alt text>', 'draft', '<duda post id>', '<duda preview/editor url>',
   '<jsonb QA verdict>'::jsonb)
RETURNING id;
```

Columns: `id`, `client_id`, `topic_id`, `title`, `slug`, `meta_description`, `content`, `excerpt`,
`focus_keyword`, `seo_score` (numeric, nullable), `readability_score` (numeric, nullable), `avatar_id`,
`topic_category`, `target_keywords` (text[]), `featured_image_url`, `featured_image_alt`, `body_images`
(jsonb), `status` (text, default `'draft'` — **always insert as `'draft'`**; never `'scheduled'` or
`'published'`, that's Mila's call), `scheduled_for`, `published_at`, `duda_page_id`, `duda_publish_url`,
`ai_model` (default `'claude-sonnet-4-5'`), `generation_prompt`, `generation_metadata` (jsonb),
`created_by`, `created_at`, `updated_at`, `last_edited_by`, `scheduled_publish_date`,
`images_generated` (bool, default false), `auto_run_id`, `quality_scores` (jsonb — put the doctrine
rule-by-rule verdict here), `rejection_reason`, `revision_count` (int, default 0).

`quality_scores` example payload:
```json
{"verdict": "SHIP", "word_count": 1650, "h2_count": 6, "internal_links": 7,
 "sources_cited": ["https://...", "https://..."], "faq_schema_manual_followup": true}
```

## 4. `tasks` + `task_assignees` — the review task for Mila

Only when a draft was actually created. Look up the client's `'to_do'` column in `task_columns`, and
Mila's user id from `profiles`.

```sql
-- Mila's user id (stable — safe to reuse, but re-verify if this ever returns nothing):
-- SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
--   -> 4b9df73e-bc38-49ef-840e-75fdb0b29c29

-- The client's 'to_do' column:
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do' LIMIT 1;

INSERT INTO public.tasks (client_id, title, description, status, priority, column_id, assignee_user_id)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger created a draft for review. Duda: <preview/editor url>. Topic: <topic_title>. Keyword: <focus_keyword>. Sources: <source URLs>.',
  'to_do',
  'normal',
  '<task_columns.id from above>',
  '4b9df73e-bc38-49ef-840e-75fdb0b29c29'
)
RETURNING id;  -- keep as <task_id>

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '4b9df73e-bc38-49ef-840e-75fdb0b29c29');
```

`tasks.status` is an enum (`task_status`): `to_do` | `in_progress` | `done`. `tasks.priority` is an enum
(`task_priority`): `low` | `normal` | `high` | `urgent`. Both `assignee_user_id` (on `tasks`) and a row in
`task_assignees` are populated — the app reads assignment from `task_assignees`, `assignee_user_id` is a
denormalized convenience column, so set both.

If a client has no `'to_do'` `task_columns` row, don't invent one — insert the task with `column_id =
NULL` and note the missing column setup in the end-of-run report as a client-config gap.

## Write-order summary (per client, once a draft is created)

1. `blog_auto_runs` insert (open, at the top of this client's processing — before research).
2. `blog_topics` update/insert (mark the calendar topic `in_progress`, or generate + insert one).
3. Duda `create_blog_post` (outside SpearlanceOS — see routine.md Publish step).
4. `blog_posts` insert (status `'draft'`, carrying the Duda ids and the QA verdict).
5. `blog_topics` update — `status = 'published'`, set `blog_post_id`.
6. `tasks` + `task_assignees` insert (assign to Mila).
7. `blog_auto_runs` update (close — `status = 'completed'`).

If the branch is stopped instead (no draft): skip 2–6 (or stop partway if a topic was already marked
`in_progress` — leave it there, don't revert it, the next run's Step 4a query already excludes
`in_progress` rows), and close `blog_auto_runs` with `status = 'failed'` + `error_log` explaining why.
