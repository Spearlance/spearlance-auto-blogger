---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, enum values, and copy-pasteable SQL for the Auto Blogger's write-back into SpearlanceOS (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read before the "Open the run log" step and before the Publish step of routine.md, every run — this is the file routine.md/CLAUDE.md refer to for that write-back.
---

# SpearlanceOS Blog Pipeline — write-back reference

Project id: **`chikljxwgiskyjsnjelf`**. Schema below was captured by live introspection
(`information_schema.columns`, `pg_constraint`, `pg_enum`) on **2026-09-27** — reconfirm with the same
queries if a write ever fails on an unknown column/constraint, since this file can drift from the live
schema.

Only write to the five things below. Everything else in SpearlanceOS is read-only for this routine, and
its `blog-*` edge functions are never called.

---

## 0. Known live DB behavior not documented elsewhere in this repo

- **`trg_one_blog_draft_per_client_per_day`** — a `BEFORE INSERT` trigger on `public.blog_posts`
  (function `guard_one_blog_draft_per_client_per_day`) rejects a second `blog_posts` insert for the same
  `client_id` on the same day. `routine.md` Step 0c describes the due-check as calendar-only with no
  same-day-posting gate; in practice this DB trigger is a second, silent gate. If an insert in Step
  "Publish" below fails, check whether the due client already has a `blog_posts` row created today
  before treating it as a real error — it likely means this tick is a correct no-op (a draft already
  exists for today), not a failure. Query first:

  ```sql
  SELECT id, title, created_at FROM public.blog_posts
  WHERE client_id = '<client_id>' AND created_at::date = current_date;
  ```

  If a row already exists for today, do not attempt the insert; log the run as `completed` with 0
  articles (or skip logging a fresh run row at all if you catch this before opening one) and move to the
  next due client.

---

## 1. `blog_auto_runs` — one row per client per attempt

| column | type | notes |
|---|---|---|
| `id` | uuid, default `gen_random_uuid()` | PK, returned on insert — keep as `<run_id>` |
| `client_id` | uuid, **NOT NULL** | |
| `triggered_at` | timestamptz, default `now()` | |
| `trigger_type` | text, **NOT NULL** | CHECK: `'scheduled'` \| `'manual'` |
| `status` | text, **NOT NULL**, default `'running'` | CHECK: `'running'` \| `'completed'` \| `'failed'` |
| `topics_generated` | int, default `0` | |
| `articles_generated` | int, default `0` | |
| `articles_passed_gate` | int, default `0` | |
| `articles_flagged` | int, default `0` | |
| `research_summary` | jsonb, nullable | free-form; put source URLs / keyword research notes here |
| `completed_at` | timestamptz, nullable | set when closing the row |
| `error_log` | text, nullable | set when `status = 'failed'`, or to record why a branch stopped |
| `created_at` | timestamptz, default `now()` | |

**Open (top of per-client processing):**

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Use `trigger_type = 'manual'` only for an ad hoc single-`client_id` invocation.

**Close — success (a draft was created):**

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',
    topics_generated = <0 or 1>,
    articles_generated = 1,
    articles_passed_gate = 1,
    articles_flagged = 0,
    research_summary = '<jsonb: source URLs, keyword/intent notes>'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';
```

**Close — stopped branch (QA fail / no fuel / no strategy / doctrine halt / escalation / already-posted-today):**

```sql
UPDATE public.blog_auto_runs
SET status = 'failed',
    error_log = '<why: QA fail reason, missing story, no blog_content_strategy row, doctrine file unreadable, escalation trigger, etc.>',
    completed_at = now()
WHERE id = '<run_id>';
```

(`'failed'` is the only non-`'completed'` terminal value — it covers every stopped-branch reason, not
just literal errors. There is no `'skipped'`/`'no_content'` status; don't invent one, the CHECK
constraint will reject it.)

---

## 2. `blog_topics` — the topic queue

| column | type | notes |
|---|---|---|
| `id` | uuid, default `gen_random_uuid()` | |
| `client_id` | uuid, **NOT NULL** | |
| `topic_title` | text, **NOT NULL** | |
| `description` | text, nullable | |
| `summary` | text, nullable | used in practice more than `description` — put the working summary here |
| `keywords` | text[], nullable | |
| `category` | text, nullable, free-text | observed values include `how_to`, `best_practices`,
  `legal_regulatory` / `legal_regulatory_update` (inconsistent casing exists in live data — pick one
  going forward, don't worry about normalizing old rows) — align to `blog_content_strategy.content_mix`
  keys where possible |
| `priority` | text, default `'medium'` | free-text, not an enum; `'high'`/`'medium'`/`'low'` by convention |
| `status` | text, default `'idea'` | not an enum; live values in use: `'idea'`, `'in_progress'`,
  `'published'` (see routine.md's convention: `'published'` means "turned into a post", not "live") |
| `blog_post_id` | uuid, nullable | set once a `blog_posts` row exists for this topic |
| `ai_generated` | boolean, default `false` | |
| `created_at` | timestamptz, default `now()` | |
| `strategy_batch_id` | uuid, nullable | leave null unless generating as part of a batch |
| `suggested_publish_date` | date, nullable | |
| `avatar_id` | uuid, nullable | leave null — `avatars` is unreliable per CLAUDE.md |

**4a — mark a selected calendar topic in progress:**

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**4b — insert a generated topic:**

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>']::text[], '<category>', true,
   'in_progress', current_date)
RETURNING id;
```

**Publish step — mark done:**

