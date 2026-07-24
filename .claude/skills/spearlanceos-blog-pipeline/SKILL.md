---
name: spearlanceos-blog-pipeline
description: Reference for writing into SpearlanceOS's blog production pipeline via the Supabase MCP (project chikljxwgiskyjsnjelf) — blog_topics, blog_posts (the draft/scheduled/published record), blog_auto_runs (per-run log), and tasks/task_assignees (Mila's review queue). Use this whenever creating or updating a blog draft's SpearlanceOS record, logging an Auto Blogger run, checking or building Mila's review-task queue, or doing any ad hoc lookup, backfill, or debugging across these tables — not just from inside the Auto Blogger routine itself. Encodes exact column names, status values, and copy-pasteable SQL so nobody has to rediscover the schema from scratch.
---

# SpearlanceOS Blog Pipeline — Schema Reference

This is the write-shape reference for the four SpearlanceOS tables a blog post moves through on its way
from idea to something Mila can schedule. `routine.md` (the Auto Blogger) is the main consumer, but
anything else touching this pipeline — checking Mila's queue, backfilling a missed run log, fixing a
stuck task — should read this first instead of re-deriving the schema from a fresh `list_tables` sweep.

## Before writing anything

Project `chikljxwgiskyjsnjelf` is **production**. Reads are always fine, no need to ask. The INSERT/UPDATE
patterns below are the Auto Blogger routine's normal, already-authorized operation — `routine.md` and
`CLAUDE.md` document exactly when the routine performs them, and running them as part of that routine
doesn't need a fresh yes each time. If you're using this skill for something *else* — an ad hoc check, a
backfill, a one-off fix, anything not already described in `routine.md` — that's a production write
outside the routine's standing authorization, so it still needs Garrett's explicit go-ahead. See the
`supabase-prod-guard` skill for what counts as a write and what an explicit yes looks like; this skill
doesn't repeat that, it just tells you the shapes to use once you have the green light.

## The shape of a post: four tables, in the order it moves through them

```
blog_topics (idea) → [drafted] → blog_posts (status='draft') → Mila reviews →
                                  blog_posts (status='scheduled' / 'published')
blog_auto_runs — logs the attempt, independent of whether it produced a post
tasks + task_assignees — the nudge that gets it in front of Mila
```

### 1. `blog_topics` — the idea queue

| column | type | notes |
|---|---|---|
| `id` | uuid | |
| `client_id` | uuid | |
| `topic_title` | text | |
| `description` | text | |
| `summary` | text | |
| `keywords` | text[] | |
| `category` | text | |
| `avatar_id` | uuid | FK, usually null |
| `priority` | text | default `'medium'` |
| `status` | text | `'idea'` (default) → `'in_progress'` → `'published'` |
| `blog_post_id` | uuid | legacy dedup field; not the FK to `blog_posts` (see below) |
| `ai_generated` | boolean | |
| `strategy_batch_id` | uuid | FK, set when the topic came from a monthly batch |
| `suggested_publish_date` | date | |

**`status='published'` here means "turned into a post," not "live."** That's the existing convention —
180 of the current 259 rows use it that way. Don't reinterpret it as a publish signal.

```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

### 2. `blog_posts` — the actual review/schedule/publish record

This is the table Mila works from — it's what a "blog calendar" or "content queue" view in SpearlanceOS
would be built on. Distinct rows exist per drafted article; as of writing there are only 16 rows total
(7 `draft`, 7 `scheduled`, 2 `published`), so this table is lightly used and safe to reason about in full.

| column | type | notes |
|---|---|---|
| `id` | uuid | |
| `client_id` | uuid | |
| `title` | text | not null |
| `slug` | text | not null — kebab-case the title |
| `meta_description` | text | |
| `content` | text | full HTML body |
| `excerpt` | text | intro/summary paragraph |
| `focus_keyword` | text | the target keyword |
| `target_keywords` | text[] | |
| `seo_score` / `readability_score` | numeric | optional, leave null unless you're computing them |
| `avatar_id` | uuid | |
| `topic_category` | text | |
| `featured_image_url` / `featured_image_alt` | text | |
| `body_images` | jsonb | optional |
| `status` | text | **`'draft'` \| `'scheduled'` \| `'published'`** — see below |
| `scheduled_for` | timestamptz | when Mila schedules it |
| `scheduled_publish_date` | date | |
| `published_at` | timestamptz | |
| `duda_page_id` | text | the Duda post id from `create_blog_post`'s response |
| `duda_publish_url` | text | |
| `ai_model` | text | default `'claude-sonnet-4-5'` — **set explicitly** to the model actually used (e.g. `'claude-sonnet-5'`) rather than relying on the stale default |
| `generation_prompt` | text | optional |
| `generation_metadata` | jsonb | source URLs cited, keyword data used, internal links used |
| `quality_scores` | jsonb | the doctrine QA verdict (the §1..§14.3 rule-by-rule table) |
| `rejection_reason` | text | fill in if a QA fail produced this row anyway (usually you just won't insert a row at all — see below) |
| `topic_id` | uuid | FK → `blog_topics.id` |
| `auto_run_id` | uuid | FK → `blog_auto_runs.id` |
| `created_by` / `last_edited_by` | uuid | FK → `profiles.id`, leave null for automated inserts |
| `images_generated` | boolean | |
| `revision_count` | integer | default 0 |

**Only `status='draft'` gets inserted by an automated writer.** Never insert as `'scheduled'` or
`'published'` — that's Mila's action, taken through the SpearlanceOS UI, not something to pre-empt.

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt, focus_keyword, target_keywords,
  featured_image_url, featured_image_alt, status, duda_page_id, topic_id, auto_run_id,
  quality_scores, generation_metadata, ai_model
) VALUES (
  '<client_id>', '<title>', '<slug>', '<meta_description>', '<full HTML content>',
  '<intro/excerpt>', '<target_keyword>', ARRAY['<kw1>','<kw2>'],
  '<thumbnail_url>', '<thumbnail_alt>', 'draft', '<duda_post_id>', '<topic_id>', '<run_id>',
  '<jsonb verdict>'::jsonb, '<jsonb sources/keywords/links>'::jsonb, 'claude-sonnet-5'
);
```

