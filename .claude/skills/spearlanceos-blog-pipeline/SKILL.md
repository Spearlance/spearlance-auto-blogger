---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS table schemas, status/enum values, and copy-pasteable SQL for the Auto Blogger's write-back steps (blog_auto_runs, blog_topics, blog_posts, tasks, task_assignees). Read this before writing back to SpearlanceOS from routine.md's Open the run log / Publish / Record the run steps.
---

# SpearlanceOS Blog Pipeline — write-back reference

`routine.md` writes to five tables, in this order per client, per run: open `blog_auto_runs` first,
then (only if a draft ships) `blog_topics`, `blog_posts`, `tasks` + `task_assignees`, then close
`blog_auto_runs` last. Project id **`chikljxwgiskyjsnjelf`**. Everything else in SpearlanceOS is
read-only — never write anywhere else, never call its `blog-*` edge functions.

Reconstructed 2026-08-06 from live `information_schema` introspection and real rows written by prior
successful Auto Blogger runs (this file was missing from the repo though referenced throughout
`routine.md`; regenerate/verify against `information_schema.columns` if schema drift is suspected).

---

## 1. `blog_auto_runs` — one row per client per tick attempt

Open **before** any research or drafting; close (via `UPDATE`) once, at the end of that client's branch,
whether or not a draft shipped.

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, `gen_random_uuid()` default — capture on insert as `<run_id>` |
| `client_id` | uuid | required |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | `'scheduled'` (hourly cron) or `'manual'` (single `client_id` test/dry-run) |
| `status` | text | `'running'` (default, on open) → `'completed'` or `'failed'` (on close) |
| `topics_generated` | int | default 0; 1 if Step 4b generated a new topic this run |
| `articles_generated` | int | default 0; 1 if a draft was created |
| `articles_passed_gate` | int | default 0; 1 if QA verdict was SHIP |
| `articles_flagged` | int | default 0; 1 if a doctrine escalation trigger fired |
| `research_summary` | jsonb | optional — sources checked, angle considered, etc. |
| `completed_at` | timestamptz | set on close |
| `error_log` | text | required when `status='failed'` — the reason the branch stopped |
| `created_at` | timestamptz | default `now()` |

**Open:**
```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

**Close — success:**
```sql
UPDATE public.blog_auto_runs
SET status = 'completed', articles_generated = 1, articles_passed_gate = 1,
    topics_generated = <0_or_1>, completed_at = now()
WHERE id = '<run_id>';
```

**Close — no draft (bar not cleared / no fuel / no strategy / doctrine halt / escalation):**
```sql
UPDATE public.blog_auto_runs
SET status = 'failed', error_log = '<why this branch stopped>', completed_at = now()
WHERE id = '<run_id>';
```

If the branch never gets past due-check/preflight for a given client (e.g. skipped this tick to respect
`POSTS_PER_RUN`), do **not** open a row for it at all — a `blog_auto_runs` row means the client's branch
was actually entered.

### Avoiding redundant research on a re-selected client

Because `selected_days` marks a client due for the *entire* day and the cron is hourly, the same due
client can legitimately be re-entered multiple times in one day (this is intentional — see `routine.md`
Step 0c: existing-draft counts are not a due-ness signal). But re-running a full Tavily/DataforSEO
research pass every single tick when the prior tick already exhaustively searched and found nothing is
pure waste. Before Step 4b's research:

```sql
SELECT id, status, error_log, triggered_at
FROM public.blog_auto_runs
WHERE client_id = '<client_id>' AND status = 'failed'
  AND triggered_at > now() - interval '3 hours'
