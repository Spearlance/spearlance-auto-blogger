---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase write-back mechanics for the Auto Blogger — column names, status values, and copy-pasteable SQL for blog_topics, blog_posts, blog_auto_runs, tasks, and task_assignees. Use whenever the Auto Blogger routine needs to open/close a run log, insert a drafted article, or file Mila's review task.
---

# SpearlanceOS Blog Pipeline — write-back reference

This documents the **actual, verified** schema (queried directly via the Supabase MCP
`list_tables`/`execute_sql`/`pg_proc`, project `chikljxwgiskyjsnjelf`) for the five tables the
draft-only, Mila-review Auto Blogger writes to. It exists because the routine references this file
before the file itself existed — provenance below.

## Provenance

Created 2026-08-24 after a scheduled run found the file referenced (by the stored scheduled-task
prompt) but absent from the repo. A prior same-day run's `blog_auto_runs.error_log` claimed to have
created this file at a specific commit — that commit and branch do **not** exist in git history
(verified via `git log --all` / `git branch -a`); that claim was false. This file is the first real
version, built from direct schema inspection, not from that claim.

**Known open issue, not fixed here:** the repo's `routine.md` / `CLAUDE.md` still describe an older,
single-table (`blog_topics`-only, publish-live-per-client) architecture. The stored scheduled-task
prompt that actually fires each tick already uses the newer 5-table, always-draft, Mila-review model
documented below. Editing `routine.md` in this repo has **no effect** on what the scheduler sends —
that prompt lives in the scheduling config, not this repo — so reconciling the two needs Garrett to
update the schedule's stored prompt (or explicitly confirm this file's model is the target and the
repo docs should be rewritten to match).

## Hard DB guardrail (not documented elsewhere)

```sql
CREATE TRIGGER trg_one_blog_draft_per_client_per_day
  BEFORE INSERT ON public.blog_posts
  FOR EACH ROW EXECUTE FUNCTION guard_one_blog_draft_per_client_per_day();
```

The function raises a hard exception (`ERRCODE = 'P0001'`) if a `blog_posts` row for the same
`client_id` was already `created_at::date = now()::date`. **Before starting any research for a due
client, check for an existing same-day `blog_posts` row** — if one exists, the insert will fail
regardless of how good the draft is, and Duda will still accept `create_blog_post`, producing an
orphaned, untracked draft. Check first:

```sql
SELECT id, duda_page_id, created_at FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at::date = current_date;
```

If a row comes back, skip research for that client this tick (log why in `blog_auto_runs`) rather than
spend Tavily/DataForSEO calls on a draft that cannot be written back.

## 1. Open the run log — `blog_auto_runs`

Insert **before** research begins, one row per client attempted this tick:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

`trigger_type`: `'scheduled'` for the normal hourly cron; `'manual'` only when invoked ad hoc for a
single `client_id`. Keep the returned `id` as `<run_id>`.

## 2. If a topic was generated — `blog_topics`

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, priority, ai_generated, status, suggested_publish_date)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', 'medium', true, 'in_progress', current_date)
RETURNING id;
```

If reusing an existing calendar topic instead, mark it in progress first:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

## 3. After the Duda draft is created — `blog_posts`

Always `status = 'draft'` — never `'scheduled'` or `'published'`; that is Mila's call.

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, target_keywords,
   topic_category, featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
   topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta_description>', '<html_content>', '<focus_keyword>',
   ARRAY['kw1','kw2'], '<category>', '<image_url>', '<image_alt>', 'draft', '<duda_post_id>',
   '<duda_preview_url>', '<topic_id>', '<run_id>',
   '{"verdict":"SHIP","word_count":1650,"h2_count":6,"internal_links":7}'::jsonb)
RETURNING id;
```

Then close the topic:

```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_post_id>' WHERE id = '<topic_id>';
```

(`status='published'` on `blog_topics` means "turned into a post," matching the table's existing
convention — not that it went live on the site.)

## 4. Close the run log — `blog_auto_runs`

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = 1, articles_generated = 1, articles_passed_gate = 1
WHERE id = '<run_id>';
```

On a stopped branch (QA fail, no fuel, no strategy row, doctrine halt, escalation, or blocked by the
guardrail above):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, specifically>'
WHERE id = '<run_id>';
```

`blog_auto_runs` columns actually available for `error_log`/counts: `topics_generated`,
`articles_generated`, `articles_passed_gate`, `articles_flagged`, `research_summary` (jsonb),
`error_log` (text). There is no `article_count` column — don't guess it.

## 5. File Mila's review task — `tasks` + `task_assignees`

Only when a `blog_posts` row was actually created. Look up Mila's profile and the client's `to_do`
column:

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';  -- confirm exact address first
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

`profiles` has `name`/`email` (no `full_name`). Then:

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, column_id)
VALUES ('<client_id>', 'Review blog draft: <post title>',
        'Auto Blogger drafted this post. Duda preview: <preview_url>. Review and schedule/publish in Duda.',
        'to_do', 'normal', '<to_do_column_id>')
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_profile_id>');
```

`task_status` enum: `to_do`, `in_progress`, `done`. `task_priority` enum: `low`, `normal`, `high`,
`urgent`.

## Write scope reminder

The routine writes **only**: `blog_topics`, `blog_posts`, `blog_auto_runs`, `tasks`,
`task_assignees`. Never touch anything else in SpearlanceOS; never call its `blog-*` edge functions.
