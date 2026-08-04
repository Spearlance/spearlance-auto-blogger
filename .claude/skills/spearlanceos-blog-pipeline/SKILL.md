---
name: spearlanceos-blog-pipeline
description: >
  Exact SpearlanceOS Supabase write-back sequence for the Auto Blogger routine — column names,
  status values, and copy-pasteable SQL for blog_auto_runs, blog_topics, blog_posts, tasks, and
  task_assignees. Use this any time the routine opens or closes a run log, marks a topic used,
  records a drafted article, or creates Mila's review task. Project id chikljxwgiskyjsnjelf.
  Schema verified live via information_schema on 2026-08-04; re-verify if a write errors on an
  unknown column, since this file can drift from the live schema.
---

# SpearlanceOS blog pipeline — write-back reference

The routine reads most of SpearlanceOS and writes to exactly five places: `blog_topics`,
`blog_posts`, `blog_auto_runs`, `tasks`, `task_assignees`. Nothing else. Never call SpearlanceOS's
`blog-*` edge functions.

Project id: `chikljxwgiskyjsnjelf`. All SQL below runs via the Supabase MCP `execute_sql`.

## 1. Open the run log — first thing, per client, before any research

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type`: `'scheduled'` for the normal hourly cron tick. Use `'manual'` only when invoked
  ad hoc for a single `client_id` (Step 0b test/dry-run case).
- Keep the returned `id` as `<run_id>` — every other write this client-turn threads through it.

`blog_auto_runs` columns: `id, client_id, triggered_at (default now()), trigger_type, status
(default 'running'), topics_generated (default 0), articles_generated (default 0),
articles_passed_gate (default 0), articles_flagged (default 0), research_summary jsonb,
completed_at, error_log, created_at`.

## 2. Topic selection write-back (Step 4)

**4a — using an existing calendar topic:** mark it claimed so nothing else picks it up mid-run.

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**4b — generating a new topic:**

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, priority, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<one-line summary>', ARRAY['kw1','kw2'], '<category>',
   'medium', true, 'in_progress', current_date)
RETURNING id;
```

`blog_topics` columns actually present: `id, client_id, topic_title, description, keywords[],
avatar_id, priority (text, default 'medium'), status (text, default 'idea'; observed values:
idea / in_progress / published), blog_post_id, ai_generated, created_at, strategy_batch_id,
summary, suggested_publish_date, category`. Both `description` and `summary` columns exist —
routine.md's Step 4 SQL uses `summary`; write there. `priority` is free-text, not an enum
(routine.md's `ORDER BY priority DESC` sorts lexically — `'high' > 'medium' > 'low'` happens to
sort correctly ASCII-wise, but don't rely on numeric ordering).

## 3. After drafting — QA passed, Duda draft created

Do these three inserts/updates as one sequence, in this order, once you have the Duda post's id:

**a. Close the topic loop** (stops the next tick reselecting it — "published" here means "turned
into a post," not "live on the client's site"; that's the existing table convention):

```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts_row_id>'
WHERE id = '<topic_id>';
```

**b. Insert the `blog_posts` row** — this is Mila's actual draft/schedule/publish queue record:

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status, duda_page_id,
   duda_publish_url, topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta description>', '<full HTML content>',
   '<short excerpt>', '<primary keyword>', '<category>', ARRAY['kw1','kw2'],
   '<thumbnail/main image url>', '<alt text>', 'draft', '<duda post id>',
   '<duda preview/editor url>', '<topic_id>', '<run_id>',
   '{"verdict":"SHIP","word_count":1650,"h2_count":6,"internal_links":8,"sourced_stats":4}'::jsonb)
RETURNING id;
```

- `status` is **always** `'draft'` here — never `'scheduled'` or `'published'`; only Mila changes
  that, from her own review queue.
- `slug` and `title` are `NOT NULL`. `quality_scores` is free-form jsonb — put the doctrine QA
  verdict and whatever counts justified SHIP; whatever shape is useful for Mila/Garrett to audit
  later without re-opening the draft.
- Table also has `seo_score, readability_score, avatar_id, body_images, scheduled_for,
  published_at, ai_model (default 'claude-sonnet-4-5'), generation_prompt, generation_metadata,
  created_by, last_edited_by, scheduled_publish_date, images_generated, rejection_reason,
  revision_count` — optional, fill in if you have the data, skip otherwise.

**c. Create Mila's review task:**

```sql
-- look up her user id once per run (stable — safe to cache across clients in the same tick)
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
-- → 4b9df73e-bc38-49ef-840e-75fdb0b29c29 as of 2026-08-04; re-look-up if this ever 0-rows.

-- the client's default "To Do" column
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';

INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger drafted this post — <one-line what/why, doctrine verdict, Duda preview link>.',
  'to_do',
  'normal',
  '4b9df73e-bc38-49ef-840e-75fdb0b29c29',
  '<to_do column_id from above>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id from above>', '4b9df73e-bc38-49ef-840e-75fdb0b29c29');
```

`tasks.status` and `tasks.priority` are enum types (`task_status`, `task_priority`) — use the
lowercase values shown (`'to_do'`, `'normal'`); don't guess casing. `task_columns` is per-client;
every client has a default `'to_do'` key row (`is_default = true`), but always look it up by
`client_id` rather than assuming the same `column_id` across clients.

Only do 3b/3c when a draft was actually created. A client whose branch was stopped (QA fail, no
usable topic, no fuel) gets neither a `blog_posts` row nor a task.

## 4. Close the run log — last thing, per client, always

Whether or not a draft was created, close the same row opened in step 1:

**Draft created:**

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = <0 or 1>, articles_generated = 1, articles_passed_gate = 1,
    articles_flagged = <0 or 1>, research_summary = '{"sources":["url1","url2"]}'::jsonb
WHERE id = '<run_id>';
```

**Branch stopped instead (QA fail / no fuel / no strategy row / doctrine halt / escalation):**

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, specific enough to act on>'
WHERE id = '<run_id>';
```

Observed `blog_auto_runs.status` values in practice: `running` (open, default) → `completed` or
`failed` (closed). There is no `'skipped'` value in use — a client that was due but didn't clear
the bar closes as `'failed'` with `error_log` explaining why (that's the convention already in the
table; don't invent a new status value).

## Known gaps as of 2026-08-04 (fix or flag, don't silently work around)

- No per-client daily/lifetime draft cap: the due-check (routine Step 0c) only looks at
  `selected_days`, not "already posted today," so a due client with no fresh angle left gets
  re-evaluated every hour it's due, all day. Several clients (Sunshine State, Invictus
  Snowfighters) have saturated their topic space this way. This is a scheduling design decision
  for Garrett, not something to patch ad hoc in a single run.
- `clients.website_url` is stale for at least one client (Invictus Snowfighters: row shows
  `invictusnorthwestgroup.com`, live/Duda-canonical is `invictussnowfighters.com`) — worth a
  one-time correction in SpearlanceOS.
