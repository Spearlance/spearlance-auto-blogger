---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the Auto Blogger's SpearlanceOS write-back — blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees. Read by routine.md at "Open the run log" and "Publish → write back to SpearlanceOS." Project chikljxwgiskyjsnjelf; supabase-prod-guard's "ask first" rule does not re-litigate these specific, pre-authorized, narrowly-scoped writes — they are the routine's designed job — but never write outside this scope.
---

# SpearlanceOS Blog Pipeline — write-back reference

Schema confirmed against `information_schema.columns` / `pg_enum` on project `chikljxwgiskyjsnjelf`,
2026-08-24. Re-verify with `information_schema` (read-only, always allowed) if a write errors on a
missing/renamed column — do not guess a second time, look it up.

## Hard guardrail: one `blog_posts` draft per client per day

A DB trigger enforces this — it is not optional and routine.md's Step 0c due-check does **not**
override it:

```sql
CREATE TRIGGER trg_one_blog_draft_per_client_per_day BEFORE INSERT ON public.blog_posts
  FOR EACH ROW EXECUTE FUNCTION guard_one_blog_draft_per_client_per_day();
-- rejects INSERT if a blog_posts row already exists for NEW.client_id with created_at::date = now()::date
```

**Before doing any research or drafting for a client, check this first:**

```sql
SELECT id FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at::date = current_date;
```

If a row exists, **do not** run research/draft/QA for that client this tick — the eventual
`blog_posts` INSERT will be rejected, wasting a full research+draft+QA cycle and leaving an
orphaned, untracked **unpublished** draft in Duda (harmless but messy — nothing publishes live, see
CLAUDE.md Go-live posture). Treat this the same as "not due" and move to the next candidate, or end
the run if no other due client qualifies. This directly narrows routine.md Step 0c/0d: "due" (calendar)
and "eligible to draft today" (this guardrail) are two separate checks — both must pass.

## 1. Open the run log — `blog_auto_runs`

Insert **before** any research/drafting, one row per client actually attempted this tick:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type`: `'scheduled'` for the normal hourly cron; `'manual'` only for an ad hoc single-`client_id` invocation.
- Keep the returned `id` as `<run_id>` for the rest of this client's processing.

Close it out once, at the end of this client's branch, whether or not a draft was created:

```sql
-- success path
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = <0 or 1>, articles_generated = <0 or 1>,
    articles_passed_gate = <0 or 1>, articles_flagged = <0 or 1>,
    research_summary = '<jsonb, optional: sources/keywords used>'::jsonb
WHERE id = '<run_id>';

-- stopped-branch path (QA fail, no fuel, no strategy row, doctrine halt, guardrail block, escalation)
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, plainly>'
WHERE id = '<run_id>';
```

`status` is free-text (`'running' | 'completed' | 'failed'` by convention, not an enum — no CHECK
constraint). `topics_generated` / `articles_generated` / `articles_passed_gate` / `articles_flagged`
are `integer NOT NULL DEFAULT 0`.

## 2. Topic write-back — `blog_topics`

**Generating a new topic (Step 4b):**

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>']::text[], '<category>',
   true, 'in_progress', current_date)
RETURNING id;
```

**Using an existing calendar topic (Step 4a)** — mark it claimed before drafting:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**After the draft is created in Duda** — mark the topic done (this means "turned into a post," not
"live"):

```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>' WHERE id = '<topic_id>';
```

`status` values in use: `'idea'` (default) → `'in_progress'` → `'published'`. Free-text, no enum.

## 3. Article record — `blog_posts`

One row per drafted article, **always `status = 'draft'`** (never `'scheduled'` or `'published'` —
that is Mila's call). Only the columns routine.md actually needs are listed; the table has many more
optional columns (see full list below) — fill what you have, leave the rest null.

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, topic_id, auto_run_id,
   topic_category, target_keywords, status, duda_page_id, duda_publish_url, quality_scores)
VALUES
  ('<client_id>', '<title>', '<url-slug>', '<meta description>', '<html or plain content copy>',
   '<topic_id>', '<run_id>', '<category>', ARRAY['<kw1>','<kw2>']::text[], 'draft',
   '<duda blog post id>', '<duda preview/editor url>',
   '<jsonb: {"doctrine_verdict":"SHIP","checks":{...},"sources":[...]}>'::jsonb)
RETURNING id;
```

Required NOT NULL columns: `client_id`, `title`, `slug`. Everything else above is nullable but
expected by the routine. Full column list (`information_schema`, 2026-08-24): `id, client_id, title,
slug, meta_description, content, excerpt, focus_keyword, seo_score, readability_score, avatar_id,
topic_category, target_keywords, featured_image_url, featured_image_alt, body_images, status,
scheduled_for, published_at, duda_page_id, duda_publish_url, ai_model, generation_prompt,
generation_metadata, created_by, created_at, updated_at, last_edited_by, topic_id,
scheduled_publish_date, images_generated, auto_run_id, quality_scores, rejection_reason,
revision_count`. `status` defaults to `'draft'`; no enum/CHECK — but never write anything but
`'draft'` per the routine's publishing posture.

**This INSERT is the one the one-draft-per-day trigger guards.** If it raises
`P0001 Auto Blogger guardrail: ...`, that client was already served today — close the run log as
`'failed'` with that message as `error_log` and move on; do not retry.

## 4. Review task — `tasks` + `task_assignees`

Only when a draft was actually created. Look up Mila's `profiles.id` by email, and the client's
`'to_do'` column id from `task_columns` (per-client row, not global):

```sql
-- Mila's user id (profiles.email, not auth.users)
SELECT id FROM public.profiles WHERE email = '<mila's email>';

-- this client's To Do column
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';

INSERT INTO public.tasks (client_id, title, description, status, priority, column_id, creator_user_id)
VALUES ('<client_id>', 'Review blog draft: <post title>',
        'Auto Blogger created a draft for review. Duda: <preview url>. Keyword: <keyword>. Sources: <url list>.',
        'to_do', 'normal', '<to_do column_id>', NULL)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila profile id>');
```

`tasks.status` is enum `task_status`: `to_do | in_progress | done`. `tasks.priority` is enum
`task_priority`: `low | normal | high | urgent`. `task_columns.key` values seen in practice: `to_do`,
`in_progress`, `done` (always present, `is_default = true`), plus optional client-specific extras
(e.g. `paused`, `for_review`). Always resolve `column_id` per-client — do not hardcode an id from one
client onto another.

## Scope reminder

Write **only** to `blog_topics`, `blog_posts`, `blog_auto_runs`, `tasks`, `task_assignees` — nothing
else in SpearlanceOS, and never its `blog-*` edge functions. This is the routine's entire, standing,
pre-authorized write surface (see CLAUDE.md); it does not extend to schema changes, other tables, or
ad hoc fixes — those still need Garrett's explicit yes per `supabase-prod-guard`.
