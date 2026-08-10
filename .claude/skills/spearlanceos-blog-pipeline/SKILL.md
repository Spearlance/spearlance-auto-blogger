---
name: spearlanceos-blog-pipeline
description: Exact table/column names, status values, and copy-pasteable SQL for the Auto Blogger's SpearlanceOS write-back — blog_auto_runs (run log), blog_posts (draft/review queue), blog_topics (topic state), and tasks/task_assignees (Mila's review task). Read by routine.md at the "open the run log" step and again at Publish. Referenced throughout routine.md; this file did not exist in the repo until 2026-08-10 (schema derived via information_schema introspection because the file had been promised twice before — 2026-08-07 14:01 UTC and 23:40 UTC runs — but never committed).
---

# SpearlanceOS Blog Pipeline — write-back reference

Project id: `chikljxwgiskyjsnjelf`. The routine reads almost everything and writes to exactly five
tables: `blog_topics`, `blog_posts`, `blog_auto_runs`, `tasks`, `task_assignees`. Never write anywhere
else; never call SpearlanceOS's `blog-*` edge functions.

## 1. Open the run log — `blog_auto_runs` (per client, before any research)

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type`: `'scheduled'` for the normal hourly tick, `'manual'` only when invoked ad hoc for a
  single `client_id` (Step 0b test/dry-run path).
- Keep the returned `id` as `<run_id>` — every downstream `blog_posts` row for this client this tick
  gets `auto_run_id = <run_id>`.
- Columns available to fill in at close-out: `topics_generated`, `articles_generated`,
  `articles_passed_gate`, `articles_flagged` (all `integer`, default `0`), `research_summary` (`jsonb`,
  optional), `error_log` (`text`), `completed_at` (`timestamptz`).

## 2. Close the run log (always — success or failure, exactly once)

**On a successful draft:**
```sql
UPDATE public.blog_auto_runs
SET status = 'completed', articles_generated = 1, articles_passed_gate = 1,
    completed_at = now()
WHERE id = '<run_id>';
```

**On a stopped branch** (QA fail, no fuel, no strategy row, doctrine halt, escalation, or a declined
tick like a saturation/duplication guard):
```sql
UPDATE public.blog_auto_runs
SET status = 'failed', error_log = '<why, specific enough to act on>', completed_at = now()
WHERE id = '<run_id>';
```

`status` has no DB-enforced enum — used values in practice: `running`, `completed`, `failed`. Use
`failed` for "did not produce a draft," not just hard errors — that is the convention already in use.

## 3. `blog_topics` — mark the topic used

After a topic is selected (4a) or generated (4b) and used in a draft, whether or not the draft ships:

```sql
-- 4a: using an existing calendar topic
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';

-- 4b: generating a new one
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>',
        true, 'in_progress', current_date)
RETURNING id;

-- After the Duda draft is created and blog_posts row inserted:
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

`blog_topics.status` values in use: `idea` (default), `in_progress`, `published` (= "turned into a
draft," not "live" — the table's existing convention, do not read it as a publish signal).
`blog_post_id` is nullable `uuid`; only set once the `blog_posts` row exists.

## 4. `blog_posts` — the actual review/schedule record Mila works from

Insert **only when a Duda draft was actually created** (i.e., QA passed). Never insert a row for a
branch that was stopped before drafting — `blog_auto_runs.error_log` is where stopped branches get
recorded, not `blog_posts`.

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status, topic_id, auto_run_id,
   quality_scores, duda_page_id, duda_publish_url)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta_description>', '<full HTML/markdown body>',
   '<excerpt>', '<primary keyword>', '<category>', ARRAY['<kw1>','<kw2>'],
   '<image_url>', '<alt text>', 'draft', '<topic_id>', '<run_id>',
   '{"doctrine_verdict":"SHIP","word_count":1650,"h2_count":6,"internal_links":7,"sources_cited":4}'::jsonb,
   '<duda post id from create_blog_post response>', '<duda preview/editor url>')
RETURNING id;
```

- `status`: **always `'draft'`**. Never insert `'scheduled'` or `'published'` — that transition is
  Mila's call, made in the SpearlanceOS UI, not the routine's.
- `quality_scores` is free-form `jsonb` — put the doctrine-compliance verdict and the metrics that
  informed it (word count, H2 count, internal link count, sources cited) so Mila can see the QA
  reasoning without re-deriving it.
- `topic_id` and `auto_run_id` are both nullable `uuid` FKs — always set both when available.

## 5. `tasks` + `task_assignees` — notify Mila

Only when a draft was created (step 4 above ran). Look up Mila's `profiles.id` by email, and the
client's `'to_do'` `task_columns.id`:

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';

INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES ('<client_id>', 'Review blog draft: <post title>',
        'Auto Blogger created a draft for review. Duda preview: <preview_url>. Topic: <topic_title>. Doctrine verdict: SHIP.',
        'to_do', 'normal', '<mila_profile_id>', '<to_do_column_id>')
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_profile_id>');
```

- `tasks.status` is the `task_status` enum: `to_do | in_progress | done` (default `to_do` — leave it).
- `tasks.priority` is the `task_priority` enum: `low | normal | high | urgent` (default `normal`).
- `task_columns` is per-client; every client observed so far has the three defaults (`to_do`,
  `in_progress`, `done`) with `is_default = true`. Look it up per client rather than assuming a fixed
  UUID — column ids are not shared across clients.
- `tasks.assignee_user_id` AND a `task_assignees` row are both needed — the board reads assignment from
  `task_assignees`; `assignee_user_id` is the legacy single-assignee column. Write both.

## Gotchas learned the hard way

- **Mila's email** is `mila@spearlance.com`, looked up in `public.profiles` (not `auth.users` — there is
  no direct `users` table in the public schema; `profiles` is the app-facing identity table).
- **`blog_content_strategy`** may have both a client-specific row and an `is_global = true` fallback row;
  `ORDER BY is_global ASC LIMIT 1` in Step 0c already prefers the client-specific one when it exists.
- **`blog_ai_preferences`** frequently has no row at all for a client — treat a zero-row result as "no
  preferences configured," not an error.
- Do not gate topic/draft creation on Duda's `list_blog_posts` for a same-day duplicate check — for a
  client with a long posting history that call can return 100+ posts (proxy this via `blog_auto_runs`
  instead, which the routine already writes to and can filter by `triggered_at::date = current_date`
  cheaply).
