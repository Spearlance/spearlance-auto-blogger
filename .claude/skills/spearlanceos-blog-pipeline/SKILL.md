---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the four SpearlanceOS tables the Auto Blogger routine writes to (blog_topics, blog_posts, blog_auto_runs, tasks + task_assignees). Read this during Auto Blogger's per-client run whenever routine.md says "see the spearlanceos-blog-pipeline skill" — opening the run log at the start of a client's processing, and writing back after a draft is created or a branch is stopped.
---

# SpearlanceOS blog write-back pipeline

The Auto Blogger routine (`routine.md`) reads most of SpearlanceOS Supabase (project id
**`chikljxwgiskyjsnjelf`**) read-only and writes to exactly four tables:
`blog_auto_runs`, `blog_topics`, `blog_posts`, `tasks` (+ `task_assignees`). This skill is the
schema reference so you never have to re-derive it via `information_schema` mid-run. Do not write
to any other table. Never call SpearlanceOS's `blog-*` edge functions.

Schema below was captured 2026-08-07 via `information_schema.columns` / `pg_enum` against the live
project. If a write fails on an unknown column, the schema has drifted — re-check
`information_schema.columns` for that table and update this file.

## Sequence per client

1. **Open** a `blog_auto_runs` row (Step "Open the run log") *before* any research/drafting.
2. Do the work (research, draft, doctrine QA).
3. **Close** that same `blog_auto_runs` row exactly once, whether or not a draft was created.
4. Only if a draft was created and published to Duda as a hold/draft:
   - insert `blog_topics` (or update the existing row if a calendar topic was used)
   - insert `blog_posts`
   - insert `tasks` + `task_assignees`

---

## `blog_auto_runs` — one row per client per tick; the system of record for what a tick did

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, `gen_random_uuid()` default |
| `client_id` | uuid | required |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | `'scheduled'` for the hourly cron; `'manual'` only for an ad hoc single-`client_id` invocation |
| `status` | text | `'running'` (default, set on open) → `'completed'` or `'failed'` (set on close) |
| `topics_generated` | int | default 0 |
| `articles_generated` | int | default 0 |
| `articles_passed_gate` | int | default 0 |
| `articles_flagged` | int | default 0 |
| `research_summary` | jsonb | free-form: target keyword, intent, source URLs cited, etc. |
| `completed_at` | timestamptz | set on close |
| `error_log` | text | set on close **only** when `status='failed'` — why the branch stopped |
| `created_at` | timestamptz | default `now()` |

**Open** (run this first, before Step 1 doctrine read):

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;  -- keep as <run_id>
```

**Close — draft created:**

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', articles_generated = 1, articles_passed_gate = 1,
    topics_generated = <0 or 1>, completed_at = now(),
    research_summary = '<jsonb: {"keyword": "...", "intent": "...", "sources": ["url1","url2"]}>'
WHERE id = '<run_id>';
```

**Close — branch stopped (QA fail / no fuel / no strategy / doctrine halt / escalation):**

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', articles_generated = 0, completed_at = now(),
    error_log = '<why: which gate failed, or "no strategy row configured", etc.>'
WHERE id = '<run_id>';
```

A client that isn't due this tick never gets a `blog_auto_runs` row at all — only clients actually
processed (Step 0d scope) get one opened and closed.

---

## `blog_topics` — the topic queue; `status='published'` means "turned into a post," not "live"

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | required |
| `topic_title` | text | required |
| `description` | text | (routine.md calls this `summary` in its SELECT — the actual column is `description`; there is a separate `summary` column too, see below) |
| `summary` | text | short summary; both `description` and `summary` exist, populate `summary` (matches routine.md's read query) |
| `keywords` | text[] | |
| `category` | text | one of the client's `content_mix` categories |
| `avatar_id` | uuid | optional, usually null |
| `priority` | text | default `'medium'` |
| `status` | text | default `'idea'`. Lifecycle: `'idea'` (from calendar) → `'in_progress'` (claimed this run, Step 4a) → `'published'` (turned into a `blog_posts` row — **not** a claim the post is live) |
| `blog_post_id` | uuid | set once a post exists for this topic (dedup check in Step 4a filters `WHERE blog_post_id IS NULL`) |
| `ai_generated` | boolean | default false; set `true` when the routine generated the topic (Step 4b) |
| `suggested_publish_date` | date | Step 4a due-check compares this `<= current_date` |
| `strategy_batch_id` | uuid | optional, usually null for AI-generated topics |
| `created_at` | timestamptz | default `now()` |

**Claim an existing calendar topic (Step 4a):**

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**Generate + insert a new topic (Step 4b):**

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', true, 'in_progress', current_date)
RETURNING id;  -- keep as <topic_id>
```

