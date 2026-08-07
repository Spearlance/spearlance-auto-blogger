---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back to SpearlanceOS (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read this before writing back at the end of a client's run, per routine.md's Publish step and "Open the run log" step.
---

# SpearlanceOS blog write-back pipeline

Reference for the exact tables/columns the Auto Blogger writes to in SpearlanceOS Supabase
(project `chikljxwgiskyjsnjelf`). Schema was derived by introspecting the live production tables
via `information_schema.columns` and sampling recent rows (2026-08-07) — there was no prior
skill file for this, so several ticks that day each had to re-derive it independently. This file
closes that gap. If these tables' schema ever changes, re-derive with:

```sql
SELECT table_name, column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema='public' AND table_name IN
  ('blog_auto_runs','blog_posts','blog_topics','tasks','task_assignees','task_columns')
ORDER BY table_name, ordinal_position;
```

## Write order for one client's run

1. **Open** `blog_auto_runs` at the start of the client's processing (before research/drafting).
2. Do the work (Steps 1–4 of `routine.md`, Draft → QA).
3. If a draft is created in Duda: insert `blog_topics` (if a new topic was generated) or update
   the existing one, insert `blog_posts`, update `blog_topics.blog_post_id`, insert `tasks` +
   `task_assignees` for Mila.
4. **Close** the `blog_auto_runs` row opened in step 1 — always, whether or not a draft shipped.

## 1. `blog_auto_runs` — one row per client per tick

Open at the start:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;  -- keep as <run_id>
```

`trigger_type`: `'scheduled'` for the normal hourly cron tick; `'manual'` only when invoked ad hoc
for a single `client_id` (test/dry-run).

Close at the end (same row, one UPDATE):

```sql
UPDATE public.blog_auto_runs SET
  status = 'completed',           -- or 'failed' if the branch was stopped (QA fail, no fuel,
                                   -- no strategy row, doctrine halt, escalation)
  topics_generated = 1,           -- 0 if no topic was generated/used
  articles_generated = 1,         -- 0 if no draft was created
  articles_passed_gate = 1,       -- 1 if QA verdict = SHIP, 0 if FIX_FIRST/FAIL/not attempted
  articles_flagged = 0,           -- 1 if the draft shipped with a flag (e.g. FIX_FIRST note)
  research_summary = '{"topic": "...", "target_keyword": "...", "duda_draft_url": "...",
                        "duda_post_id": "...", "sources": ["..."], "word_count": 1693,
                        "verdict": "SHIP", "note": "..."}'::jsonb,
  error_log = NULL,               -- set to the reason when status = 'failed'; free text
  completed_at = now()
WHERE id = '<run_id>';
```

Columns: `id`, `client_id`, `triggered_at`, `trigger_type` (text), `status` (text, default
`'running'`), `topics_generated`/`articles_generated`/`articles_passed_gate`/`articles_flagged`
(int, default 0), `research_summary` (jsonb, freeform — put whatever the end-of-run report needs),
`completed_at`, `error_log` (text), `created_at`.

## 2. `blog_topics` — the topic used or generated

If generating a new topic (Step 4b), insert it (see `routine.md` Step 4b for the columns that
matter — `topic_title`, `summary`, `keywords`, `category`, `ai_generated=true`,
`status='in_progress'`, `suggested_publish_date=current_date`). If reusing an existing calendar
topic (Step 4a), just update its status.

Full column list: `id`, `client_id`, `topic_title` (**not** `title`), `description`, `keywords`
(text array), `avatar_id`, `priority` (text: `'low'`/`'medium'`/`'high'`, default `'medium'`),
`status` (text, default `'idea'`), `blog_post_id`, `ai_generated` (bool), `created_at`,
`strategy_batch_id`, `summary` (text — a separate column from `description`; the routine's own
queries reference `summary`, so populate that one), `suggested_publish_date` (date), `category`.

Status values observed in use: `'idea'` (default/unused), `'in_progress'` (claimed for this
run), `'published'` (turned into a post — this is the routine's done-marker, not "live").

After the `blog_posts` row exists, mark the topic done:

```sql
UPDATE public.blog_topics
SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

