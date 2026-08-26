---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back steps (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read this before writing to any of those tables during a routine.md run.
---

# SpearlanceOS blog pipeline: write-back reference

Project id: `chikljxwgiskyjsnjelf` (production — see `supabase-prod-guard`; this routine's writes to
these five tables are pre-authorized standing behavior per `routine.md`, nothing else is).

This file was missing for weeks despite `routine.md` referencing it throughout, which forced every
tick to re-derive the schema live from `information_schema.columns`. Schema below was captured
2026-08-26 directly from the live database — re-verify with `list_tables`/`information_schema` if it
drifts, don't assume it's still accurate indefinitely.

## Guardrail you must know about before doing any research

`public.blog_posts` has a **BEFORE INSERT** trigger, `trg_one_blog_draft_per_client_per_day`, running
`guard_one_blog_draft_per_client_per_day()`:

```sql
IF EXISTS (
  SELECT 1 FROM public.blog_posts
  WHERE client_id = NEW.client_id AND created_at::date = now()::date
) THEN RAISE EXCEPTION ...
```

**One draft per client per calendar day (UTC), full stop** — it throws, it doesn't silently no-op.
`routine.md` Step 0c's due-check is weekday-only and does **not** check this, so on a due weekday the
hourly cron will pick the same client every tick all day. **Before spending any Tavily/DataForSEO call
or opening a new `blog_auto_runs` row, check cheaply first:**

```sql
SELECT id, created_at, duda_page_id FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at::date = current_date;
```

If a row already exists for today, the guard will reject any insert attempt regardless of how good the
draft is — stop there, don't research, don't draft, and don't bother opening a fresh `blog_auto_runs`
row either if one already exists today recording the same "already satisfied" outcome (check first;
adding a 10th identical row per tick is pure noise, not an audit trail). This gap has caused sustained
Tavily quota exhaustion (HTTP 432) — see the run history on 2026-08-26 for CLE Remodeling, Invictus
Snowfighters, and Sunshine State. **Fixing Step 0c/0d itself requires editing the routine's stored
prompt in the scheduler, which this repo does not control** — flag it in the end-of-run report rather
than assuming a repo-side fix reaches the scheduled task.

## `blog_topics`

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, default `gen_random_uuid()` |
| `client_id` | uuid | required |
| `topic_title` | text | required |
| `description` | text | (routine.md calls this "summary" in the SELECT at Step 4a — it's actually `summary`, see below; `description` is a separate free column) |
| `summary` | text | the topic summary |
| `keywords` | text[] | |
| `avatar_id` | uuid | optional target-audience avatar |
| `priority` | text | default `'medium'` |
| `status` | text | default `'idea'`; lifecycle used by this routine: `'idea'` → `'in_progress'` → `'published'` |
| `blog_post_id` | uuid | set once a `blog_posts` row exists for this topic |
| `ai_generated` | boolean | default `false`; set `true` when the routine generates the topic (Step 4b) |
| `strategy_batch_id` | uuid | optional link to a strategy batch |
| `suggested_publish_date` | date | |
| `category` | text | |
| `created_at` | timestamptz | default `now()` |

**Mark a used calendar topic in-progress (Step 4a):**
```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**Generate + save a new topic (Step 4b):**
```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>', true, 'in_progress', current_date)
RETURNING id;
```

**Close out the topic once the draft is created (Publish step 1):**
```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_post_id>' WHERE id = '<topic_id>';
```
(Per routine.md this means "turned into a post," not "live" — the table's existing convention.)

## `blog_auto_runs`

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | required |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | required — use `'scheduled'` for the hourly cron, `'manual'` only for a single-`client_id` ad hoc invocation |
| `status` | text | default `'running'`; set to `'completed'` or `'failed'` when you close the row |
| `topics_generated` | integer | default 0 |
| `articles_generated` | integer | default 0 |
| `articles_passed_gate` | integer | default 0 |
| `articles_flagged` | integer | default 0 |
| `research_summary` | jsonb | optional — sources, keyword data, etc. |
| `completed_at` | timestamptz | set when closing |
| `error_log` | text | required when `status = 'failed'`; also fine to use for a completed-but-nothing-happened explanation |
| `created_at` | timestamptz | default `now()` |

**Open at the start of a client's processing:**
```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

**Close — draft created:**
```sql
UPDATE public.blog_auto_runs SET
  status = 'completed', completed_at = now(),
  topics_generated = <0 or 1>, articles_generated = 1, articles_passed_gate = 1,
  research_summary = '<jsonb of sources/keywords>'::jsonb
WHERE id = '<run_id>';
```

**Close — branch stopped (no draft):**
```sql
UPDATE public.blog_auto_runs SET
  status = 'failed', completed_at = now(), error_log = '<why: QA fail / no fuel / no strategy row / doctrine halt / escalation / guard already satisfied>'
WHERE id = '<run_id>';
```

## `blog_posts`

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | required |
| `title` | text | required |
| `slug` | text | required |
| `meta_description` | text | |
| `content` | text | the HTML/markdown body |
| `excerpt` | text | |
| `focus_keyword` | text | |
| `seo_score` / `readability_score` | numeric | optional |
| `avatar_id` | uuid | optional |
| `topic_category` | text | |
| `target_keywords` | text[] | |
| `featured_image_url` / `featured_image_alt` | text | |
| `body_images` | jsonb | |
| `status` | text | default `'draft'` — **always insert as `'draft'`**, never `'scheduled'`/`'published'`, that's Mila's call |
| `scheduled_for` / `published_at` | timestamptz | leave null; Mila sets these |
| `duda_page_id` | text | the Duda post ID from `create_blog_post`'s response |
| `duda_publish_url` | text | the Duda preview/editor URL |
| `ai_model` | text | default `'claude-sonnet-4-5'` — override if a different model actually generated it |
| `generation_prompt` / `generation_metadata` | text / jsonb | optional |
| `created_by` / `last_edited_by` | uuid | optional |
| `topic_id` | uuid | link back to `blog_topics.id` |
| `scheduled_publish_date` | date | |
| `images_generated` | boolean | default `false` (images are deferred per CLAUDE.md — leave false) |
| `auto_run_id` | uuid | link to the `blog_auto_runs.id` opened for this client this tick |
| `quality_scores` | jsonb | the doctrine QA verdict (rule-by-rule) |
| `rejection_reason` | text | |
| `revision_count` | integer | default 0 |

**Insert (Publish step 2) — remember the one-draft-per-client-per-day guard above will reject a second same-day insert:**
```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword, target_keywords,
   topic_category, featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
   topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta description>', '<content>', '<excerpt>', '<focus kw>',
   ARRAY['<kw1>','<kw2>'], '<category>', '<duda stock image url>', '<alt text>', 'draft',
   '<duda_page_id>', '<duda preview url>', '<topic_id>', '<run_id>', '<jsonb QA verdict>'::jsonb)
RETURNING id;
```

## `tasks` + `task_assignees` (only when a draft was created)

`tasks.status` and `tasks.priority` are enum types (`task_status`, `task_priority`), not free text —
use the plain literal, no cast needed for common values (`'to_do'`, `'normal'`).

Look up the client's `'to_do'` column and Mila's user id:
```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';  -- verified 2026-08-26: mila@spearlance.com exists here
```

Insert the task, then assign it:
```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, column_id, assignee_user_id)
VALUES ('<client_id>', 'Review blog draft: <post title>', 'Auto Blogger draft ready for review/scheduling. Duda: <duda preview url>',
        'to_do', 'normal', '<to_do column_id>', '<mila_user_id>')
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_user_id>');
```

(`tasks.assignee_user_id` and the separate `task_assignees` join table both exist in this schema —
populate both so the task shows up under Mila regardless of which one the UI reads.)
