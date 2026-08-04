---
name: spearlanceos-blog-pipeline
description: Exact table schemas, status values, and copy-pasteable SQL for every write the Auto Blogger routine makes to SpearlanceOS (blog_auto_runs, blog_posts, blog_topics, tasks, task_assignees). Read this whenever routine.md says "see the spearlanceos-blog-pipeline skill" — at opening a run log, and at Publish (writing back the topic, the post, closing the run, and creating Mila's review task).
---

# SpearlanceOS blog pipeline — write reference

The routine reads broadly from SpearlanceOS but writes **only** to the five tables below, all on
project `chikljxwgiskyjsnjelf`. Nothing here should be used to justify writing anywhere else, and never
call SpearlanceOS's `blog-*` edge functions.

## 1. Open the run log — `blog_auto_runs`

Insert this **before** any research/drafting for the client, at the top of Step 0d's per-client
processing. Keep the returned `id` as `<run_id>` for the close-out write and for `blog_posts.auto_run_id`.

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type`: `'scheduled'` for the normal hourly cron tick; `'manual'` only when invoked ad hoc for
  a single `client_id` (Step 0b dry-run case).
- Columns available for the close-out update: `status`, `topics_generated`, `articles_generated`,
  `articles_passed_gate`, `articles_flagged`, `research_summary` (jsonb), `completed_at`, `error_log`.

### Close the run log (always — success or decline)

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',            -- or 'failed' if the branch was stopped
    topics_generated = <0 or 1>,
    articles_generated = <0 or 1>,
    articles_passed_gate = <0 or 1>, -- 1 only if QA verdict was SHIP
    articles_flagged = <0 or 1>,     -- 1 if escalated to Garrett
    completed_at = now(),
    error_log = <null on success, else the reason: QA fail / no fuel / no strategy row / doctrine halt / escalation>
WHERE id = '<run_id>';
```

If you never got as far as opening a run log (e.g., preflight halted the whole run before client
selection), there is nothing to close — the report covers it instead.

## 2. Mark the topic done — `blog_topics`

Whether the topic came from the calendar (Step 4a) or was freshly generated (Step 4b), once a draft is
successfully created in Duda:

```sql
UPDATE public.blog_topics
SET status = 'published'   -- means "turned into a post," not "live" — the table's existing convention
WHERE id = '<topic_id>';
```

Leave `blog_post_id` on `blog_topics` alone unless you're deliberately linking it; the routine's own
`blog_posts` row (below) is the actual review record.

If a topic was newly generated in Step 4b, its INSERT already used `status = 'in_progress'`; this UPDATE
is what flips it to `'published'` after the draft is created. If QA fails and no draft is created, leave
a generated topic at `'in_progress'` (or revert it) rather than marking it `'published'` — it wasn't used.

## 3. Insert the review record — `blog_posts`

One row per drafted article, inserted only when `create_blog_post` succeeded in Duda:

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt,
  focus_keyword, target_keywords, topic_category,
  featured_image_url, featured_image_alt,
  status, topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<duda_post_path_or_slug>', '<meta_description>', '<full_html_content>', '<short_excerpt>',
  '<primary_keyword>', ARRAY['<kw1>','<kw2>', ...], '<category>',
  '<thumbnail_url>', '<alt_text>',
  'draft', '<topic_id>', '<run_id>', '<qa_verdict_jsonb>'::jsonb
);
```

- `status` is **always** `'draft'` — never `'scheduled'` or `'published'`; that transition is Mila's
  call, made in her own review flow, not the routine's.
- `quality_scores` (jsonb): store the doctrine compliance verdict, e.g.
  `{"verdict":"SHIP","word_count":1650,"h2_count":6,"internal_links":8,"sources":["https://...","https://..."]}`.
- `duda_page_id` / `duda_publish_url`: fill in from the `create_blog_post` response so Mila's queue links
  straight to the Duda draft.
- There is no unique constraint tying this row to the Duda post beyond `duda_page_id` — set it every
  time so the two systems can be cross-referenced.

## 4. Create Mila's review task — `tasks` + `task_assignees`

Only when a draft was actually created. Look up Mila's user id by email in `auth.users` (there is no
`public.users`/`public.profiles`-based assignee table — `task_assignees.user_id` points at
`auth.users.id`), and the client's `'to_do'` task column:

```sql
-- Mila's user id (cache per run if processing multiple clients) — confirmed: mila@spearlance.com
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';

-- the client's To Do column
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

```sql
INSERT INTO public.tasks (client_id, title, description, status, column_id, related_asset_ids)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger created a draft for review. Duda preview: <preview_url>. Target keyword: <keyword>. QA verdict: SHIP. Sources: <source urls>.',
  'to_do',
  '<to_do_column_id>',
  ARRAY['<blog_posts.id>']::uuid[]
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_user_id>');
```

- `tasks.status` is an enum (`task_status`); `'to_do'` is the correct starting state.
- `related_asset_ids` is a uuid array — link it to the new `blog_posts.id` so Mila can jump straight
  from the task to the draft record.

## Write order per client (recap)

1. Open `blog_auto_runs` (status `'running'`) before research starts.
2. Do the work (Steps 1–4, Draft → QA).
3. If QA fails / no fuel / no strategy / halted: close `blog_auto_runs` as `'failed'` with `error_log`
   set. Stop — no `blog_topics`, `blog_posts`, or `tasks` writes.
4. If QA passes and Duda's `create_blog_post` succeeds: mark `blog_topics.status = 'published'`, insert
   the `blog_posts` row, close `blog_auto_runs` as `'completed'` with counts, then create the `tasks` +
   `task_assignees` row for Mila.

Never write to any other SpearlanceOS table, and never call its `blog-*` edge functions.
