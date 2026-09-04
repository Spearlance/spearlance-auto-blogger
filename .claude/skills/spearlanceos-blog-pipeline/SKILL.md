---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back tables (blog_topics, blog_auto_runs, blog_posts, tasks, task_assignees). Read by routine.md at "Open the run log" and at "Publish". Project id chikljxwgiskyjsnjelf. Schema introspected live via information_schema/pg_constraint/pg_trigger; re-verify if a write fails with an unknown-column or check-constraint error, since the app owns this schema and can change it.
---

# SpearlanceOS blog write-back: exact schema + SQL

All statements target project **`chikljxwgiskyjsnjelf`**. Every table below is otherwise read-only
per CLAUDE.md; this skill only covers the columns the Auto Blogger actually writes.

## Important DB behavior not in routine.md

- **`public.blog_posts` has a trigger `trg_one_blog_draft_per_client_per_day`** (`BEFORE INSERT`,
  function `guard_one_blog_draft_per_client_per_day()`) that rejects a second `blog_posts` insert for
  the same `client_id` on the same calendar day. In practice this means: once a due client has one
  draft today, every later tick for that client this day will fail at the `blog_posts` insert — this
  is expected, not a bug to route around. **Check `blog_posts` for an existing row with
  `created_at::date = current_date` for the client before doing any research**; if one exists, skip
  straight to logging a `failed` run (see below) and move to the next due client. Don't burn a
  Tavily/DataForSEO research pass on a client that's already capped for the day.
- `blog_posts.status` has no DB check constraint (app-level convention only) — always insert
  `'draft'`, per routine.md's publishing posture.

## 1. Open the run log — `blog_auto_runs`

Columns: `id (uuid, pk, default gen_random_uuid())`, `client_id (uuid, not null)`,
`triggered_at (timestamptz, not null, default now())`, `trigger_type (text, not null — CHECK IN
('scheduled','manual'))`, `status (text, not null, default 'running' — CHECK IN
('running','completed','failed'))`, `topics_generated (int, default 0)`,
`articles_generated (int, default 0)`, `articles_passed_gate (int, default 0)`,
`articles_flagged (int, default 0)`, `research_summary (jsonb)`, `completed_at (timestamptz)`,
`error_log (text)`, `created_at (timestamptz, default now())`.

Open at the start of a client's processing:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

Use `trigger_type = 'manual'` only for an ad hoc single-`client_id` invocation (Step 0b test/dry-run).
Keep the returned `id` as `<run_id>`.

If the branch is stopped **before** any research even starts (e.g. `blog_posts` already has today's
draft for this client — the daily-cap trigger above), it's fine to skip the open-then-update dance and
insert the row already closed in one statement:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status, error_log, completed_at)
VALUES ('<client_id>', 'scheduled', 'failed',
        'Already has a blog_posts draft today (id <existing_post_id>, created <ts>); trg_one_blog_draft_per_client_per_day blocks a second insert. No research attempted this tick.',
        now())
RETURNING id;
```

## 2. Close the run log (end of client processing)

Draft created:

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',
    articles_generated = 1,
    articles_passed_gate = 1,
    topics_generated = <0 or 1 — 1 if Step 4b generated a new topic>,
    research_summary = '<jsonb: {"sources": [...], "keyword": "...", "intent": "..."}>'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';
```

Branch stopped instead (QA fail, no fuel, no strategy row, doctrine halt, escalation, daily-cap block):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed',
    error_log = '<why — QA fail reason / no strategy row / stale topics only / etc.>',
    completed_at = now()
WHERE id = '<run_id>';
```

Every run log opened this tick gets closed one way or the other before moving to the next client.

## 3. `blog_topics` — mark the used/created topic done

```sql
-- topic taken from the calendar (Step 4a)
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
-- ... later, once the post is created ...
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>' WHERE id = '<topic_id>';

-- OR: topic generated fresh (Step 4b)
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, priority, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', 'medium', true, 'in_progress', current_date)
RETURNING id;
-- then mark 'published' + blog_post_id once the draft is created, same as above.
```

`status` is free-text (no DB check constraint); the app's own convention is
`idea → in_progress → published` (`published` = "turned into a post," not "live" — matches
CLAUDE.md). `priority` free-text too; the app uses `low`/`medium`/`high`.

## 4. `blog_posts` — the actual draft record (Mila's review queue)

Columns actually used by the routine: `client_id (uuid, not null)`, `title (text, not null)`,
`slug (text, not null)`, `meta_description (text)`, `content (text)`, `excerpt (text)`,
`focus_keyword (text)`, `topic_category (text)`, `target_keywords (text[])`,
`featured_image_url (text)`, `featured_image_alt (text)`, `status (text, default 'draft')`,
`duda_page_id (text)`, `duda_publish_url (text)`, `topic_id (uuid)`, `auto_run_id (uuid)`,
`quality_scores (jsonb)`, `created_at (timestamptz, default now())`.

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
   topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<url-slug>', '<meta description>', '<full HTML or plain content>',
   '<short excerpt>', '<primary keyword>', '<category>', ARRAY['kw1','kw2'],
   '<thumbnail/main image URL used in Duda>', '<alt text>', 'draft',
   '<duda post id from create_blog_post>', '<duda preview/editor URL>',
   '<topic_id or NULL>', '<run_id>',
   '{"verdict":"SHIP","word_count":1650,"h2_count":6,"internal_links":7,"sourced_claims":4}'::jsonb)
RETURNING id;
```

**Always `status = 'draft'`** — never `'scheduled'` or `'published'`; that's Mila's call.
`duda_publish_url` should be the post's real preview/editor URL from `create_blog_post`'s response,
not a bare dashboard link — if the Duda response only gives a dashboard URL, note that as a follow-up
in the end-of-run report rather than fabricating a page URL.

## 5. `tasks` + `task_assignees` — the review task for Mila

Mila's `profiles` row: `id = '4b9df73e-bc38-49ef-840e-75fdb0b29c29'`, `email = 'mila@spearlance.com'`,
`name = 'Mila Uriarte'`. Re-verify if this ever 404s — email is the stable lookup:

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Each client has its own `task_columns` row keyed `'to_do'` (`is_default = true`); look it up per
client rather than hardcoding an id:

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

Then:

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, column_id, creator_user_id)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger created a draft for review. Duda: <duda_publish_url or editor link>. Keyword: <primary keyword>. Sources: <n> cited. QA verdict: SHIP.',
  'to_do', 'normal', '<to_do column_id for this client>', '<Mila''s profiles.id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task id above>', '4b9df73e-bc38-49ef-840e-75fdb0b29c29');
```

`tasks.status` and `tasks.priority` are enum types (`task_status`, `task_priority`) — use the
column's existing default-matching values (`'to_do'`, `'normal'`) unless the app clearly uses others;
don't invent enum labels.

Only create the task when a draft was actually created this tick. No task on a stopped/failed branch.
