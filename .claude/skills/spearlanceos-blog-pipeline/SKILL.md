---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS (Supabase, project chikljxwgiskyjsnjelf) table columns, enum values, and copy-pasteable SQL for the Auto Blogger's write-back sequence — blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees. Read this before writing any of these tables so column names and statuses match the real schema exactly.
---

# SpearlanceOS Blog Pipeline — write-back reference

Schema below was pulled directly from `information_schema.columns` on project `chikljxwgiskyjsnjelf`
(production — confirmed via `list_branches`, `is_default: true`). Re-verify if this file is ever stale;
don't hand-guess column names for these tables.

The routine writes to five places, per client, per run: `blog_auto_runs` (open at start, close at end),
`blog_topics` (mark the topic used), `blog_posts` (one row per drafted article), and `tasks` +
`task_assignees` (one review task for Mila, only when a draft was created).

## `blog_auto_runs` — one row per client per tick

Columns: `id` (uuid, default), `client_id` (uuid, required), `triggered_at` (timestamptz, default now()),
`trigger_type` (text, required — `'scheduled'` for the hourly cron, `'manual'` for a single-client
dry run), `status` (text, default `'running'`), `topics_generated`/`articles_generated`/
`articles_passed_gate`/`articles_flagged` (int, default 0), `research_summary` (jsonb, nullable),
`completed_at` (timestamptz, nullable), `error_log` (text, nullable), `created_at` (default now()).

Open at the start of a client's processing:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Close at the end — success:

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',
    topics_generated = <0 or 1>,
    articles_generated = <0 or 1>,
    articles_passed_gate = <0 or 1>,
    articles_flagged = 0,
    research_summary = '<jsonb: {"sources": [...], "keyword": "..."}>'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';
```

Close at the end — stopped branch (QA fail, no fuel, no strategy row, doctrine halt, escalation):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed',
    error_log = '<why: e.g. "QA FAIL — unsourced growth stat in section 3">',
    completed_at = now()
WHERE id = '<run_id>';
```

## `blog_topics` — mark the topic used

Columns: `id`, `client_id`, `topic_title` (text, required), `description` (text), `summary` (text),
`keywords` (text[]), `avatar_id` (uuid), `priority` (text, default `'medium'`), `status` (text, default
`'idea'` — observed values include `idea`, `in_progress`, `published`), `blog_post_id` (uuid, nullable),
`ai_generated` (bool, default false), `strategy_batch_id` (uuid), `suggested_publish_date` (date),
`category` (text), `created_at`.

Generating a new topic (Step 4b):

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>', true, 'in_progress', current_date)
RETURNING id;
```

Marking a calendar topic in progress (Step 4a) / done after a draft ships:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
-- ...after the Duda draft + blog_posts row are created:
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

`status = 'published'` here means "turned into a post" (matches the table's existing convention), not
"live on the client's site" — every Duda post this routine creates is left unpublished; see CLAUDE.md.

## `blog_posts` — one row per drafted article (Mila's review/schedule queue)

Columns: `id`, `client_id` (required), `title` (required), `slug` (required), `meta_description`,
`content`, `excerpt`, `focus_keyword`, `seo_score` (numeric), `readability_score` (numeric), `avatar_id`,
`topic_category`, `target_keywords` (text[]), `featured_image_url`, `featured_image_alt`, `body_images`
(jsonb), `status` (text, default `'draft'` — **always insert as `'draft'`**, never `'scheduled'` or
`'published'` — that's Mila's call), `scheduled_for` (timestamptz), `published_at` (timestamptz),
`duda_page_id` (text), `duda_publish_url` (text), `ai_model` (text, default
`'claude-sonnet-4-5'` — override to whatever's actually running), `generation_prompt`,
`generation_metadata` (jsonb), `created_by` (uuid), `topic_id` (uuid, FK to `blog_topics.id`),
`scheduled_publish_date` (date), `images_generated` (bool, default false), `auto_run_id` (uuid, FK to
`blog_auto_runs.id`), `quality_scores` (jsonb — put the doctrine rule-by-rule verdict here),
`rejection_reason` (text), `revision_count` (int, default 0).

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword, target_keywords,
   topic_category, featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
   topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta description>', '<full HTML content>', '<excerpt>',
   '<primary keyword>', ARRAY['<kw1>','<kw2>'], '<category>', '<image url>', '<alt text>', 'draft',
   '<duda post id>', '<duda preview/editor url>', '<topic_id>', '<run_id>',
   '<jsonb doctrine verdict, e.g. {"title":"PASS","description":"PASS","headings":"PASS","content":"PASS","interlinking":"PASS","schema":"PASS","intent":"PASS","verdict":"SHIP"}>'::jsonb)
RETURNING id;
```

## `tasks` + `task_assignees` — the review task for Mila

`tasks` columns: `id`, `client_id` (required), `title` (required), `description`, `status`
(`task_status` enum: `to_do` | `in_progress` | `done`; default `'to_do'`), `priority` (`task_priority`
enum: `low` | `normal` | `high` | `urgent`; default `'normal'`), `assignee_user_id` (uuid),
`creator_user_id` (uuid), `due_date` (date), `related_asset_ids`/`related_meeting_ids` (uuid[]),
`activity_log` (text), `column_id` (uuid, FK to `task_columns.id`), plus recurrence/subtask fields not
relevant here.

`task_columns` is per-client — look up the client's own `to_do` column (`key = 'to_do'`), don't reuse
another client's column id:

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

Mila's `profiles.id` (uuid) — look up once, cache for the run, don't hardcode across sessions since the
row could change:

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Create the task, then the assignee row:

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger created a draft for review. Duda editor: <preview url>. Keyword: <keyword>. Sources: <url1>, <url2>.',
  'to_do', 'normal', '<mila_profile_id>', '<to_do_column_id_for_this_client>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_profile_id>');
```

## Order of operations (per client, after a draft clears QA and is created in Duda)

1. `UPDATE blog_topics ... status='published'` (or the topic id from 4b if generated this run).
2. `INSERT INTO blog_posts ...` (status `'draft'`, `topic_id`, `auto_run_id` set) → `RETURNING id`.
3. `UPDATE blog_auto_runs SET status='completed', ... completed_at=now()` on the run row opened at the
   start of this client's processing.
4. `INSERT INTO tasks ...` then `INSERT INTO task_assignees ...` for Mila.

If the branch was stopped instead (QA fail, no fuel, no strategy, doctrine halt) — skip steps 2 and 4,
still close the `blog_auto_runs` row as `'failed'` with `error_log` set, and leave the topic row as-is
(or `'in_progress'` if you don't want it reselected next tick — team's call; don't mark `'published'`
for a post that was never drafted).

## Notes

- These are production writes (`chikljxwgiskyjsnjelf`, `is_default: true`). This routine has standing,
  documented authorization to write these five tables as part of its normal hourly operation — that
  authorization does not extend to any other table or to DDL.
- Re-run the `information_schema.columns` queries above if a write fails on an unknown column — schema
  may have moved on since this file was last verified.
