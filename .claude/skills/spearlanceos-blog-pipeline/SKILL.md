# SpearlanceOS Blog Pipeline (write-back schema)

Exact column names, status values, and copy-pasteable SQL for the four tables the current
(draft-only, never-publish-live) Auto Blogger playbook writes to: `blog_topics`, `blog_posts`,
`blog_auto_runs`, and `tasks` + `task_assignees`. Schema below was pulled live from
`information_schema.columns` on project `chikljxwgiskyjsnjelf` — reread it if a write fails on an
unknown column, since this table may drift from the app side without this file being updated.

This file did not exist before 2026-08-26; multiple prior ticks logged writing to these tables by
re-deriving the schema from `information_schema` every single run. Keep this file current so that
stops.

## 1. Open the run log, per client, before any research

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Keep the returned `id` as `<run_id>` for the rest of this client's processing. Use
`trigger_type = 'manual'` only when invoked ad hoc for a single `client_id`.

Columns: `id`, `client_id`, `triggered_at` (default now()), `trigger_type`, `status` (default
`'running'`), `topics_generated` (int, default 0), `articles_generated` (int, default 0),
`articles_passed_gate` (int, default 0), `articles_flagged` (int, default 0), `research_summary`
(jsonb, optional), `completed_at`, `error_log`, `created_at`.

## 2. Mark the topic used/created

```sql
UPDATE public.blog_topics
SET status = 'published'   -- means "turned into a draft post", not "live" — existing convention
WHERE id = '<topic_id>';
```

`blog_topics` columns of note: `id`, `client_id`, `topic_title`, `description`, `summary`,
`keywords` (text[]), `category`, `priority` (text, default `'medium'`), `status` (text, default
`'idea'`), `blog_post_id` (uuid, nullable — set NULL; the actual link lives on `blog_posts.topic_id`
below), `ai_generated` (bool), `suggested_publish_date`, `strategy_batch_id`, `avatar_id`,
`created_at`. Step 4a's `WHERE ... status NOT IN ('published','in_progress')` filter is what this
write satisfies.

## 3. Insert the drafted article

```sql
INSERT INTO public.blog_posts (
  client_id, topic_id, auto_run_id, title, slug, meta_description, content, excerpt,
  focus_keyword, target_keywords, topic_category, featured_image_url, featured_image_alt,
  status, quality_scores, ai_model
) VALUES (
  '<client_id>', '<topic_id>', '<run_id>', '<title>', '<slug>', '<meta_description>', '<html_content>',
  '<excerpt>', '<focus_keyword>', ARRAY['<kw1>','<kw2>'], '<category>', '<duda_image_url>',
  '<image_alt_text>', 'draft', '<quality_scores_jsonb>', 'claude-sonnet-4-5'
)
RETURNING id;
```

- `status` is **always `'draft'`** — never `'scheduled'` or `'published'` from this routine; that
  transition is Mila's call.
- `quality_scores` (jsonb) should carry the doctrine QA verdict (word count, H2 count, link count,
  sourcing check, intent match) so the review UI can show it without re-reading the draft.
- `duda_page_id` / `duda_publish_url` — set these from the `create_blog_post` response so Mila's
  queue links straight to the Duda draft.
- A DB trigger (`trg_one_blog_draft_per_client_per_day` on `blog_posts`) blocks a second insert for
  the same `client_id` on the same calendar day. If you hit it, that means a draft already exists
  for this client today — treat it exactly like any other "already served" outcome (Step 0c/0d):
  do not retry, close the run log as below, and move to the next client.

## 4. Close the run log

```sql
-- success (draft created):
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = 1, articles_generated = 1, articles_passed_gate = 1
WHERE id = '<run_id>';

-- stopped instead (QA fail, no fuel, no strategy row, doctrine halt, escalation, already-served):
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why>'
WHERE id = '<run_id>';
```

Every run log opened in step 1 gets closed one way or the other before moving to the next client.
`error_log` is free text — the convention in this table's history is a dense paragraph naming what
was checked, what was found, and why the branch stopped; keep doing that, it's what makes the run
history auditable without a separate ledger.

## 5. Create Mila's review task (only when a draft was created)

Look up Mila's user id from her email, then find the client's `'to_do'` column:

```sql
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';  -- confirm exact address once; reuse the id

SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

```sql
INSERT INTO public.tasks (client_id, title, status, column_id, assignee_user_id)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'to_do',
  '<to_do_column_id>',
  '<mila_user_id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_user_id>');
```

`tasks.status` and `task_columns.mapped_status` are the `task_status` enum — `'to_do'` is correct
for a fresh review task. `creator_user_id` may be left NULL (existing rows created by the routine do
this). Title convention seen in production: `"Review blog draft: <post title>"` or `"Review Auto
Blogger draft: <post title>"` — either is fine, keep it identifiable as this routine's output.

## Nothing else

Write nothing to any other SpearlanceOS table, and never call its `blog-*` edge functions. If a
write here fails on a missing/renamed column, re-pull `information_schema.columns` for that table
before guessing, and update this file so the next tick doesn't repeat the lookup.
