---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase schema, status values, and copy-pasteable SQL for the Auto Blogger's write-back — blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees. Read this before writing back to SpearlanceOS in the Publish and run-log steps, and read it BEFORE the due-check (Step 0c) so the one-draft-per-day guardrail is honored instead of discovered after a Duda draft already exists.
---

# SpearlanceOS Blog Pipeline — write-back reference

The routine reads most of SpearlanceOS and writes to exactly five tables: `blog_topics`,
`blog_posts`, `blog_auto_runs`, `tasks`, `task_assignees`. This skill gives the real, introspected
column names and constraints so those writes succeed on the first try — no guessing at
`information_schema` mid-run, and no orphaned Duda drafts from a write-back that fails after
the content already exists on the client's site.

Verified against the live `chikljxwgiskyjsnjelf` project. If a write to any of these tables ever
errors as "column does not exist," re-run the `information_schema.columns` query below rather than
guessing — the schema may have moved since this was written.

```sql
SELECT table_name, column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name IN ('blog_topics','blog_posts','blog_auto_runs','tasks','task_assignees','task_columns')
ORDER BY table_name, ordinal_position;
```

---

## Read this BEFORE the due-check, not just before publishing

`blog_posts` has a hard DB guardrail:

```sql
CREATE TRIGGER trg_one_blog_draft_per_client_per_day
  BEFORE INSERT ON public.blog_posts
  FOR EACH ROW EXECUTE FUNCTION guard_one_blog_draft_per_client_per_day();
-- function body:
--   IF EXISTS (SELECT 1 FROM blog_posts WHERE client_id = NEW.client_id AND created_at::date = now()::date)
--   THEN RAISE EXCEPTION 'Auto Blogger guardrail: client already has a blog_posts draft created today...';
```

**One `blog_posts` row per `client_id` per calendar day (UTC), full stop.** A due client that
already has a `blog_posts` row dated today is not actually eligible this tick, even though it
still shows up in the calendar due-check (Step 0c only checks `selected_days`, not this).

Because of this, **before doing any research or drafting for a candidate client**, run:

```sql
SELECT id, created_at FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at::date = current_date;
```

If a row comes back, that client is **not** actually eligible this tick — move to the next
candidate, or end the run if none remain. Checking this *after* `create_blog_post` has already
run against Duda is how orphaned drafts happen: the post exists live on the client's site, but
the `blog_posts` insert is rejected, so it never gets a `blog_posts` row or a review `task` and
Mila never sees it. If you discover this collision after a Duda draft already exists, don't leave
it silently orphaned — either delete the Duda draft (if `delete_blog_post` is available and safe
to call unattended) or, at minimum, flag it explicitly in the end-of-run report with the Duda
post ID and site, so it can be manually reconciled.

---

## `blog_auto_runs` — one row per client you process, opened first, closed last

```
id               uuid PK, default gen_random_uuid()
client_id        uuid NOT NULL
triggered_at     timestamptz NOT NULL, default now()
trigger_type     text NOT NULL   -- CHECK: 'scheduled' | 'manual'
status           text NOT NULL, default 'running'   -- CHECK: 'running' | 'completed' | 'failed'
topics_generated     int NOT NULL, default 0
articles_generated   int NOT NULL, default 0
articles_passed_gate int NOT NULL, default 0
articles_flagged     int NOT NULL, default 0
research_summary jsonb
completed_at     timestamptz
error_log        text
created_at       timestamptz NOT NULL, default now()
```

Open at the start of a client's processing (after it's confirmed actually eligible per the
same-day check above):

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Use `trigger_type = 'manual'` only for an ad hoc single-`client_id` invocation. Close the same
row once, at the end of that client's branch — whether or not a draft was created:

```sql
-- success (a draft was created and QA-passed):
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = 1, articles_generated = 1, articles_passed_gate = 1,
    research_summary = '<jsonb of sources/keywords/links used>'::jsonb
WHERE id = '<run_id>';

-- stopped branch (QA fail / no fuel / no strategy / doctrine halt / escalation):
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, specific and falsifiable>'
WHERE id = '<run_id>';
```

Every opened row gets closed one way or the other before moving to the next client. Do not leave
a row `status = 'running'` at the end of the run.

---

## `blog_posts` — the actual review/schedule record Mila works from

```
id                     uuid PK, default gen_random_uuid()
client_id              uuid NOT NULL
title                  text NOT NULL
slug                   text NOT NULL
meta_description       text
content                text
excerpt                text
focus_keyword          text
seo_score              numeric
readability_score      numeric
avatar_id              uuid
topic_category         text
target_keywords        text[]
featured_image_url     text
featured_image_alt     text
body_images            jsonb
status                 text, default 'draft'
scheduled_for          timestamptz
published_at           timestamptz
duda_page_id           text
duda_publish_url       text
ai_model               text, default 'claude-sonnet-4-5'
generation_prompt      text
generation_metadata    jsonb
created_by             uuid
created_at             timestamptz, default now()
updated_at             timestamptz, default now()
last_edited_by         uuid
topic_id               uuid
scheduled_publish_date date
images_generated       boolean, default false
auto_run_id            uuid
quality_scores         jsonb
rejection_reason       text
revision_count         int NOT NULL, default 0
```

