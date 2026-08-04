---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the five SpearlanceOS Supabase tables the Auto Blogger writes to (blog_auto_runs, blog_topics, blog_posts, tasks, task_assignees). Read before the "Open the run log" and "Publish" write-back steps in routine.md. Project id chikljxwgiskyjsnjelf.
---

# SpearlanceOS Blog Pipeline — write-back reference

Schema below was pulled directly from `information_schema.columns` on project `chikljxwgiskyjsnjelf`
(verified 2026-08-04) rather than assumed — if a write in this file ever errors on a missing/renamed
column, re-derive from `information_schema.columns` rather than guessing, and update this file.

All writes below are the routine's own pre-authorized, designed behavior (see `CLAUDE.md` /
`routine.md`) — they don't need a fresh ask each run. Anything outside these five tables does.

## 1. Open the run log (before research, per due client)

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Use `trigger_type = 'manual'` instead of `'scheduled'` only when invoked ad hoc for a single
`client_id` (Step 0b test/dry-run case). Keep the returned `id` as `<run_id>` — you close this same
row once, at the end of this client's processing (step 6, below), whether or not a draft resulted.

`blog_auto_runs` columns: `id`, `client_id`, `triggered_at` (default now), `trigger_type` (**required**,
no default), `status` (default `'running'`), `topics_generated`, `articles_generated`,
`articles_passed_gate`, `articles_flagged` (all default `0`), `research_summary` (jsonb, nullable),
`completed_at` (nullable), `error_log` (nullable), `created_at`.

## 2. blog_topics — using an existing calendar topic (Step 4a)

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

## 3. blog_topics — generating a new topic (Step 4b)

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<topic_title>', '<summary>', ARRAY['kw1','kw2'], '<category>',
   true, 'in_progress', current_date)
RETURNING id;
```

`blog_topics` columns: `id`, `client_id`, `topic_title`, `description`, `keywords` (text[]),
`avatar_id`, `priority` (default `'medium'`), `status` (default `'idea'`), `blog_post_id`,
`ai_generated` (default false), `created_at`, `strategy_batch_id`, `summary`,
`suggested_publish_date`, `category`. Note both `description` and `summary` exist as separate
columns — routine.md's Step 4b template uses `summary`.

## 4. blog_posts — the draft record (only when a draft was actually created in Duda)

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status, topic_id, auto_run_id,
   quality_scores, duda_page_id, duda_publish_url)
VALUES
  ('<client_id>', '<title>', '<url_slug>', '<meta_description>', '<content_html_or_summary>',
   '<excerpt>', '<focus_keyword>', '<category>', ARRAY['kw1','kw2'], '<featured_image_url>',
   '<alt_text>', 'draft', '<topic_id>', '<run_id>', '<doctrine_qa_verdict_json>'::jsonb,
   '<duda_post_id>', '<duda_editor_or_preview_url>')
RETURNING id;
```

`status` is **always** `'draft'` — never insert `'scheduled'` or `'published'`; that's Mila's call.
`quality_scores` is jsonb — store the §1–§14.3 doctrine-compliance verdict block from the QA step.

Then close the topic used (mirrors "turned into a post", not "live" — existing table convention):

```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

## 5. tasks + task_assignees — Mila's review task (only when a draft was created)

```sql
-- Mila's user id (auth.users, not a public.profiles/users table)
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';

-- the client's 'to_do' column (task_columns is scoped per client_id)
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';

INSERT INTO public.tasks
  (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES
  ('<client_id>', 'Review blog draft: <post_title>',
   '<Duda preview/editor link + doctrine verdict summary + FAQ-schema-needs-manual-add note>',
   'to_do', 'normal', '<mila_user_id>', '<to_do_column_id>')
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_user_id>');
```

`tasks.status` and `tasks.priority` are Postgres enums (`task_status`, `task_priority`); `'to_do'` /
`'normal'` are valid members. Set both `tasks.assignee_user_id` directly and a matching
`task_assignees` row — the app reads assignment from `task_assignees`; `assignee_user_id` mirrors it.

## 6. blog_auto_runs — close out (always, exactly once per client per tick)

Success:

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = <n>, articles_generated = <n>, articles_passed_gate = <n>,
    articles_flagged = <n>, research_summary = '<jsonb_of_sources_and_notes>'::jsonb
WHERE id = '<run_id>';
```

Declined / failed (QA fail, no fuel, no strategy row, doctrine halt, escalation, nothing usable to
draft today):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, specific enough to act on>'
WHERE id = '<run_id>';
```

## Gotchas

- No `public.users`/`public.profiles` table for staff — Mila (and other agency staff) live in
  `auth.users`; look up by `email`.
- `task_columns` and its `'to_do'` key are **per client_id**, not global — always scope the lookup.
- `list_tables` on this project returns a very large payload (200k+ chars) and will exceed tool output
  limits; query `information_schema.columns` filtered to the specific table names you need instead.
- `blog_posts.status` must stay `'draft'` — the routine never publishes live and never pre-schedules.
