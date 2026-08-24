---
name: spearlanceos-blog-pipeline
description: >
  Exact SpearlanceOS Supabase write-back for the Auto Blogger routine — column names, status
  values, and copy-pasteable SQL for blog_topics, blog_posts, blog_auto_runs, tasks, and
  task_assignees. Use this any time routine.md's Publish or Record-the-run steps say "write
  back to SpearlanceOS." Project id chikljxwgiskyjsnjelf (production — see supabase-prod-guard;
  this routine's writes to these five tables are pre-authorized by CLAUDE.md/routine.md as its
  standing, scoped design, not a one-off write needing a fresh yes each run).
---

# SpearlanceOS blog pipeline write-back

Five tables, in this order, once per client per tick. Never write anything else in SpearlanceOS;
never call its `blog-*` edge functions.

## 0. Before you do ANY research or drafting: the daily-draft guard

A DB trigger, `guard_one_blog_draft_per_client_per_day`, rejects a second `blog_posts` INSERT for
a client on the same calendar day (UTC). **routine.md Step 0c's due-check is calendar-only by
design and does not check this** — so check it yourself, cheaply, before opening the doctrine or
doing any research:

```sql
SELECT id, created_at FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at >= current_date
LIMIT 1;
```

If a row comes back, this client has already been served today. **Do not draft.** A full
research+draft+QA cycle run anyway will pass QA, create a real Duda draft, and then fail at the
`blog_posts` INSERT — burning Tavily/DataforSEO calls and leaving an orphaned, untracked Duda
draft for nothing. Log a `blog_auto_runs` row (below) noting the skip and move to the next due
client, or end the run if none remain.

## 1. Open the run log (start of each client's processing)

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

`trigger_type` = `'scheduled'` for the normal hourly cron; `'manual'` only when invoked ad hoc for
a single `client_id`. Keep the returned `id` as `<run_id>` — every other write this tick references
it, and this same row is closed exactly once, at the end (step 5).

## 2. `blog_topics` — mark the topic used

If you used an existing calendar topic (routine.md Step 4a), mark it `in_progress` as soon as you
commit to it, then `published` once the Duda draft is created (this table's `published` means
"turned into a post," matching its existing convention — it does not mean the Duda post went live):

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
-- ... later, after the Duda draft exists ...
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

If you generated a new topic (Step 4b), insert it directly with `status = 'in_progress'`, then flip
it to `'published'` at the same point:

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, priority, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>', 'medium', true, 'in_progress', current_date)
RETURNING id;
```

Columns that exist but routine.md doesn't mention: `description` (free text, distinct from
`summary`), `avatar_id`, `strategy_batch_id`, `blog_post_id` (leave NULL — the FK the other
direction, `blog_posts.topic_id`, is what actually links them).

## 3. `blog_posts` — one row per drafted article (Mila's review queue)

Insert only after the Duda draft has actually been created (you need `duda_page_id`). `status` is
**always `'draft'`** — never `'scheduled'` or `'published'`; that transition is Mila's call, made in
her own review flow, not this routine's.

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status, duda_page_id,
   ai_model, topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<duda-path-slug>', '<meta_description>', '<full HTML content>',
   '<focus_keyword>', '<category>', ARRAY['<kw1>','<kw2>'], '<thumbnail_url>', '<alt_text>',
   'draft', '<duda_post_id>', 'claude', '<topic_id>', '<run_id>',
   '{"verdict":"SHIP","word_count":1850,"h2_count":6,"internal_links":7}'::jsonb)
RETURNING id;
```

`quality_scores` is free-form JSONB — put the doctrine QA verdict and whatever counts you checked
(word count, H2 count, internal link count, sourced-stat count) so Mila and Garrett can see why it
passed without re-reading the draft. `slug` is required (`NOT NULL`) — reuse the Duda path. Other
real columns available if useful: `excerpt`, `seo_score`, `readability_score`, `body_images` (jsonb),
`scheduled_publish_date`, `rejection_reason` (for a future QA-fail record if you ever insert a
rejected draft — routine.md currently says don't publish on fail, so this stays unused for now).

## 4. `blog_auto_runs` — close the row opened in step 1

```sql
-- success (a draft was created and blog_posts/tasks written):
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = 1, articles_generated = 1, articles_passed_gate = 1
WHERE id = '<run_id>';

-- branch stopped instead (QA fail, no fuel, no strategy row, doctrine halt, daily-draft guard, escalation):
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, one or two sentences>'
WHERE id = '<run_id>';
```

Every opened run log gets closed exactly once, one way or the other, before moving to the next
client. `research_summary` (jsonb) is available if you want to capture the source URLs / keyword
data used, but is not required.

## 5. `tasks` + `task_assignees` — only when a `blog_posts` row was actually created

Look up the client's "To Do" column and Mila's user id, then insert the task and assign it:

```sql
-- 1) the client's default To-Do column
SELECT id FROM public.task_columns
WHERE client_id = '<client_id>' AND key = 'to_do'
LIMIT 1;

-- 2) Mila's user id (stable; safe to hardcode once looked up, but re-verify by email if it ever 404s)
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';

-- 3) the task itself
INSERT INTO public.tasks (client_id, title, description, status, column_id)
VALUES (
  '<client_id>',
  'Review Duda draft: <post title>',
  'Auto Blogger created a draft for review. Duda preview: <preview_url>. Keyword: <keyword>. QA verdict: SHIP.',
  'to_do',
  '<to_do_column_id>'
)
RETURNING id;

-- 4) assign it to Mila
INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_user_id>');
```

`tasks.status` and `task_columns.mapped_status` share the `task_status` enum (`to_do` /
`in_progress` / `done`) — keep them in sync (`status='to_do'` with the `to_do` column's id).

## Known open items (as of 2026-08-24, flagged repeatedly in blog_auto_runs.error_log all day)

- This file was missing from the repo entirely until now, forcing every prior tick today to guess
  at schema or skip the write-back steps. It's now committed — no more in-session-only reconstruction.
- The **live scheduled trigger's stored prompt** (what actually fires hourly) is a separate, newer
  copy of the playbook than what's checked into `routine.md` in this repo, and neither one had the
  step-0 daily-draft guard above. Editing files in this repo does not change the stored schedule
  prompt — only Garrett can refresh that from the scheduling UI. Until he does, expect the *next*
  scheduled run to still skip the guard (this file and `routine.md` are correct as of this commit;
  the live prompt is not) — an agent working a due client should apply the guard above manually even
  though the fired prompt doesn't say to.
