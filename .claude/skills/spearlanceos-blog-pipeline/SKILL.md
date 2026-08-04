---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase table columns, status values, and copy-pasteable SQL for the Auto Blogger's write-back steps (opening/closing a run log, inserting a draft record, marking a topic done, and creating Mila's review task). Use this whenever routine.md's Publish or "Open the run log" steps say to consult this skill.
---

# SpearlanceOS Blog Pipeline — write-back reference

This skill documents the **only** four tables the Auto Blogger writes to in SpearlanceOS
(project `chikljxwgiskyjsnjelf`), with real column names and constraints verified against
`information_schema` (not guessed). Everything else in SpearlanceOS is read-only — see
`CLAUDE.md` and `routine.md` for the read paths.

Column names below are current as of 2026-08-04. If a query in this file ever errors on an
unknown column, re-verify with:

```sql
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema='public' AND table_name='<table>'
ORDER BY ordinal_position;
```

---

## 1. `blog_auto_runs` — one row per client per tick

Open this **first**, before any research/drafting, so the attempt is tracked even if the
branch stops early. Close it **once**, at the end of that client's processing.

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, default `gen_random_uuid()` |
| `client_id` | uuid | required |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | **CHECK: only `'scheduled'` or `'manual'`** — not `'cron'`. Use `'manual'` only for an ad hoc single-`client_id` invocation (routine.md Step 0b); every hourly cron tick is `'scheduled'`. |
| `status` | text | **CHECK: only `'running'`, `'completed'`, or `'failed'`** — default `'running'` |
| `topics_generated` | int | default 0 |
| `articles_generated` | int | default 0 |
| `articles_passed_gate` | int | default 0 |
| `articles_flagged` | int | default 0 |
| `research_summary` | jsonb | optional |
| `completed_at` | timestamptz | set when you close the row |
| `error_log` | text | required narrative when `status='failed'`; also fine to fill on `'completed'` for context |
| `created_at` | timestamptz | default `now()` |

**Open (insert) at the start of a client's processing:**

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Keep the returned `id` as `<run_id>` for the rest of this client's processing.

**Close on success** (draft was created and passed QA):

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',
    completed_at = now(),
    articles_generated = 1,
    articles_passed_gate = 1,
    topics_generated = <0 or 1 — 1 only if Step 4b generated a brand-new topic>
WHERE id = '<run_id>';
```

**Close on a stopped branch** (QA fail, no fuel, no strategy row, doctrine halt, no
qualifying angle, escalation, or a tool/connector failure mid-run):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed',
    completed_at = now(),
    error_log = '<why this branch stopped — be specific and non-generic>'
WHERE id = '<run_id>';
```

