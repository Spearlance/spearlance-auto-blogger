---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read this during the Publish and "open the run log" steps of routine.md instead of guessing column names.
---

# SpearlanceOS blog pipeline write-back

Project: `chikljxwgiskyjsnjelf` (production — see `supabase-prod-guard`; these writes are
pre-authorized by `CLAUDE.md`/`routine.md` as the routine's designed write scope, nothing else is).

Introspected via `information_schema.columns` and confirmed sample rows. Re-verify with
`list_tables(verbose=true)` if a write ever errors on an unknown column — the app owns this schema
and can change it without notice.

## Known trigger: one draft per client per day

`trg_one_blog_draft_per_client_per_day` (enabled, on `public.blog_posts`) blocks inserting a second
`blog_posts` row for the same `client_id` on the same calendar day. **Before drafting anything**,
check same-day inventory first — cheaper than doing research that gets thrown away:

```sql
SELECT id, status, created_at, topic_id, auto_run_id
FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at::date = current_date;
```

If a row already exists, do not draft — close the run log as `failed` (see below) with an
`error_log` explaining the same-day cap, and move to the next due client. This is expected to hit
often on an hourly cron with a calendar that says "due" for the whole day; it is not a bug to route
around, it's the actual cap. (The due-check in routine.md Step 0c currently doesn't account for this,
so most hourly ticks on a due day after the first draft will find nothing to do — that's a known,
reported inefficiency, not a signal to loosen the trigger or fake a status.)

## 1. `blog_auto_runs` — one row per client per attempt

Open at the very start of a client's processing, before any research:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type`: `'scheduled'` for the normal hourly cron tick; `'manual'` only when invoked ad hoc
  for a single `client_id` (routine.md Step 0b).
- Keep the returned `id` as `<run_id>` for `blog_posts.auto_run_id` and the closing update below.

Columns: `id` (uuid, pk), `client_id` (uuid, required), `triggered_at` (timestamptz, default now()),
`trigger_type` (text, required), `status` (text, default `'running'`), `topics_generated` (int,
default 0), `articles_generated` (int, default 0), `articles_passed_gate` (int, default 0),
`articles_flagged` (int, default 0), `research_summary` (jsonb), `completed_at` (timestamptz),
`error_log` (text), `created_at` (timestamptz, default now()).

Close it once, at the end of this client's branch, whether or not a draft was created:

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',              -- or 'failed' if the branch stopped short
    articles_generated = 1,            -- counts actually produced this run
    articles_passed_gate = 1,          -- 1 if the draft cleared doctrine QA, else 0
    topics_generated = 0,              -- 1 if Step 4b generated+saved a new topic
    completed_at = now(),
    error_log = NULL                   -- set to the reason when status = 'failed'
WHERE id = '<run_id>';
```

`status = 'failed'` covers: same-day draft cap already hit, no strategy row (shouldn't reach here —
filtered in Step 0c), no usable topic + generation failed, doctrine/QA kill gate, missing brand fuel
combined with an unsourceable draft, or a halted preflight. Put the specific reason in `error_log` —
that text is what the end-of-run report and the next tick's operator read.

## 2. `blog_topics` — mark the used/created topic done

```sql
-- Existing calendar topic taken in Step 4a:
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
-- ... after a draft is created:
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';

-- New topic generated in Step 4b:
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, priority, ai_generated, status, suggested_publish_date)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', 'high', true, 'in_progress', current_date)
RETURNING id;
```

Columns: `id`, `client_id`, `topic_title` (text, required), `description` (text — routine.md's
"summary" for calendar topics selected in 4a lands here on older rows; newer inserts should use
`summary` below, both exist), `keywords` (text[]), `avatar_id` (uuid), `priority` (text, default
`'medium'`; observed values `'high'`/`'medium'`/`'low'`), `status` (text, default `'idea'`; the
routine's lifecycle is `idea`/`in_progress`/`published` — never insert `'published'` or leave a used
topic at `'idea'`), `blog_post_id` (uuid, set once a `blog_posts` row exists), `ai_generated`
(boolean), `created_at`, `strategy_batch_id` (uuid, only set by SpearlanceOS's own calendar batches —
leave null), `summary` (text), `suggested_publish_date` (date), `category` (text).

## 3. `blog_posts` — one row per drafted article, always `status = 'draft'`

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
   topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<url-slug>', '<meta description>', '<full HTML or markdown body sent to Duda>',
   '<short excerpt>', '<primary keyword>', '<category>', ARRAY['kw1','kw2'],
   '<thumbnail/main image URL used in Duda>', '<alt text>', 'draft',
   '<Duda create_blog_post response id>', '<real constructed public post URL, not a dashboard/editor link>',
   '<topic_id>', '<run_id>', '{"doctrine_verdict":"SHIP","word_count":1650,"h2_count":5,"internal_links":7,"sources":["https://..."]}'::jsonb)
RETURNING id;
```

- `status` is always `'draft'` from this routine — never `'scheduled'` or `'published'`, that's
  Mila's call.
- `duda_publish_url`: use the real public post URL you can construct from the client's live domain +
  the post's `path` (from `create_blog_post`/`get_blog_post`), e.g.
  `https://<client-domain>/<post-path>`. Do **not** store a Duda editor/dashboard link
  (`mywebsitemanager.co/home/site/<site_id>...`) here — that's an internal Duda admin link, not
  something Mila or a reader can use to preview the post, and it has been the cause of at least one
  repeat data-quality flag on this table.
- Other columns exist (`seo_score`, `readability_score`, `avatar_id`, `body_images`, `scheduled_for`,
  `published_at`, `ai_model` default `'claude-sonnet-4-5'`, `generation_prompt`,
  `generation_metadata`, `created_by`, `last_edited_by`, `scheduled_publish_date`,
  `images_generated`, `rejection_reason`, `revision_count`) — leave them at their defaults/null unless
  you have a real value; don't guess into `rejection_reason` or `seo_score`.

## 4. `tasks` + `task_assignees` — review task for Mila (only when a draft was created)

```sql
-- Find Mila's user id and the client's default/"to_do" column once per client:
SELECT id FROM public.users WHERE email = 'mila@spearlance.com' LIMIT 1; -- confirm actual email/table first
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do' LIMIT 1;

INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES ('<client_id>', 'Review blog draft: <post title>',
        'Auto Blogger created a draft for review/scheduling. Duda post: <duda_publish_url>. Topic: <topic_title>.',
        'to_do', 'normal', '<mila_user_id>', '<to_do_column_id>')
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task_id>', '<mila_user_id>');
```

`tasks.status` and `tasks.priority` are Postgres enums (`task_status`, `task_priority`) — use the
same literal values already present in `task_columns.mapped_status` / existing rows rather than
inventing new ones. Verify the exact user-lookup table/column (`users` vs `profiles` vs `team_members`)
and Mila's email against the live schema before the first insert of a given session; don't assume.

## Never

Never write to any other SpearlanceOS table, never call a `blog-*` edge function, never set
`blog_posts.status` to anything but `'draft'`, never call Duda's `publish_blog_post`.