ORDER BY triggered_at DESC LIMIT 1;
```

If the most recent failed run within the lookback window already concluded "no usable topic / no new
sourceable angle" for the same underlying state (re-verify cheaply: `list_blog_posts` latest post
unchanged, `blog_topics` unused-row count unchanged), skip re-running full research this tick — decline
again citing that prior run's `id` and conclusion. Re-run research if the state genuinely changed (new
Duda post appeared, `blog_topics` backlog was refreshed, or the lookback window has elapsed).

---

## 2. `blog_topics` — the topic queue

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | required |
| `topic_title` | text | required |
| `description` | text | (routine.md's "summary" field for the *selection* query maps to this in some rows — see `summary` below; the two overlap in practice, prefer `summary`) |
| `summary` | text | primary summary field used by real inserts |
| `keywords` | text[] | |
| `avatar_id` | uuid | nullable, rarely set by the routine |
| `priority` | text | free text, default `'medium'` |
| `status` | text | default `'idea'`; observed values: `'idea'`, `'in_progress'`, `'published'` (routine's done-marker) |
| `blog_post_id` | uuid | FK to `blog_posts.id`; observed set on some rows, null on others — both are fine, `status='published'` is the authoritative dedup signal (Step 4a filters on `status`, not this column) |
| `ai_generated` | boolean | default false; set `true` for Step 4b-generated topics |
| `strategy_batch_id` | uuid | nullable |
| `suggested_publish_date` | date | |
| `category` | text | matches a `content_mix` key, e.g. `how_to`, `best_practices`, `industry_news` |
| `created_at` | timestamptz | default `now()` |

**Step 4b — generate + save a new topic:**
```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>', true, 'in_progress', current_date)
RETURNING id;
```

**Step 4a — mark a used calendar topic in progress:**
```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**Publish step — mark the topic done (stops next tick reselecting it):**
```sql
UPDATE public.blog_topics
SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

---

## 3. `blog_posts` — one row per drafted article (Mila's draft/schedule/publish record)

Always insert with `status = 'draft'` — never `'scheduled'` or `'published'`; that transition is Mila's
call, made in her own tooling, not by the routine.

| column | type | notes |
|---|---|---|
| `id` | uuid | PK |
| `client_id` | uuid | required |
| `title` | text | required |
| `slug` | text | required — the Duda post path/slug |
| `meta_description` | text | |
| `content` | text | full HTML body sent to Duda |
| `excerpt` | text | optional |
| `focus_keyword` | text | primary target keyword |
| `target_keywords` | text[] | full keyword set |
| `topic_category` | text | matches `blog_topics.category` / `content_mix` key |
| `featured_image_url` / `featured_image_alt` | text | |
| `status` | text | **always `'draft'`** |
| `duda_page_id` | text | the Duda `create_blog_post` response's post id |
| `duda_publish_url` | text | the Duda preview/live URL |
| `ai_model` | text | default `'claude-sonnet-4-5'` |
| `topic_id` | uuid | FK to `blog_topics.id` |
| `auto_run_id` | uuid | FK to `blog_auto_runs.id` (this run's `<run_id>`) |
| `quality_scores` | jsonb | the full doctrine QA verdict (see shape below) |
| `revision_count` | int | default 0 |
| `created_at` / `updated_at` | timestamptz | default `now()` |

`quality_scores` jsonb shape (flexible — match this spirit, exact keys have varied run to run):
```json
{
  "verdict": "SHIP",
  "word_count": 1615,
  "h2_count": 7,
  "internal_links": 10,
  "external_sources": 3,
  "sources": ["https://...", "https://..."],
  "checks": {
    "title": "PASS", "description": "PASS", "headings": "PASS - no H1 in body",
    "content": "PASS - ...", "interlinking": "PASS - ...", "schema": "PASS - Article auto-applied; FAQ JSON-LD needs manual Duda Custom HTML add",
    "intent": "PASS - remains a blog post", "escalation": "none"
  }
}
```

**Insert (after `create_blog_post` succeeds in Duda):**
```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, target_keywords, topic_category,
   featured_image_url, status, duda_page_id, duda_publish_url, topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta_description>', '<html_content>',
   ARRAY['<kw1>','<kw2>'], '<category>', '<image_url>', 'draft',
   '<duda_post_id>', '<duda_preview_url>', '<topic_id>', '<run_id>', '<quality_scores_jsonb>'::jsonb)
RETURNING id;
```

---

## 4. `tasks` + `task_assignees` — one review task for Mila per draft

Only insert when a draft was actually created. `tasks.status` and `tasks.priority` are **Postgres enums**,
not free text:

- `task_status`: `to_do` | `in_progress` | `done` — always insert `'to_do'`.
- `task_priority`: `low` | `normal` | `high` | `urgent` — default `'normal'` is fine unless the doctrine
  QA flagged something worth prioritizing.

`tasks.column_id` must be a real `task_columns.id` for that client's board — look up the client's `to_do`
column (not just the enum status) before inserting:
```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do' LIMIT 1;
```

Look up Mila's `profiles.id` by email (do this once per run, or cache):
```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

**Insert the task:**
```sql
INSERT INTO public.tasks
  (client_id, title, description, status, priority, column_id, assignee_user_id, related_asset_ids)
VALUES
  ('<client_id>',
   'Review draft: <short title> (<client name>)',
   'Auto Blogger created a new draft for <client name>. QA verdict: <SHIP details>. Duda preview: <url> (unpublished). Follow-up: add FAQ JSON-LD manually via Duda Settings -> Custom HTML if the post has an FAQ section.',
   'to_do', 'normal', '<to_do_column_id>', '<mila_profile_id>', ARRAY['<blog_posts.id>']::uuid[])
RETURNING id;
```

**Insert the assignee link (both `tasks.assignee_user_id` and a `task_assignees` row are set in practice — keep them consistent):**
```sql
INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_profile_id>');
```

---

## 5. Full per-client write sequence (happy path)

```
1. INSERT blog_auto_runs (status='running')                 → <run_id>
2. (Step 4a/4b) INSERT or UPDATE blog_topics (status='in_progress') → <topic_id>
3. create_blog_post (Duda)                                  → <duda_page_id>, <duda_preview_url>
4. INSERT blog_posts (status='draft', topic_id, auto_run_id, quality_scores) → <blog_post_id>
5. UPDATE blog_topics SET status='published', blog_post_id=<blog_post_id> WHERE id=<topic_id>
6. INSERT tasks (status='to_do', column_id=<to_do_column_id for client>, assignee_user_id=<mila_id>, related_asset_ids=[<blog_post_id>])
   → <task_id>
7. INSERT task_assignees (task_id=<task_id>, user_id=<mila_id>)
8. UPDATE blog_auto_runs SET status='completed', articles_generated=1, articles_passed_gate=1, completed_at=now() WHERE id=<run_id>
```

No-draft path: skip 2–7 (unless a topic was marked `in_progress` and then abandoned — leave it
`in_progress` rather than reverting, so the next tick's Step 4a naturally excludes it too, or explicitly
revert to `'idea'` if the topic itself was the problem, e.g. genuinely off-season); always still close
`blog_auto_runs` at step 8 with `status='failed'` and a populated `error_log`.