To check what's sitting in Mila's queue right now:

```sql
SELECT client_id, title, status, duda_page_id, scheduled_publish_date, created_at
FROM public.blog_posts
WHERE status = 'draft'
ORDER BY created_at;
```

### 3. `blog_auto_runs` — the per-client run log

One row per client per attempt — logs what happened whether or not it produced a post. Open it at the
start of processing a client, close it at the end.

| column | type | notes |
|---|---|---|
| `id` | uuid | |
| `client_id` | uuid | not null |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | **CHECK: `'scheduled'` \| `'manual'`** |
| `status` | text | **CHECK: `'running'` \| `'completed'` \| `'failed'`** |
| `topics_generated` | integer | default 0 |
| `articles_generated` | integer | default 0 |
| `articles_passed_gate` | integer | default 0 |
| `articles_flagged` | integer | default 0 |
| `research_summary` | jsonb | sources + keyword data used |
| `error_log` | text | why the branch stopped, if it did |
| `completed_at` | timestamptz | |

```sql
-- open, at the start of processing this client
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;

-- close on success
UPDATE public.blog_auto_runs
SET status = 'completed', articles_generated = 1, articles_passed_gate = 1,
    topics_generated = <0 or 1>, completed_at = now(),
    research_summary = '<jsonb>'::jsonb
WHERE id = '<run_id>';

-- close on a stopped branch (QA fail, no fuel, no strategy row, doctrine halt, escalation)
UPDATE public.blog_auto_runs
SET status = 'failed', error_log = '<why>', articles_generated = 0, completed_at = now()
WHERE id = '<run_id>';
```

Every row opened with `status='running'` should get closed one way or the other — a `blog_auto_runs`
query where `status='running'` and `triggered_at` is more than an hour or two old means something crashed
mid-run without closing its own log row; that's worth a look before writing it off as noise.

### 4. `tasks` + `task_assignees` + `task_columns` — Mila's review nudge

Only create this when a draft was actually produced. `tasks` uses real Postgres enums, not free text:

| column | type | notes |
|---|---|---|
| `id` | uuid | |
| `client_id` | uuid | not null |
| `title` / `description` | text | |
| `status` | `task_status` enum | `'to_do'` \| `'in_progress'` \| `'done'` — start at `'to_do'` |
| `priority` | `task_priority` enum | `'low'` \| `'normal'` \| `'high'` \| `'urgent'` |
| `assignee_user_id` | uuid | FK → `profiles.id` |
| `creator_user_id` | uuid | leave null for automated inserts |
| `due_date` | date | |
| `column_id` | uuid | FK → `task_columns.id`, the per-client kanban board column |

`task_columns` is per-client — every client has its own `'To Do'`/`'In Progress'`/`'Done'` set (`key`
values `to_do`/`in_progress`/`done`). Look up the right column each time rather than assuming an id:

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do' LIMIT 1;
```

If that returns nothing (rare — a client's board wasn't set up with the standard columns), omit
`column_id` on the insert rather than guess a value from a different client.

**Look up Mila by email, every time — never hardcode her `profiles.id`.** If her account ever changes
(new email, someone else takes over review), a hardcoded UUID silently breaks; a lookup doesn't.

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Then create the task, and mirror the assignment into `task_assignees` — the schema has both an
`assignee_user_id` column on `tasks` and a separate `task_assignees` join table, and it isn't obvious
from the schema alone which one the frontend actually reads for display, so set both:

```sql
INSERT INTO public.tasks (
  client_id, title, description, status, priority, assignee_user_id, column_id, due_date
) VALUES (
  '<client_id>', 'Review & schedule: <post title>',
  'Draft ready in Duda: <duda editor/preview URL>. Keyword: <target_keyword>. Category: <category>. '
    || 'Sources cited: <n>. Doctrine QA: SHIP.',
  'to_do', 'normal', '<mila_profile_id>', '<todo_column_id>', current_date + 2
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<new_task_id>', '<mila_profile_id>');
```

To see everything currently waiting on Mila:

```sql
SELECT t.id, t.client_id, t.title, t.status, t.due_date
FROM public.tasks t
JOIN public.profiles p ON p.id = t.assignee_user_id
WHERE p.email = 'mila@spearlance.com' AND t.status != 'done'
ORDER BY t.due_date NULLS LAST;
```

## Full write, in order, for one drafted article

1. `UPDATE blog_topics SET status='published' WHERE id=<topic_id>`
2. `INSERT INTO blog_posts (... status='draft' ...)`
3. `UPDATE blog_auto_runs SET status='completed', ... WHERE id=<run_id>` (opened before drafting started)
4. `INSERT INTO tasks (...)` + `INSERT INTO task_assignees (...)`

If step 2 never happens (the draft didn't clear QA, or the client had no fuel), still close the
`blog_auto_runs` row from step 3 — with `status='failed'` and an `error_log` — and skip steps 1 and 4.