**Mark done once the draft is created (Publish step, write #1):**

```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts_id>' WHERE id = '<topic_id>';
```

---

## `blog_posts` — one row per drafted article; Mila's actual review/schedule/publish queue

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | required |
| `title` | text | required |
| `slug` | text | required — derive from the Duda post `path` |
| `meta_description` | text | |
| `content` | text | full HTML body as sent to Duda |
| `excerpt` | text | short teaser, optional |
| `focus_keyword` | text | primary target keyword |
| `seo_score` / `readability_score` | numeric | optional, leave null unless you scored them |
| `avatar_id` | uuid | optional |
| `topic_category` | text | matches the topic's `category` |
| `target_keywords` | text[] | |
| `featured_image_url` / `featured_image_alt` | text | the Duda main/thumbnail image used |
| `body_images` | jsonb | optional list of in-body image refs |
| `status` | text | **always `'draft'`** on insert — never `'scheduled'` or `'published'`; that's Mila's call |
| `scheduled_for` / `published_at` | timestamptz | leave null — the routine never sets these |
| `duda_page_id` | text | the Duda `create_blog_post` response id |
| `duda_publish_url` | text | preview/editor URL from the Duda response |
| `ai_model` | text | default `'claude-sonnet-4-5'`; set to the model actually used |
| `generation_prompt` / `generation_metadata` | text / jsonb | optional context |
| `created_by` / `last_edited_by` | uuid | leave null (system-authored) |
| `topic_id` | uuid | FK to `blog_topics.id` |
| `scheduled_publish_date` | date | `current_date` |
| `images_generated` | boolean | default false (Duda stock images used, not the AI image pipeline) |
| `auto_run_id` | uuid | FK to `blog_auto_runs.id` |
| `quality_scores` | jsonb | the doctrine QA verdict (rule-by-rule or summary) |
| `rejection_reason` | text | only set if inserting a failed draft for audit purposes (routine.md doesn't require this — normally a QA fail just closes `blog_auto_runs` as `'failed'` and no `blog_posts` row is created at all) |
| `revision_count` | int | default 0 |

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status, duda_page_id,
   duda_publish_url, ai_model, topic_id, auto_run_id, quality_scores, scheduled_publish_date)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta description>', '<html content>', '<keyword>',
   '<category>', ARRAY['kw1','kw2'], '<image url>', '<alt text>', 'draft', '<duda post id>',
   '<duda preview url>', 'claude-sonnet-5', '<topic_id>', '<run_id>',
   '<jsonb QA verdict>', current_date)
RETURNING id;  -- keep as <blog_post_id>
```

---

## `tasks` + `task_assignees` — one review task for Mila per draft created

`tasks.status` and `tasks.priority` are Postgres enums, not free text:

- `task_status`: `to_do` | `in_progress` | `done`
- `task_priority`: `low` | `normal` | `high` | `urgent`

`tasks.column_id` should point at the client's **`to_do`** row in `task_columns` (a per-client
Kanban column config) so the task shows up in the right place:

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

Look up Mila's user id once per run (stable, safe to cache within a run):

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Create the task and assign it:

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, column_id, assignee_user_id)
VALUES
  ('<client_id>', 'Review blog draft: <post title>',
   '<duda preview url> — <one-line summary of the post and why it was drafted>',
   'to_do', 'normal', '<to_do column_id>', '<mila_user_id>')
RETURNING id;  -- keep as <task_id>

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_user_id>');
```

`tasks.assignee_user_id` and the `task_assignees` row are set redundantly on purpose — the schema
carries both a single-assignee column and a join table; populate both so the task shows up
correctly regardless of which one SpearlanceOS's UI reads for a given view.

---

## Full write-back order for one successful draft

```sql
-- 1. mark the topic done
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_post_id>' WHERE id = '<topic_id>';
-- (blog_posts insert already ran to get <blog_post_id> — see above)
-- 2. close the run log
UPDATE public.blog_auto_runs SET status = 'completed', articles_generated = 1,
  articles_passed_gate = 1, completed_at = now(), research_summary = '<jsonb>'
  WHERE id = '<run_id>';
-- 3. task + assignee (see above)
```
