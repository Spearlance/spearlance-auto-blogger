---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the Auto Blogger's SpearlanceOS write-back (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read this before writing any of these rows during a routine.md run.
---

# SpearlanceOS blog pipeline write-back

Schema confirmed live against project `chikljxwgiskyjsnjelf` (information_schema introspection,
2026-08-05). This file exists so a run never has to reverse-engineer the schema from scratch —
previous runs did this repeatedly via ad hoc `information_schema` queries because this file was
missing. If a column below stops matching reality, re-introspect and update this file; don't guess.

## 1. Open the run log — start of every client's processing

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type`: `'scheduled'` for the normal hourly cron; `'manual'` only when invoked ad hoc for
  a single `client_id` (Step 0b test/dry-run).
- Keep the returned `id` as `<run_id>` — every other write-back this client references it.
- Columns available for the closing update: `status`, `topics_generated`, `articles_generated`,
  `articles_passed_gate`, `articles_flagged`, `research_summary` (jsonb), `completed_at`, `error_log`.

## 2. Topic selection/generation

**Using an existing calendar topic (Step 4a):** mark it `in_progress` as soon as you commit to it,
before drafting:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**Generating a new topic (Step 4b):** insert it directly as `in_progress` (there's no separate
"idea → in_progress" transition needed for AI-generated topics):

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>',
   true, 'in_progress', current_date)
RETURNING id;
```

- `keywords` is a text array (`ARRAY[...]`), not jsonb.
- `category` should match one of the client's `blog_content_strategy.content_mix` keys
  (`how_to`, `case_studies`, `industry_news`, `best_practices`, `company_updates`, etc.) so the mix
  stays trackable.
- `priority` defaults to `'medium'` (text, not numeric) — only set it explicitly if you have a
  reason to outrank the rest of the backlog.
- `blog_post_id` stays NULL here; it's populated by SpearlanceOS's own review flow, not by this
  routine. The routine's own "done" marker is `status`, not this column.

**After a draft is created (either path)** — mark the topic done. This is the routine's
"turned into a post" marker; it does not mean the post is live in Duda:

```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

**If the branch is stopped instead (QA fail, no fuel, saturated, etc.):** leave the topic as-is
(`in_progress` if you'd already claimed it) rather than marking it `published` — a future run should
be able to reconsider it, unless the topic itself is the problem (stale/off-season/duplicate), in
which case set `status = 'skipped'` so it stops surfacing in the Step 4a query.

## 3. Insert the article row — after the Duda draft is created

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, topic_id, auto_run_id,
   status, focus_keyword, target_keywords, featured_image_url, featured_image_alt,
   duda_page_id, duda_publish_url, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta_description>', '<html_content>',
   '<topic_id>', '<run_id>',
   'draft', '<primary_keyword>', ARRAY['<kw1>','<kw2>'], '<main_image_url>', '<alt_text>',
   '<duda_post_id_from_create_blog_post_response>', '<duda_preview_or_editor_url>',
   '{"verdict":"SHIP","word_count":1650,"h2_count":6,"internal_links":8,"sourced_stats":3}'::jsonb)
RETURNING id;
```

- `status` is **always `'draft'`** from this routine — never `'scheduled'` or `'published'`; that
  transition is Mila's call, made in SpearlanceOS, not here.
- `content` is the full HTML body you sent to `create_blog_post` (keep it verbatim for the review
  record).
- `quality_scores` is jsonb — put the doctrine-compliance verdict here (§ scores, word count, H2
  count, link count, sourced-stat count) so Mila's review view can show it without opening Duda.
- `slug` should match the Duda post's path if `create_blog_post` normalized the title into one.

## 4. Close the run log — end of every client's processing

**Draft created:**

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',
    completed_at = now(),
    topics_generated = 1,
    articles_generated = 1,
    articles_passed_gate = 1,
    research_summary = '{"sources":["https://...","https://..."],"internal_links":8}'::jsonb
WHERE id = '<run_id>';
```

**Branch stopped instead (no draft):**

```sql
UPDATE public.blog_auto_runs
SET status = 'failed',
    completed_at = now(),
    error_log = '<why: QA fail detail / no fuel / no strategy row / topic saturation / doctrine halt / escalation>'
WHERE id = '<run_id>';
```

- `status = 'failed'` here means "this tick did not ship a draft," not necessarily an error —
  a well-reasoned decline (topic saturation, no qualifying angle) is a correct use of `'failed'`.
  Put the actual reasoning in `error_log`; the end-of-run report reads from here.
- Every opened run row (Section 1) must be closed one way or the other before moving to the next
  client. Never leave a row `status = 'running'`.

## 5. Task for Mila — only when a draft was created

Look up Mila by email, and the client's `'to_do'` column:

```sql
WITH mila AS (
  SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com' LIMIT 1
), todo_col AS (
  SELECT id FROM public.task_columns
  WHERE client_id = '<client_id>' AND key = 'to_do'
  LIMIT 1
)
INSERT INTO public.tasks (client_id, title, description, assignee_user_id, column_id, status)
SELECT '<client_id>',
       'Review blog draft: <post_title>',
       'Auto Blogger draft ready for review. Duda preview: <preview_url>. Doctrine verdict: SHIP.',
       mila.id, todo_col.id, 'to_do'
FROM mila, todo_col
RETURNING id;
```

```sql
INSERT INTO public.task_assignees (task_id, user_id)
SELECT '<task_id>', id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

- Confirm Mila's actual `profiles.email` once per session rather than hardcoding — the address
  above is a placeholder if it doesn't match.
- `tasks.status` is an enum (`task_status`); `'to_do'` matches `task_columns.mapped_status` default
  and is the correct initial value — never insert as further along.
- If the client has no `'to_do'` column yet, don't silently skip the task — note it in the
  end-of-run report as a setup gap for that client (SpearlanceOS should provision default columns
  per client, but evidently doesn't always).

## Known standing defects (not fixed by write-back logic — flag, don't silently work around)

- **`create_blog_post` double-fire**: some Duda posts have been created in duplicate (identical
  title, creation timestamps ~3 min apart). Confirmed across multiple clients. If `list_blog_posts`
  shows a title you're about to use already appearing twice, that's this defect, not a dedup miss
  on your part — flag it for Mila to delete the redundant copy, don't try to "fix" it by skipping
  your own topic.
- **No per-client daily cooldown in some deployed copies of the schedule prompt**: `routine.md` in
  this repo (Step 0c) requires a client to have **not already posted today** before it's due again.
  If the live scheduled prompt you were fired with does *not* contain that clause, it is running a
  stale copy — the fix already exists in `main` (commit `04bef08`). Flag this explicitly rather than
  mechanically producing a second/third draft for an already-served client that day.