```sql
UPDATE public.blog_topics
SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

---

## 3. `blog_posts` — one row per drafted article (Mila's draft/schedule/publish queue)

Required (`NOT NULL`, no default): `client_id`, `title`, `slug`. Everything else below is nullable.

| column | type | notes |
|---|---|---|
| `client_id` | uuid | |
| `title` | text | the Duda post title |
| `slug` | text | kebab-case; observed convention: lowercased title, non-alphanumerics → `-`, `&`→`amp` literally kept (see live data) — simplest safe approach: lowercase, replace runs of non `[a-z0-9]` with `-`, trim `-` |
| `meta_description` | text | |
| `content` | text | full HTML body as sent to Duda |
| `excerpt` | text | |
| `focus_keyword` | text | |
| `seo_score` / `readability_score` | numeric | optional, leave null if not computed |
| `avatar_id` | uuid | leave null (see above) |
| `topic_category` | text | mirror `blog_topics.category` |
| `target_keywords` | text[] | |
| `featured_image_url` / `featured_image_alt` | text | Duda stock image + alt text |
| `body_images` | jsonb | optional |
| `status` | text, default `'draft'` | **always `'draft'`** — never `'scheduled'`/`'published'` from this routine, that is Mila's call |
| `scheduled_for` / `published_at` / `scheduled_publish_date` | — | leave null — Mila sets these |
| `duda_page_id` | text | the Duda post id returned by `create_blog_post` |
| `duda_publish_url` | text | the Duda preview/editor URL |
| `ai_model` | text, default `'claude-sonnet-4-5'` | set to whatever model is actually running the routine |
| `generation_prompt` / `generation_metadata` | text / jsonb | optional provenance |
| `topic_id` | uuid | FK to `blog_topics.id` |
| `auto_run_id` | uuid | FK to `blog_auto_runs.id` (the `<run_id>` opened this client) |
| `quality_scores` | jsonb | the doctrine QA verdict — see shape below |
| `images_generated` | boolean, default `false` | leave `false` (images deferred per CLAUDE.md) |
| `rejection_reason` | text | only relevant if you insert a non-shipped draft for record-keeping (routine.md doesn't currently do this — most stopped branches don't reach this table at all) |
| `revision_count` | int, default `0` | |
| `created_by` / `last_edited_by` | uuid | leave null (no human author in this flow) |

`quality_scores` observed shape (free-form jsonb, keep it useful over exact-matching this):

```json
{
  "verdict": "SHIP",
  "checks": { "sourcing": "PASS - ...", "citations": "PASS - ...", "interlinking": "PASS - ...", "schema": "PASS - ..." },
  "sources": ["https://...", "https://..."],
  "h2_count": 7,
  "word_count": 1650,
  "internal_links": 10,
  "follow_ups": ["Add FAQ JSON-LD manually via Duda Settings -> Custom HTML", "..."]
}
```

**Insert (Publish step), after `create_blog_post` returns its Duda id/URL:**

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status,
   duda_page_id, duda_publish_url, ai_model, topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta description>', '<html content>', '<focus keyword>',
   '<category>', ARRAY['<kw1>','<kw2>']::text[], '<image url>', '<alt text>', 'draft',
   '<duda_page_id>', '<duda_preview_or_editor_url>', '<model name>', '<topic_id>', '<run_id>',
   '<quality_scores jsonb>'::jsonb)
RETURNING id;
```

Before this insert, always check the one-draft-per-day guard in section 0 — an insert for a client that
already has a `blog_posts` row `created_at::date = current_date` will be rejected by the DB trigger.

---

## 4. `tasks` + `task_assignees` — the review task for Mila

`tasks.status` and `tasks.priority` are **real Postgres enums** (not free text):

- `task_status`: `to_do` | `in_progress` | `blocked` | `done` | `cancelled`
- `task_priority`: `low` | `normal` | `high` | `urgent`

Mila's identity (`profiles`, looked up by email, not hardcoded — reconfirm if it ever comes back empty):

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

The task's `column_id` must be that **client's own** `'to_do'` `task_columns` row (one exists per
client, different `id` each time — never reuse another client's column id):

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

**Insert the task**, then assign it:

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, column_id)
VALUES (
  '<client_id>',
  'Review draft: <post title>',
  'Auto Blogger created this draft — <one-line summary, keyword/intent, Duda preview link>.',
  'to_do',
  'normal',
  (SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do')
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', (SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com'));
```

Only do this when a draft was actually created (`blog_posts` insert succeeded). Title wording doesn't
need to match exactly across clients — live data shows several phrasings (`Review draft: ...`,
`Review Auto Blogger draft: ...`, `Review blog draft: ...`); any of these is fine, pick one and be
consistent within a run.

---

## 5. Order of operations for one client (ties routine.md's Publish step to the above)

1. `blog_auto_runs` insert → `<run_id>` (done at the very top of the client's processing, before
   research — see routine.md).
2. ... research, draft, QA ...
3. Before inserting into `blog_posts`, check section 0's one-draft-per-day guard.
4. Duda `create_blog_post` → get `duda_page_id` + preview/editor URL.
5. `blog_topics` update (`status='published'`, `blog_post_id=<new id>` or mark `in_progress` earlier if
   using a calendar topic — see section 2).
6. `blog_posts` insert (section 3), using `<run_id>` and the topic id.
7. `tasks` + `task_assignees` insert (section 4) — only if step 6 succeeded.
8. `blog_auto_runs` close (section 1) — `'completed'` if step 6 succeeded, `'failed'` with `error_log`
   otherwise, whether the branch stopped here or earlier.
