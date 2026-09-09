# SpearlanceOS Blog Pipeline

Exact table schemas, status values, and copy-pasteable SQL for the Auto Blogger's write-back to
SpearlanceOS. Project id **`chikljxwgiskyjsnjelf`** (production — see `supabase-prod-guard`; this
routine's writes here are its designed, standing, scheduled function, not an ad hoc write).

> Reconstructed 2026-09-09 from the live schema and 866+ real `blog_auto_runs` rows after this file
> was found missing (referenced by the routine but never committed). If this file and `routine.md`
> ever disagree again, trust the live schema (`information_schema.columns`) and recent rows over prose.

Every post is created in Duda as an **unpublished draft**. `auto_blog_mode` (`queue`/`auto_publish`) is
enrollment-only; it does not change publish behavior. **Never call `publish_blog_post`.**

## Guard: one draft per client per day

There is **no DB constraint** enforcing this — no unique index on `(client_id, date(created_at))` in
`blog_posts`. It's a convention this routine must enforce itself: before drafting, check

```sql
SELECT id, title, status, created_at FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at >= current_date;
```

If a row exists, **do not draft again this tick** — log a `blog_auto_runs` row with `status='failed'`
and move on. This guard exists because the cron has historically fired more often than hourly; without
it, a due client accumulates duplicate drafts every tick. Confirm the cron's actual firing cadence with
Garrett if `blog_auto_runs.triggered_at` timestamps for one day are denser than hourly.

## `blog_auto_runs` — one row per client processed per tick

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, default `gen_random_uuid()` |
| `client_id` | uuid | NOT NULL |
| `trigger_type` | text | `'scheduled'` for cron ticks, `'manual'` for single-client test/dry-run |
| `status` | text | `'running'` (default) → `'completed'` (draft created) or `'failed'` (branch stopped) |
| `topics_generated` | int | default 0 |
| `articles_generated` | int | default 0 |
| `articles_passed_gate` | int | default 0 |
| `articles_flagged` | int | default 0 |
| `research_summary` | jsonb | optional |
| `error_log` | text | required when `status='failed'`; be concise, don't repeat prior ticks' full history — reference it |
| `completed_at` | timestamptz | set when closing the row |

Open at the start of a client's processing (`status='running'`), close it once — `completed`
(with the article/topic counts) or `failed` (with `error_log`) — before moving to the next client.

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status, error_log, completed_at)
VALUES ('<client_id>', 'scheduled', 'failed', '<reason>', now())
RETURNING id;
```

## `blog_posts` — one row per drafted article (Mila's review/schedule queue)

Key columns actually used by this pipeline (full schema has more; leave the rest null/default):

| column | notes |
|---|---|
| `client_id` | uuid, NOT NULL |
| `title` | NOT NULL |
| `slug` | NOT NULL — derive from title |
| `meta_description`, `content`, `excerpt`, `focus_keyword` | |
| `target_keywords` | text[] |
| `topic_category` | matches `blog_topics.category` |
| `featured_image_url`, `featured_image_alt` | |
| `status` | **always `'draft'`** — never `'scheduled'` or `'published'`, that's Mila's call |
| `duda_page_id` | the Duda post id from `create_blog_post`'s response |
| `topic_id` | FK → `blog_topics.id` (`ON DELETE SET NULL`) |
| `auto_run_id` | FK → `blog_auto_runs.id` (`ON DELETE SET NULL`) |
| `quality_scores` | jsonb — the doctrine QA verdict (see shape below) |
| `created_at` | default `now()` — this is what the one-draft-per-day guard checks |

`quality_scores` shape (observed convention, not enforced — keep it close to this):
```json
{
  "verdict": "SHIP",
  "word_count": 1450,
  "h2_count": 8,
  "internal_links": 9,
  "sources_cited": ["https://...", "https://..."],
  "doctrine_check": { "title": "PASS", "intent": "PASS", "schema": "PASS - ...", "content": "PASS - ...",
                       "headings": "PASS - ...", "sourcing": "PASS - ...", "interlinking": "PASS - ...",
                       "description": "PASS", "escalation": "none" }
}
```

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, target_keywords,
   topic_category, featured_image_url, featured_image_alt, status, duda_page_id,
   topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta_description>', '<content excerpt or full html>',
   '<focus_keyword>', ARRAY['<kw1>','<kw2>'], '<category>', '<image_url>', '<alt_text>',
   'draft', '<duda_post_id>', '<topic_id>', '<auto_run_id>', '<quality_scores jsonb>'::jsonb)
RETURNING id;
```

## `blog_topics` — write-back after use

```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```
(`'published'` means "turned into a post," not "live" — the table's existing convention.)

## `tasks` + `task_assignees` — the review task for Mila

Only when a draft was actually created. `tasks.status` and `tasks.priority` are Postgres enums
(`task_status`, `task_priority`); `'to_do'` and `'normal'` are safe defaults and match observed rows.

Mila: `id = '4b9df73e-bc38-49ef-840e-75fdb0b29c29'`, `email = 'mila@spearlance.com'` (verify by email,
not by hardcoding this id, in case it ever changes — `SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com'`).

Look up the client's "To Do" column:
```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do' LIMIT 1;
```

```sql
WITH new_task AS (
  INSERT INTO public.tasks (client_id, title, status, column_id, external_source, external_ref)
  VALUES ('<client_id>', 'Review draft: <post title> (<client name>)', 'to_do', '<column_id>',
          'auto_blogger', '<blog_posts.id>')
  RETURNING id
)
INSERT INTO public.task_assignees (task_id, user_id)
SELECT id, '4b9df73e-bc38-49ef-840e-75fdb0b29c29' FROM new_task;
```

`external_ref` is conventionally the new `blog_posts.id` (uuid). A few historical rows used the Duda
post id instead — either is fine for Mila's purposes, but prefer `blog_posts.id` since it round-trips
back into SpearlanceOS.

## Full close-out order, per client with a draft

1. `UPDATE blog_topics SET status='published'`
2. `INSERT INTO blog_posts (...)` → get its `id`
3. `UPDATE blog_auto_runs SET status='completed', articles_generated=1, articles_passed_gate=1, completed_at=now() WHERE id='<run_id>'`
4. `INSERT INTO tasks (...)` + `INSERT INTO task_assignees (...)`

If the branch is stopped instead (QA fail, guard block, no fuel, no strategy row, doctrine halt): just
close `blog_auto_runs` with `status='failed'` and a concise `error_log`. Steps 1, 2, and 4 don't happen.
