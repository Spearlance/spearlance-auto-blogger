---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase schema and copy-pasteable SQL for the Auto Blogger's write-back tables (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Use whenever routine.md's Publish or "Open the run log" steps call for a write to SpearlanceOS, so column names, status enums, and lookup keys are correct on the first try.
---

# SpearlanceOS Blog Pipeline — write-back reference

Project id: `chikljxwgiskyjsnjelf`. The routine reads freely from SpearlanceOS but writes **only**
to the five objects below. Never touch anything else, never call SpearlanceOS's `blog-*` edge
functions. Schema confirmed live via `information_schema.columns` — if a run finds it's drifted,
re-confirm with that query rather than trusting this file blindly.

## 1. `blog_auto_runs` — one row per client per tick

Opened before research/drafting starts, closed once at the end of that client's processing.

```sql
-- Open (before any research/drafting for this client)
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')  -- trigger_type = 'manual' only for an ad hoc single-client invocation
RETURNING id;  -- keep as <run_id>

-- Close: success path
UPDATE public.blog_auto_runs
SET status = 'completed',
    topics_generated = <0 or 1>,       -- 1 if Step 4b generated a new topic this run
    articles_generated = 1,
    articles_passed_gate = 1,
    research_summary = '{"sources": [...], "keyword": "...", "intent": "..."}'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';

-- Close: stopped branch (QA fail, no fuel, no strategy row, doctrine halt, saturation, escalation)
UPDATE public.blog_auto_runs
SET status = 'failed',
    completed_at = now(),
    error_log = '<why this branch stopped, specific enough for the end-of-run report>'
WHERE id = '<run_id>';
```

Columns: `id, client_id, triggered_at (default now()), trigger_type, status (default 'running'),
topics_generated, articles_generated, articles_passed_gate, articles_flagged, research_summary
(jsonb), completed_at, error_log, created_at`. No `updated_at`; you update the same row once via
`WHERE id = '<run_id>'`, not by inserting a second row.

If a run is a lightweight re-verification of a same-day saturation finding (not a full grind),
still open+close a row — set `triggered_at`/`completed_at` close together and say so in
`error_log`, don't skip logging just because the outcome is "nothing changed."

## 2. `blog_topics` — the topic queue

Selecting an existing due topic (Step 4a):

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
-- ... after a draft is created ...
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>' WHERE id = '<topic_id>';
```

Generating a new topic (Step 4b):

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, priority, status, ai_generated, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<one-paragraph summary>', ARRAY['<kw1>','<kw2>'], '<category>',
   'medium', 'in_progress', true, current_date)
RETURNING id;  -- keep as <topic_id>, then mark 'published' + set blog_post_id once the draft exists
```

Columns: `id, client_id, topic_title, description, keywords (text[]), avatar_id, priority (text,
default 'medium'), status (text, default 'idea'), blog_post_id, ai_generated, created_at,
strategy_batch_id, summary, suggested_publish_date, category`. Note both `description` and
`summary` exist — `routine.md`'s Step 4a/4b select and write `summary`; leave `description` null
unless a value is already present. `status` is free text, not an enum — the routine's own
convention is `idea → in_progress → published` (`'published'` means "turned into a post," not
"live"). `priority` is free text too (`'low' | 'medium' | 'high'` by convention, not a DB enum).

## 3. `blog_posts` — Mila's draft/schedule/publish queue

One row per drafted article, always inserted as `status = 'draft'` — never `'scheduled'` or
`'published'`; that transition is Mila's call, made in her own tooling, not by the routine.

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status, topic_id, auto_run_id,
   quality_scores, duda_page_id, duda_publish_url)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta description>', '<full HTML/markdown body actually published>',
   '<short excerpt>', '<primary keyword>', '<category>', ARRAY['<kw1>','<kw2>'],
   '<thumbnail/main image URL used in Duda>', '<alt text>', 'draft', '<topic_id>', '<run_id>',
   '{"verdict": "SHIP", "doctrine_checks": {"S1_title":"PASS", "S6_content":"PASS", "S7_links":"PASS", "S8_schema":"PASS"}, "word_count": 1650, "internal_links": 7}'::jsonb,
   '<duda blog post id from create_blog_post response>', '<duda preview/editor URL>')
RETURNING id;
```

Columns include `ai_model` (defaults to `'claude-sonnet-4-5'` — leave as default unless you need
to record a different model), `generation_prompt`, `generation_metadata`, `scheduled_for`,
`scheduled_publish_date`, `published_at`, `images_generated`, `rejection_reason`,
`revision_count` — populate only if useful, none are required beyond what's shown above.

## 4. `tasks` + `task_assignees` — Mila's review task

Only when a draft was actually created. Look up her `'to_do'` column for this client, then create
the task and assign her.

```sql
-- Mila's user id (stable, but re-look-up if this ever comes back empty)
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';  -- 4b9df73e-bc38-49ef-840e-75fdb0b29c29

-- The client's 'to_do' column id
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';

INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger created a draft for review. Duda preview: <preview_url>. Keyword: <keyword>. Sources: <url1>, <url2>.',
  'to_do', 'normal', '4b9df73e-bc38-49ef-840e-75fdb0b29c29', '<column_id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '4b9df73e-bc38-49ef-840e-75fdb0b29c29');
```

`tasks.status` and `tasks.priority` are real Postgres enums: `task_status = {to_do, in_progress,
done}`, `task_priority = {low, normal, high, urgent}` — use `'to_do'` / `'normal'` unless there's a
specific reason otherwise. `tasks.assignee_user_id` is a convenience column; `task_assignees` is
the join table the UI actually reads for multi-assignee support — write both.

If a client has no `'to_do'` `task_columns` row (shouldn't happen for an onboarded client, but
don't assume), fall back to inserting the task with `column_id = NULL` and note it in the
end-of-run report rather than blocking the whole client's write-back on it.