Insert **after** the Duda draft is created (so `duda_page_id` is real), and **always** as
`status = 'draft'` — never `'scheduled'` or `'published'`; that transition is Mila's call, made
from her own review UI, not the routine's:

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt,
  focus_keyword, target_keywords, topic_category,
  featured_image_url, featured_image_alt,
  status, duda_page_id, duda_publish_url,
  topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<url-safe-slug>', '<meta description>', '<full HTML content>', '<short excerpt>',
  '<primary keyword>', ARRAY['<kw1>','<kw2>']::text[], '<content_mix category>',
  '<thumbnail/main image url>', '<alt text>',
  'draft', '<duda post id>', '<duda preview/editor url>',
  '<topic_id>', '<run_id>', '<jsonb QA verdict + rule-by-rule notes>'::jsonb
)
RETURNING id;
```

`quality_scores` should carry the doctrine QA verdict (SHIP + rule-by-rule pass/fail notes), not
just a number — it's the audit trail for why this draft was allowed to ship.

---

## `blog_topics` — mark the topic done so the next tick doesn't reselect it

Schema (unchanged from what Step 4 already expects — `summary`, not `description`, is the field
the due-topic query reads):

```
id                     uuid PK
client_id              uuid NOT NULL
topic_title            text NOT NULL
description            text
summary                text
keywords               text[]
avatar_id              uuid
priority               text, default 'medium'
status                 text, default 'idea'   -- no CHECK constraint; convention: idea -> in_progress -> published
blog_post_id           uuid
ai_generated           boolean, default false
strategy_batch_id      uuid
suggested_publish_date date
category               text
created_at             timestamptz, default now()
```

After the `blog_posts` row is created, mark the topic done and link it:

```sql
UPDATE public.blog_topics
SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

`status = 'published'` here means "turned into a post" (per the table's existing convention),
not "live on the internet" — don't let the name confuse you into gating on it differently.
**Set `blog_post_id`, not just `status`** — a topic marked `published` with a NULL `blog_post_id`
looks orphaned later and breaks traceability from topic → draft.

---

## `tasks` + `task_assignees` — one review task for Mila, only when a draft was created

`task_columns` is **per-client** (its own kanban board per client), so the "to do" column's `id`
must be looked up per client, not assumed global:

```sql
SELECT id FROM public.task_columns
WHERE client_id = '<client_id>' AND key = 'to_do'
LIMIT 1;
```

Look up Mila's user id by email (don't hardcode it — confirm it's still current each run):

```sql
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';
```

`tasks.status` / `tasks.priority` are real Postgres enums, not free text:

```
task_status:   to_do | in_progress | done
task_priority: low | normal | high | urgent
```

```
tasks columns actually used here:
  id, client_id, title, description, status (default 'to_do'), priority (default 'normal'),
  assignee_user_id, creator_user_id, due_date, column_id, created_at, updated_at
```

Insert the task, set `column_id` to that client's `to_do` column, and set `assignee_user_id`
directly (the app's kanban board reads this):

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES (
  '<client_id>',
  'Review Auto Blogger draft: <post title>',
  'Auto Blogger created a draft for review. Duda: <duda preview/editor url>. Keyword: <primary keyword>. QA: SHIP.',
  'to_do', 'normal', '<mila_user_id>', '<to_do column_id for this client>'
)
RETURNING id;
```

Then also add the `task_assignees` link row — the app's multi-assignee model reads from this
table, so `assignee_user_id` alone is not sufficient for Mila to reliably see it as "assigned to
her":

```sql
INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_user_id>');
```

---

## Write order for a successful client (summary)

1. `blog_auto_runs` insert (`status='running'`) — before any research spend.
2. Research, draft, QA.
3. `create_blog_post` in Duda → capture `duda_page_id` + preview URL.
4. `blog_posts` insert (`status='draft'`, linking `topic_id` + `auto_run_id` + `duda_page_id`).
5. `blog_topics` update (`status='published'`, `blog_post_id` set).
6. `tasks` insert + `task_assignees` insert (Mila).
7. `blog_auto_runs` update (`status='completed'`, counts + `research_summary`).

If the branch stops before step 3 (no fuel, no strategy, QA fail, doctrine halt), skip straight
to closing `blog_auto_runs` as `status='failed'` with a specific `error_log` — steps 4–6 never
run because there's nothing to record.