(Note: `routine.md`'s older single-table-write version says to leave `blog_post_id` NULL — that
was true only when the routine wrote nothing else. Now that `blog_posts` rows exist, set it; it's
the FK linking the topic to its post.)

## 3. `blog_posts` — one row per drafted article, `status` always `'draft'`

This is Mila's review/schedule/publish queue. **Never** insert as `'scheduled'` or `'published'` —
that's her call.

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, focus_keyword, topic_category,
  target_keywords, status, duda_page_id, duda_publish_url, topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<url-slug>', '<meta description>', '<primary keyword>',
  '<category>', ARRAY['kw1','kw2'], 'draft', '<duda post id>', '<duda draft url>',
  '<topic_id>', '<run_id>', '<jsonb quality_scores>'
) RETURNING id;
```

Required (`NOT NULL`): `client_id`, `title`, `slug`. Everything else nullable. Useful columns not
in the insert above: `excerpt`, `content` (the full body — optional to duplicate here since Duda
is the real store, but handy for search), `seo_score`/`readability_score` (numeric, unused by the
routine so far), `featured_image_url`/`featured_image_alt`, `scheduled_for`/`scheduled_publish_date`
(Mila sets these, not the routine), `published_at`, `rejection_reason` (fill if Mila/QA rejects
later), `revision_count`.

`quality_scores` is freeform jsonb — put the doctrine-compliance verdict here (see the
`seo-doctrine` skill's "Doctrine-compliance check" block): word count, H2 count, internal link
count, per-rule PASS/FAIL, sources cited, and the overall verdict (SHIP / FIX_FIRST / ESCALATE).
This is what a human skimming the review queue actually reads, so make it legible, not just
machine-parseable.

## 4. `tasks` + `task_assignees` — notify Mila, only when a draft was created

Look up Mila's `user_id` and the client's "To Do" `task_columns.id` (per-client, not global):

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

If the client has no `task_columns` rows at all (new client, board never initialized), fall back
to leaving `column_id` NULL — `tasks.status` (the enum) is still set correctly and the row will
still show up in her task list, just outside the kanban board until a column exists.

```sql
INSERT INTO public.tasks (
  client_id, title, description, status, priority, assignee_user_id, column_id,
  external_source, external_ref
) VALUES (
  '<client_id>', 'Review blog draft: <post title>',
  'Auto Blogger created a draft. Verdict: <SHIP/FIX_FIRST>. Duda draft: <url>. <any flags>.',
  'to_do', 'normal', '<mila_user_id>', '<to_do_column_id>', 'auto_blogger', '<blog_posts.id>'
) RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_user_id>');
```

`tasks.status` is a Postgres enum `task_status` (default `'to_do'`); `tasks.priority` is enum
`task_priority` (default `'normal'`). Use the enum value directly (`'to_do'`), not the
`task_columns.key`/`name` — those are display metadata for the kanban board, not the same as the
status enum, though they're kept in sync by `task_columns.mapped_status`.

`external_source`/`external_ref` aren't required by any constraint but are worth setting
(`'auto_blogger'` / the `blog_posts.id`) so a task can be traced back to the post that spawned it
without depending on the free-text description.

## Gotchas

- `blog_topics.topic_title` and `blog_posts.title` — different table, different column name for
  the same concept. `blog_topics.summary` is the field `routine.md`'s own Step 4a query selects;
  `description` is a separate, mostly-unused column on the same table.
- `tasks.status`/`priority` are enums (`task_status`/`task_priority`), not free text — use the
  known values (`to_do`, `in_progress`, `done`; `normal`, and whatever else the enum defines) or
  the insert will fail typecast, not silently coerce.
- `blog_posts.status` (draft/scheduled/published) is a **plain text column**, not the same enum —
  don't confuse it with `tasks.status`.
- Internal links in the draft body should point at `blog_posts`/Duda posts that are actually
  **published**, not just present in `list_blog_posts` — with dozens of drafts sitting unpublished
  in Mila's queue at any given time, linking to one risks a dead link if this draft goes live
  before the one it links to does. Filter `list_blog_posts` results to `"status": "PUBLISHED"`
  before building the internal-link shortlist.