If you never got as far as opening a row for a client this tick (e.g. it wasn't due), no
`blog_auto_runs` row is needed at all — the due-check is cheap and doesn't need to be logged
for every enrolled client, only for ones that reach "Open the run log" in routine.md.

---

## 2. `blog_topics` — the topic queue

Only the write-backs the routine performs; the strategy team owns everything else here.

**4a — using an existing due topic:** mark it in-progress before drafting starts:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**4b — generating a new topic** (expected for most clients per CLAUDE.md's 2026-06-16 caveat):

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<1-2 sentence summary>', ARRAY['<kw1>','<kw2>'],
   '<category>', true, 'in_progress', current_date)
RETURNING id;
```

Valid `category` values are whatever `blog_content_strategy.content_mix` uses for that client
(e.g. `how_to`, `best_practices`, `case_studies`, `industry_news`, `company_updates`) — bias
toward the client's under-represented categories per CLAUDE.md.

`priority` defaults to `'medium'` (text, not numeric) if you want to set it explicitly for a
generated topic; the 4a `SELECT` orders by it as text so stick to `'low'|'medium'|'high'`
unless you check the client's actual convention first.

**After a draft is created (Publish step), regardless of 4a or 4b:**

```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

This means "turned into a post," matching the table's existing convention — not that it's
live on the site (the Duda post itself stays an unpublished draft).

---

## 3. `blog_posts` — Mila's draft/schedule/publish queue (one row per article)

This is the actual record she works from. Always insert with `status = 'draft'` — never
`'scheduled'` or `'published'`; that's her call.

Relevant columns (full list has more, mostly nullable/unused by the routine):

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | required |
| `title` | text | required |
| `slug` | text | required — URL-safe slug, doesn't need to match Duda's `path` exactly but should be close |
| `meta_description` | text | |
| `content` | text | the full HTML body you sent to `create_blog_post` |
| `excerpt` | text | optional short summary |
| `focus_keyword` | text | the primary target keyword |
| `target_keywords` | text[] | |
| `topic_category` | text | |
| `featured_image_url` | text | the resolved image URL Duda used |
| `featured_image_alt` | text | |
| `status` | text | **always `'draft'`** on insert |
| `duda_page_id` | text | the ID returned by `create_blog_post` |
| `duda_publish_url` | text | preview/editor URL from the response |
| `topic_id` | uuid | FK → `blog_topics.id` |
| `auto_run_id` | uuid | FK → `blog_auto_runs.id` (the `<run_id>` for this client this tick) |
| `quality_scores` | jsonb | the doctrine QA verdict (see shape below) |
| `scheduled_publish_date` | date | leave null — Mila sets this |
| `created_at` / `updated_at` | timestamptz | default `now()` |

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, target_keywords,
   topic_category, featured_image_url, featured_image_alt, status,
   duda_page_id, duda_publish_url, topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta description>', '<full HTML content>',
   '<primary keyword>', ARRAY['<kw1>','<kw2>'], '<category>',
   '<image url used>', '<alt text>', 'draft',
   '<duda post id>', '<duda preview/editor url>', '<topic_id>', '<run_id>',
   '{"verdict":"SHIP","title":"PASS","description":"PASS","headings":"PASS","content":"PASS — 1650 words, 6 H2s","interlinking":"PASS — 7 links","schema":"PASS — Article; FAQ needs manual Custom HTML","intent":"PASS","escalation":"none"}'::jsonb)
RETURNING id;
```

Use the returned `id` when you update `blog_topics.blog_post_id` above.

---

## 4. `tasks` + `task_assignees` — Mila's review task (only when a draft was created)

Look up Mila's profile once per run (stable, safe to hardcode after verifying, but the
`ILIKE` lookup is here so it self-heals if her account ever changes):

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
-- currently: 4b9df73e-bc38-49ef-840e-75fdb0b29c29
```

Find the client's `'to_do'` column (every client has one, `is_default = true`):

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

`tasks.status` and `tasks.priority` are Postgres enums (`task_status`, `task_priority`), not
free text — use `'to_do'` / `'normal'` unless you have a reason to escalate priority.

```sql
INSERT INTO public.tasks
  (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES
  ('<client_id>',
   'Review blog draft: <post title>',
   'Auto Blogger created a new draft for review and scheduling.\nDuda preview: <duda_publish_url>\nTarget keyword: <keyword>\nQA verdict: SHIP',
   'to_do', 'normal',
   '4b9df73e-bc38-49ef-840e-75fdb0b29c29',
   '<to_do column id>')
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task id from above>', '4b9df73e-bc38-49ef-840e-75fdb0b29c29');
```

(`tasks.assignee_user_id` and the `task_assignees` join row are both populated — the app reads
from `task_assignees` for multi-assignee support, `assignee_user_id` is the legacy single-owner
column. Set both so the task shows up for Mila everywhere.)

---

## Order of operations (Publish step, full sequence)

1. `create_blog_post` in Duda → capture `duda_page_id` + preview URL.
2. `UPDATE blog_topics SET status='published', blog_post_id=...` (or the 4b insert did the
   initial row already — this is the final status flip either way).
3. `INSERT INTO blog_posts (...)` → capture the new `blog_posts.id`.
4. `UPDATE blog_auto_runs SET status='completed', completed_at=now(), articles_generated=1,
   articles_passed_gate=1 WHERE id='<run_id>'`.
5. `INSERT INTO tasks (...)` + `INSERT INTO task_assignees (...)` for Mila.

If the branch stops before step 1 (QA fail, no fuel, no qualifying topic/angle, doctrine
halt), skip straight to closing `blog_auto_runs` as `'failed'` with a specific `error_log` —
steps 2 (topic status), 3, and 5 don't happen for a branch that didn't produce a draft.
