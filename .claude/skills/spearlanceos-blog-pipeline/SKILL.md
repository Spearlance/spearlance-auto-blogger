---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back to SpearlanceOS (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read during Step 0d (open the run log) and the Publish step of routine.md, every run.
---

# SpearlanceOS blog pipeline — write-back reference

`routine.md` writes to five things each client-run: `blog_topics`, `blog_posts`, `blog_auto_runs`,
`tasks`, `task_assignees`. This file is the schema/enum reference routine.md points to so you don't
have to re-derive it from `information_schema` every tick. All of it lives in Supabase project
`chikljxwgiskyjsnjelf` (production — see the `supabase-prod-guard` skill before writing anything;
these five tables are the routine's standing, pre-authorized write scope, nothing else is).

> **Provenance:** introspected live from `information_schema`/`pg_constraint`/`pg_trigger` on
> 2026-09-04 because this file didn't exist yet, even though routine.md and multiple prior
> `blog_auto_runs.error_log` entries referenced it. If a write below starts failing on a column/enum
> that no longer matches, re-introspect and fix this file rather than guessing.

## Known DB constraint: one draft per client per day

`public.blog_posts` has `trg_one_blog_draft_per_client_per_day` (`BEFORE INSERT`, function
`guard_one_blog_draft_per_client_per_day()`, confirmed enabled). It rejects a second `blog_posts`
insert for the same `client_id` on the same calendar day. `routine.md` Step 0c intentionally does not
gate due-ness on same-day drafts, so this trigger and that step disagree — reconciling them is
Garrett's call. Until then, treat an insert rejection here as informational (that client already has
today's draft) rather than an error to retry.

## `blog_topics`

Already documented in `routine.md`/`CLAUDE.md`. Relevant columns: `id`, `client_id`, `topic_title`,
`description`, `summary`, `keywords[]`, `category`, `priority` (text, default `'medium'`), `status`
(text, default `'idea'`; routine uses `'idea'` → `'in_progress'` → `'published'`), `blog_post_id`,
`ai_generated`, `suggested_publish_date`, `avatar_id`, `strategy_batch_id`.

```sql
-- mark a used/created topic done (Publish step, item 1)
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

## `blog_auto_runs` — one row per client you process

| column | type | notes |
|---|---|---|
| `id` | uuid | default `gen_random_uuid()` |
| `client_id` | uuid | required |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | **CHECK**: `'scheduled'` or `'manual'` only |
| `status` | text | default `'running'`; **CHECK**: `'running'`, `'completed'`, `'failed'` only |
| `topics_generated` | int | default 0 |
| `articles_generated` | int | default 0 |
| `articles_passed_gate` | int | default 0 |
| `articles_flagged` | int | default 0 |
| `research_summary` | jsonb | nullable |
| `completed_at` | timestamptz | nullable |
| `error_log` | text | nullable — free text; put the full narrative here (why declined, what was checked, standing issues) |
| `created_at` | timestamptz | default `now()` |

Open it at the start of the client's processing (Step 0d):

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;  -- keep as <run_id>
```

Close it once, at the end of this client's branch — either a single `UPDATE` on `<run_id>`, or (as
prior ticks have done when the whole branch resolves synchronously) a single `INSERT` already carrying
the final `status`/`error_log`/`completed_at` instead of insert-then-update. Both are fine; don't leave
a row stuck at `status='running'`.

```sql
-- success path
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = <n>, articles_generated = <n>, articles_passed_gate = <n>, articles_flagged = <n>
WHERE id = '<run_id>';

-- declined/failed path
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<what you checked and why you declined>'
WHERE id = '<run_id>';
```

## `blog_posts` — one row per drafted article

| column | type | notes |
|---|---|---|
| `id` | uuid | default `gen_random_uuid()` |
| `client_id` | uuid | required |
| `topic_id` | uuid | FK to `blog_topics.id` |
| `auto_run_id` | uuid | FK to `blog_auto_runs.id` |
| `title` | text | required |
| `slug` | text | required |
| `meta_description` | text | |
| `content` | text | |
| `excerpt` | text | |
| `focus_keyword` | text | |
| `target_keywords` | text[] | |
| `topic_category` | text | |
| `featured_image_url` / `featured_image_alt` | text | |
| `body_images` | jsonb | |
| `status` | text | default `'draft'` — **always insert as `'draft'`**, never `'scheduled'`/`'published'` (that's Mila's call) |
| `duda_page_id` | text | the Duda post ID from `create_blog_post`'s response |
| `duda_publish_url` | text | Duda preview/editor URL |
| `quality_scores` | jsonb | the doctrine QA verdict (rule-by-rule, see `seo-doctrine` skill) |
| `scheduled_publish_date` / `scheduled_for` / `published_at` | date/timestamptz | leave null — Mila sets these |
| `rejection_reason` | text | if applicable |
| `avatar_id`, `ai_model`, `generation_prompt`, `generation_metadata`, `created_by`, `last_edited_by`, `revision_count` | | optional/bookkeeping |

```sql
INSERT INTO public.blog_posts (
  client_id, topic_id, auto_run_id, title, slug, meta_description, content, excerpt,
  focus_keyword, target_keywords, topic_category, featured_image_url, featured_image_alt,
  status, duda_page_id, duda_publish_url, quality_scores
) VALUES (
  '<client_id>', '<topic_id>', '<run_id>', '<title>', '<slug>', '<meta_description>', '<content>',
  '<excerpt>', '<focus_keyword>', ARRAY['<kw1>','<kw2>'], '<category>', '<image_url>', '<alt_text>',
  'draft', '<duda_post_id>', '<duda_preview_url>', '<quality_scores_json>'::jsonb
)
RETURNING id;  -- keep as <blog_post_id> for blog_topics.blog_post_id and the task below
```

Remember: the DB will reject this insert if the client already has a `blog_posts` row for today (see
the constraint above) — check first, or be prepared for the insert to fail and treat that as "already
capped for today," not a bug.

## `tasks` + `task_assignees` — the review task for Mila

`tasks.status` and `tasks.priority` are **enums**, not free text:

- `task_status`: `to_do`, `in_progress`, `done`
- `task_priority`: `low`, `normal`, `high`, `urgent`

Relevant `tasks` columns: `id`, `client_id`, `title`, `description`, `status` (default `'to_do'`),
`priority` (default `'normal'`), `column_id` (FK to `task_columns.id` — must be the client's own `to_do`
column, not a global one), `due_date`, `assignee_user_id`, `creator_user_id`.

Mila's identity (in `public.profiles`, not `public.users` — there is no `public.users` table):

```sql
SELECT id, name, email FROM public.profiles WHERE email = 'mila@spearlance.com';
-- id: 4b9df73e-bc38-49ef-840e-75fdb0b29c29 (re-verify — don't hardcode without checking)
```

The client's `to_do` column:

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

Create the task, then assign it:

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, column_id, assignee_user_id)
VALUES (
  '<client_id>',
  'Review draft: <post title> (<client name>)',
  'Auto Blogger created this draft — review and schedule/publish in Duda. Preview: <duda_preview_url>',
  'to_do', 'normal', '<to_do_column_id>', '<mila_profile_id>'
)
RETURNING id;  -- keep as <task_id>

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_profile_id>');
```

(`tasks.assignee_user_id` and the `task_assignees` join row are both populated in the live data — set
both, matching existing convention, rather than relying on just one.)

## Full per-client write sequence (Publish step)

1. `UPDATE blog_topics SET status='published' WHERE id=<topic_id>`
2. `INSERT INTO blog_posts (... status='draft' ...) RETURNING id`
3. `UPDATE blog_auto_runs SET status='completed', completed_at=now(), ... WHERE id=<run_id>`
4. `INSERT INTO tasks (...) RETURNING id` then `INSERT INTO task_assignees (...)`

If the branch is stopped before a draft is produced (QA fail, no fuel, no strategy row, doctrine halt,
topic-space exhausted, escalation), skip 1/2/4 and only close `blog_auto_runs` with `status='failed'`
and a descriptive `error_log`.
