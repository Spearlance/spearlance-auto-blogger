---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase table/column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read this before any write in routine.md's "Open the run log" or "Publish" steps.
---

# SpearlanceOS Blog Pipeline — Write-Back Reference

Project id: `chikljxwgiskyjsnjelf`. All writes go through the Supabase MCP `execute_sql`. The
routine writes to five tables only: `blog_topics`, `blog_posts`, `blog_auto_runs`, `tasks`,
`task_assignees`. Everything else in SpearlanceOS is read-only.

Schema below was read live from `information_schema.columns` on 2026-08-04 — re-verify if a write
starts erroring with an unknown-column message, since SpearlanceOS may add columns over time.

## 1. Open the run log — `blog_auto_runs` (per client, before any research)

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'cron', 'running')
RETURNING id;
```

- `trigger_type`: `'cron'` for the normal hourly tick; `'manual'` only when invoked ad hoc for a
  single `client_id` (routine.md Step 0b).
- Keep the returned `id` as `<run_id>` — every other write below references it.

## 2. `blog_topics` — mark the topic used

Columns: `id`, `client_id`, `topic_title`, `description`, `summary`, `keywords[]`, `category`,
`priority` (text, default `'medium'`), `status` (text, default `'idea'`), `blog_post_id`,
`suggested_publish_date`, `ai_generated` (bool), `strategy_batch_id`, `avatar_id`, `created_at`.

**If you generated a new topic (Step 4b):**

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<topic_title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>', true, 'in_progress', current_date)
RETURNING id;
```

**If you used a calendar topic (Step 4a),** mark it `in_progress` as soon as you commit to it:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**After the draft is created in Duda** (Publish step), close the topic out — `'published'` here
means "turned into a post," not "live on the web":

```sql
UPDATE public.blog_topics
SET status = 'published', blog_post_id = '<blog_posts_row_id>'
WHERE id = '<topic_id>';
```

## 3. `blog_posts` — the review/schedule record Mila works from

Key columns: `id`, `client_id`, `title`, `slug`, `meta_description`, `content`, `focus_keyword`,
`target_keywords[]`, `topic_category`, `featured_image_url`, `featured_image_alt`, `status` (text,
default `'draft'`; observed values in use: `draft` / `scheduled` / `published`), `duda_page_id`,
`duda_publish_url`, `topic_id`, `auto_run_id`, `quality_scores` (jsonb), `scheduled_publish_date`,
`created_at`.

Insert **one row per drafted article**, always `status = 'draft'` — never `'scheduled'` or
`'published'`; that transition is Mila's call, made in SpearlanceOS, not the routine's.

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, target_keywords,
   topic_category, featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
   topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta_description>', '<full HTML content>', '<primary keyword>',
   ARRAY['<kw1>','<kw2>'], '<category>', '<thumbnail/main image URL used>', '<alt text>', 'draft',
   '<duda post id>', '<duda preview/editor URL>', '<topic_id>', '<run_id>',
   '{"verdict":"SHIP","title":"PASS","description":"PASS","headings":"PASS","content":"PASS","interlinking":"PASS","schema":"PASS - Article; FAQ JSON-LD needs manual Duda Custom HTML","intent":"PASS","escalation":"none"}'::jsonb)
RETURNING id;
```

`quality_scores` should mirror the doctrine-compliance block from QA (SKILL.md's §1–§14.3
rule-by-rule verdict) so the JSON is auditable, not just a bare "SHIP".

## 4. `blog_auto_runs` — close out the row opened in step 1

Columns beyond the ones above: `topics_generated`, `articles_generated`, `articles_passed_gate`,
`articles_flagged` (all `integer`, default `0`), `research_summary` (jsonb), `completed_at`,
`error_log`.

**Draft created:**

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = <0 or 1>, articles_generated = 1, articles_passed_gate = 1, articles_flagged = 0,
    research_summary = '{"keyword":"<primary keyword>","sources":["<url1>","<url2>"]}'::jsonb
WHERE id = '<run_id>';
```

**Branch stopped instead (QA fail, no fuel, no strategy row, doctrine halt, escalation, saturated
topic space, etc.):**

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, specific enough for the next tick to act on>'
WHERE id = '<run_id>';
```

Every row opened in step 1 gets closed exactly once, one way or the other, before moving to the
next client. Do not leave a run row `status = 'running'`.

## 5. `tasks` + `task_assignees` — only when a draft was created

Mila's SpearlanceOS user id (from `public.profiles`, email `mila@spearlance.com`):
**`4b9df73e-bc38-49ef-840e-75fdb0b29c29`** — re-look-up by email if this ever bounces (`SELECT id
FROM public.profiles WHERE email = 'mila@spearlance.com'`), don't hardcode blindly forever.

`tasks.status` is an enum (`task_status`): `to_do` / `in_progress` / `done` — insert `'to_do'`.
`tasks.priority` is an enum (`task_priority`): `low` / `normal` / `high` / `urgent` — `'normal'` is
fine for a routine review task.

Look up the client's `'to_do'` column id from `task_columns` (`key = 'to_do'`, scoped to the
client — every client has its own row, not a shared global one):

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do' LIMIT 1;
```

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger drafted this post — review and schedule/publish in Duda. Preview: <duda preview/editor URL>. Doctrine QA verdict: SHIP.',
  'to_do', 'normal', '4b9df73e-bc38-49ef-840e-75fdb0b29c29', '<to_do column id>'
)
RETURNING id;
```

```sql
INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task id from above>', '4b9df73e-bc38-49ef-840e-75fdb0b29c29');
```

## Notes

- `blog_topics.description` vs `blog_topics.summary`: both columns exist; the routine (and this
  skill) use **`summary`** consistently, matching routine.md's own read queries (Step 4a/4b).
- `blog_posts.ai_model` defaults to `'claude-sonnet-4-5'` — leave it at the default unless told
  otherwise.
- Never write `blog_posts.status = 'published'` or call `publish_blog_post` in Duda. The routine
  never publishes live (see routine.md, "Publishing posture").
