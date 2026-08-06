---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase table/column reference and copy-pasteable SQL for the Auto Blogger's writes (blog_auto_runs, blog_topics, blog_posts, tasks, task_assignees). Use this whenever routine.md's Step 0 (open run log), Step 4 (topic select/generate), or Publish/Step 5 (write-back) instruct you to write to SpearlanceOS. Read this before writing to any of these five tables so INSERTs/UPDATEs use correct column names and status values.
---

# SpearlanceOS blog pipeline: table reference + SQL

Project id: `chikljxwgiskyjsnjelf`. All SQL below is copy-pasteable via the Supabase MCP `execute_sql`.
Schema captured 2026-08-06 by direct `information_schema.columns` introspection (this file didn't
exist before; several scheduled ticks worked around its absence — see routine.md's standing asks).
Re-verify column names if a write errors with "column does not exist"; the live schema is the source
of truth, not this file.

## `blog_auto_runs` — one row per client per tick

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, default `gen_random_uuid()` |
| `client_id` | uuid | required |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | required — `'scheduled'` (hourly cron) or `'manual'` (single-`client_id` test run) |
| `status` | text | default `'running'` — set to `'completed'` or `'failed'` on close |
| `topics_generated` | int | default 0 |
| `articles_generated` | int | default 0 |
| `articles_passed_gate` | int | default 0 |
| `articles_flagged` | int | default 0 |
| `research_summary` | jsonb | optional — sources cited, angles checked, etc. |
| `completed_at` | timestamptz | set on close |
| `error_log` | text | set when `status='failed'`, or to note anything worth recording even on success |
| `created_at` | timestamptz | default `now()` |

**Open (Step 0, before any research):**

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Keep the returned `id` as `<run_id>` for the rest of this client's processing.

**Close on success (Publish step):**

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = <0 or 1>, articles_generated = 1, articles_passed_gate = 1,
    research_summary = '<jsonb of sources/angles>'::jsonb
WHERE id = '<run_id>';
```

**Close on stopped branch (QA fail / no fuel / no strategy row / doctrine halt / content saturation):**

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, specific>'
WHERE id = '<run_id>';
```

Every run log opened gets closed one way or the other before moving to the next client.

## `blog_topics` — the topic queue

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | required |
| `topic_title` | text | required |
| `description` | text | nullable — short description |
| `summary` | text | nullable — used interchangeably with `description` in practice; populate both if unsure which the UI reads |
| `keywords` | text[] | |
| `category` | text | should match a `content_mix` key (`how_to`, `case_studies`, `industry_news`, `best_practices`, `company_updates`, etc.) |
| `priority` | text | default `'medium'` |
| `status` | text | default `'idea'` — lifecycle is `idea` → `in_progress` → `published` (routine's own convention: "published" means "turned into a Duda post," not "live") |
| `blog_post_id` | uuid | set once a `blog_posts` row exists for this topic |
| `ai_generated` | boolean | default false — set `true` for routine-generated topics |
| `suggested_publish_date` | date | |
| `strategy_batch_id` | uuid | nullable, links to a calendar batch if the team seeded one |
| `created_at` | timestamptz | |

**Selecting the next due calendar topic (Step 4a):**

```sql
SELECT id, topic_title, summary, description, keywords, category, priority, suggested_publish_date
FROM public.blog_topics
WHERE client_id = '<client_id>' AND blog_post_id IS NULL
  AND status NOT IN ('published', 'in_progress')
  AND suggested_publish_date <= current_date
ORDER BY priority DESC, suggested_publish_date ASC;
```

Mark the chosen row in progress immediately:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**Generating and saving a new topic (Step 4b):**

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, description, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', '<summary>', ARRAY['<kw1>','<kw2>']::text[], '<category>', true, 'in_progress', current_date)
RETURNING id;
```

**Marking a topic done once a draft is created (Publish step):**

```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_post_id>' WHERE id = '<topic_id>';
```

## `blog_posts` — Mila's draft/review queue (one row per article)

Key columns used by the routine (full table has more SEO/CMS fields not needed here):

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | required |
| `title` | text | required |
| `slug` | text | required — derive from the Duda post `path` |
| `meta_description` | text | |
| `content` | text | the HTML body sent to Duda |
| `excerpt` | text | optional short summary |
| `focus_keyword` | text | primary target keyword |
| `target_keywords` | text[] | |
| `topic_category` | text | |
| `featured_image_url` | text | from Duda's `main_image.url` |
| `featured_image_alt` | text | |
| `status` | text | **always `'draft'`** — never `'scheduled'` or `'published'`, that's Mila's call |
| `topic_id` | uuid | FK to `blog_topics.id` |
| `auto_run_id` | uuid | FK to `blog_auto_runs.id` |
| `duda_page_id` | text | the Duda post `id` from `create_blog_post`'s response |
| `duda_publish_url` | text | preview/editor URL from the response |
| `quality_scores` | jsonb | the doctrine QA verdict (rule-by-rule pass/fail + notes) |
| `ai_model` | text | default `'claude-sonnet-4-5'` |
| `created_at` / `updated_at` | timestamptz | default `now()` |

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword, target_keywords,
   topic_category, featured_image_url, featured_image_alt, status, topic_id, auto_run_id,
   duda_page_id, duda_publish_url, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta_description>', '<html_content>', '<excerpt>',
   '<focus_keyword>', ARRAY['<kw1>','<kw2>']::text[], '<category>', '<image_url>', '<image_alt>',
   'draft', '<topic_id>', '<run_id>', '<duda_post_id>', '<duda_preview_url>', '<qa_verdict_json>'::jsonb)
RETURNING id;
```

## `tasks` + `task_assignees` + `task_columns` — the review task for Mila

`tasks.status` and `tasks.priority` are enum types (`task_status`, `task_priority`); use the same
literal values already in use (`status = 'to_do'`, `priority = 'normal'` are safe defaults — do not
invent new enum labels). `tasks.column_id` should point at the client's `'to_do'` column in
`task_columns` (per-client rows; the key is stable as `'to_do'` even though `id`/`name` differ per
client). `related_asset_ids` is a `uuid[]`; put the new `blog_posts.id` in it so the task links to the
draft.

**Look up Mila and the client's To Do column:**

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

**Create the task:**

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id, related_asset_ids)
VALUES ('<client_id>', 'Review blog draft: <post_title>', 'Auto Blogger drafted this post for <client_name>. Review, edit, and schedule/publish in Duda when ready.',
        'to_do', 'normal', '<mila_profile_id>', '<to_do_column_id>', ARRAY['<blog_post_id>']::uuid[])
RETURNING id;
```

**Assign it (assignees are a separate join table, not a column on `tasks`):**

```sql
INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_profile_id>');
```

Only insert a `tasks`/`task_assignees` pair when a draft was actually created. Do not create a task for
a client whose branch was stopped (QA fail, no fuel, etc.) — that's what `blog_auto_runs.error_log`
is for.
