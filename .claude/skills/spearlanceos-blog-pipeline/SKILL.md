---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase write-back for the Auto Blogger routine — column names, status values, and copy-pasteable SQL for blog_auto_runs, blog_topics, blog_posts, and the tasks/task_assignees review handoff to Mila. Read this whenever routine.md's Draft/QA/Publish or "Record the Run" steps say to write to SpearlanceOS.
---

# SpearlanceOS Blog Pipeline — Write-Back Reference

`routine.md` writes to exactly four tables in SpearlanceOS (project `chikljxwgiskyjsnjelf`), in this
order, once per due client processed. Everything else in SpearlanceOS is read-only. Never call
SpearlanceOS's `blog-*` edge functions.

## 1. Open the run log — before any research/drafting for the selected client

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')  -- trigger_type = 'manual' only for single-client test runs
RETURNING id;  -- keep as <run_id>, close this exact row at the end of this client's processing
```

Columns: `id, client_id, triggered_at (default now()), trigger_type, status (default 'running'),
topics_generated (int, default 0), articles_generated (int, default 0), articles_passed_gate (int,
default 0), articles_flagged (int, default 0), research_summary (jsonb), completed_at, error_log,
created_at`.

## 2. If generating a new topic (Step 4b — no usable calendar topic existed)

```sql
INSERT INTO public.blog_topics (
  client_id, topic_title, summary, keywords, category, priority,
  ai_generated, status, suggested_publish_date
) VALUES (
  '<client_id>', '<title>', '<1-2 sentence summary>', ARRAY['kw1','kw2'], '<category>',
  'medium', true, 'in_progress', current_date
)
RETURNING id;  -- keep as <topic_id>
```

If instead you took an existing due row from `blog_topics` (Step 4a), mark it `in_progress` now:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

## 3. After the Duda draft is created — mark the topic done

`'published'` here means "turned into a post," not "live" — matches the table's existing convention.
Every draft leaves Duda **unpublished**; this status is about the topic queue, not the post.

```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

## 4. Insert the `blog_posts` row — the actual review/schedule record Mila works from

**Always `status = 'draft'`.** Never insert `'scheduled'` or `'published'` — that's Mila's call, made
in her own review flow, not the routine's.

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt, focus_keyword,
  topic_category, target_keywords, featured_image_url, featured_image_alt,
  status, duda_page_id, duda_publish_url, topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<full title incl. brand suffix>', '<duda path slug>', '<meta description>',
  $content$<full HTML body exactly as sent to create_blog_post>$content$,
  '<1-2 sentence excerpt>', '<primary target keyword>',
  '<category>', ARRAY['kw1','kw2'], '<main_image_url from Duda response>', '<alt text>',
  'draft', '<duda post id>', '<full public preview URL>', '<topic_id>', '<run_id>',
  '{"verdict":"SHIP","word_count":0,"h2_count":0,"internal_links":0,"external_citations":0,
    "sources":["..."]}'::jsonb
)
RETURNING id;
```

Key columns worth knowing: `status` (default `'draft'`), `duda_page_id`/`duda_publish_url` (from the
`create_blog_post` response), `topic_id`/`auto_run_id` (FKs tying the row to steps 1–2 above),
`quality_scores` (jsonb — put the doctrine QA verdict + sourcing list here for Mila's/Garrett's
review), `rejection_reason`, `revision_count`.

Use a `$content$...$content$` dollar-quoted string (not single quotes) for `content` — the HTML body
contains apostrophes and double quotes that would otherwise need escaping.

## 5. Close the run log — every opened row gets closed, success or not

**Success (draft created):**

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', topics_generated = 1, articles_generated = 1,
    articles_passed_gate = 1, articles_flagged = 0,
    research_summary = '{"topic":"...","target_keyword":"...","intent":"...",
      "sources":["..."],"duda_post_id":"...","duda_preview_url":"...","task_id":"..."}'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';
```

**Branch stopped instead (QA fail, no fuel, no strategy row, doctrine halt, saturation, escalation):**

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', error_log = '<why this client's branch stopped — be specific and reusable
      by the next hourly tick: what was checked, what was found, why no draft cleared the bar>',
    completed_at = now()
WHERE id = '<run_id>';
```

## 6. Only when a draft was created — task for Mila

Look up Mila by email in `profiles` (not `auth.users` — there is no `public.users` table), and the
client's `'to_do'` column in `task_columns`:

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';               -- <mila_user_id>
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';  -- <column_id>
```

```sql
WITH new_task AS (
  INSERT INTO public.tasks (client_id, title, description, status, priority, column_id)
  VALUES (
    '<client_id>', 'Review blog draft: <post title>',
    'Auto Blogger created a new draft for <client>. Review, add FAQ JSON-LD via Duda Settings > '
    || 'Custom HTML if the post has an FAQ section (Duda cannot inject it through the post body), '
    || 'then schedule/publish.\n\nDuda editor: <edit_site_url>\nPreview: <duda_publish_url>\n'
    || 'QA verdict: <SHIP/etc, word count, link count, sources>',
    'to_do', 'normal', '<column_id>'
  )
  RETURNING id
)
INSERT INTO public.task_assignees (task_id, user_id)
SELECT id, '<mila_user_id>' FROM new_task
RETURNING task_id;
```

`tasks.status` and `tasks.priority` are enum types (`task_status`, `task_priority`) — `'to_do'` and
`'normal'` are safe defaults matching the column defaults.

## Schema notes (introspected 2026-08-05 via `information_schema.columns` — reintrospect if writes
start failing with an unknown-column error, schema may have drifted)

- `blog_auto_runs`: no `articles_generated` ≠ posts count elsewhere — it's local to this table.
- `blog_posts.content` and most metadata columns are nullable, but populate them fully — this row
  is what Mila's review UI reads, not just a status flag.
- `blog_posts.target_keywords` and `tasks.related_asset_ids`/`related_meeting_ids` are Postgres
  arrays — use `ARRAY[...]`, not a JSON array literal.
- There is no `public.users` table. User/profile lookups (e.g., for `tasks.assignee_user_id` or
  `task_assignees.user_id`) go through `public.profiles`, keyed by `email`.
