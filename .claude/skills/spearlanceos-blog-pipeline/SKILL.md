---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase table/column reference and copy-pasteable SQL for the Auto Blogger's write-back steps (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read this before writing to any of those tables so column names and status/enum values are never guessed.
---

# SpearlanceOS blog pipeline write-back reference

Vendored from `information_schema` on project `chikljxwgiskyjsnjelf` (production) on 2026-09-04.
Re-introspect (`information_schema.columns` filtered to these table names, plus `pg_enum`) if a write
fails on an unknown column — schema may have drifted since this was last vendored — and update this
file so the fix persists for future ticks instead of being silently re-derived every hour.

Production project ref: **`chikljxwgiskyjsnjelf`**. Per the `supabase-prod-guard` skill, this routine's
writes to exactly the five tables below are pre-authorized by `CLAUDE.md`/`routine.md` as the routine's
normal, scheduled, unattended operation — no fresh confirmation needed per tick. Every other table is
read-only; never write anywhere else and never call SpearlanceOS's `blog-*` edge functions.

**Known live gotcha:** a `BEFORE INSERT` trigger `trg_one_blog_draft_per_client_per_day` on
`public.blog_posts` caps each client at one row per calendar day — a second insert for the same
`client_id` on the same date is rejected. Before attempting Draft → Publish for a due client, check
whether it already has a `blog_posts` row with `created_at::date = current_date`; if so, skip straight
to logging the decline in `blog_auto_runs` (no research needed, the insert would fail regardless of
what's found). `routine.md` Step 0c's due-check does not itself account for this trigger, so don't
expect "due" to imply "postable" — it only tells you whether to check.

## 1. Open the run log (per due client, before any research)

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Use `trigger_type = 'manual'` only for an ad hoc single-`client_id` invocation. Keep the returned `id`
as `<run_id>` — every other write below for this client references it.

## 2. `blog_topics` — mark the used/generated topic done

Only when a draft was actually created (Publish step). If a calendar topic was used (Step 4a):

```sql
UPDATE public.blog_topics
SET status = 'published', blog_post_id = '<new_blog_posts_id>'
WHERE id = '<topic_id>';
```

If a topic was generated (Step 4b), it was already inserted with `status = 'in_progress'`; run the same
UPDATE once the post exists. Columns: `id, client_id, topic_title, description, summary, keywords[],
avatar_id, priority (text, default 'medium'), status (text, default 'idea'; values used: 'idea',
'in_progress', 'published'), blog_post_id, ai_generated (bool), strategy_batch_id, suggested_publish_date
(date), category, created_at`. Note the column is `description`, not `summary`, for the long-form field
— `summary` also exists (shorter); routine.md's SELECT/INSERT examples using `summary` are fine, just be
aware both columns exist and mean slightly different things (`summary` = short, `description` = fuller).

## 3. `blog_posts` — insert the drafted article

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt, focus_keyword,
  topic_category, target_keywords, featured_image_url, featured_image_alt,
  status, topic_id, auto_run_id, duda_page_id, duda_publish_url, quality_scores
) VALUES (
  '<client_id>', '<title>', '<url-slug>', '<meta description>', '<html or plain content actually drafted>',
  '<short excerpt>', '<primary target keyword>',
  '<category>', ARRAY['<kw1>','<kw2>'], '<thumbnail/main image url>', '<alt text>',
  'draft', '<topic_id or NULL>', '<run_id>', '<duda blog post id>', '<duda preview/editor url>',
  '<jsonb QA verdict, e.g. {"doctrine":"SHIP","sourced_claims":true,"word_count":1650}>'
)
RETURNING id;
```

Always `status = 'draft'` — never `'scheduled'` or `'published'`; that's Mila's call. `duda_publish_url`
should be the actual post preview/editor URL Duda returns from `create_blog_post`, not a bare site
dashboard link (a prior tick shipped a dashboard link by mistake for one post — fix forward if you see
that pattern, don't repeat it). Other columns (`avatar_id`, `seo_score`, `readability_score`,
`generation_metadata`, `ai_model`, etc.) are optional; fill `ai_model` if easy, otherwise the column
default covers it.

## 4. Close the run log

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', articles_generated = 1, articles_passed_gate = 1,
    research_summary = '<jsonb of sources/keywords used>', completed_at = now()
WHERE id = '<run_id>';
```

If the branch was stopped instead (QA fail, no fuel, no strategy row, doctrine halt, escalation, or
blocked by `trg_one_blog_draft_per_client_per_day`):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', error_log = '<why, specific and useful to the next tick>', completed_at = now()
WHERE id = '<run_id>';
```

Every opened run row gets closed one way or the other before moving to the next client. `topics_generated`
increments only when Step 4b actually inserted a new `blog_topics` row this run.

## 5. `tasks` + `task_assignees` — only when a draft was created

Look up Mila's `task_columns` row for this client's `'to_do'` column, and her profile id:

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do' LIMIT 1;
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Then:

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, column_id)
VALUES (
  '<client_id>', 'Review blog draft: <post title>',
  'Auto Blogger created a draft for review. Duda: <duda_publish_url>. blog_posts id: <id>.',
  'to_do', 'normal', '<to_do task_columns id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<new_task_id>', '<mila_profile_id>');
```

`task_status` enum: `to_do`, `in_progress`, `done`. `task_priority` enum: `low`, `normal`, `high`,
`urgent`. Mila's profile id (re-verify if this ever 404s — email is the stable lookup):
`4b9df73e-bc38-49ef-840e-75fdb0b29c29` (`mila@spearlance.com`).

## Reference: full column list (as of 2026-09-04)

**blog_auto_runs**: id, client_id, triggered_at, trigger_type, status (default 'running'; values used:
running/completed/failed), topics_generated, articles_generated, articles_passed_gate,
articles_flagged, research_summary (jsonb), completed_at, error_log, created_at.

**blog_posts**: id, client_id, title, slug, meta_description, content, excerpt, focus_keyword,
seo_score, readability_score, avatar_id, topic_category, target_keywords (array), featured_image_url,
featured_image_alt, body_images (jsonb), status (default 'draft'), scheduled_for, published_at,
duda_page_id, duda_publish_url, ai_model (default 'claude-sonnet-4-5'), generation_prompt,
generation_metadata (jsonb), created_by, created_at, updated_at, last_edited_by, topic_id,
scheduled_publish_date, images_generated (bool), auto_run_id, quality_scores (jsonb), rejection_reason,
revision_count.

**blog_topics**: id, client_id, topic_title, description, keywords (array), avatar_id, priority (text,
default 'medium'), status (text, default 'idea'), blog_post_id, ai_generated (bool), created_at,
strategy_batch_id, summary, suggested_publish_date, category.

**tasks**: id, client_id, title, description, status (enum task_status, default 'to_do'), priority
(enum task_priority, default 'normal'), assignee_user_id, creator_user_id, due_date,
related_asset_ids (array), related_meeting_ids (array), activity_log, created_at, updated_at,
linked_channel_id, parent_task_id, subtask_order, color, is_recurring, recurrence_pattern (jsonb),
parent_recurring_task_id, next_occurrence_date, is_recurring_instance, original_due_date, column_id,
started_at, completed_at, external_source, external_ref.

**task_assignees**: id, task_id, user_id, created_at.

**task_columns**: id, client_id, name, key, color, display_order, is_default, created_at, updated_at,
mapped_status (enum task_status).
