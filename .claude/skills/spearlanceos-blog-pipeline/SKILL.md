---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the Auto Blogger's four SpearlanceOS write-back tables (blog_topics, blog_posts, blog_auto_runs, tasks/task_assignees). Use whenever routine.md's Open the run log / Publish / Record the Run steps say to read this skill.
---

# SpearlanceOS blog pipeline write-back

Schema below was read live from `information_schema.columns` and `pg_constraint` on project
`chikljxwgiskyjsnjelf`. If a run finds this drifted from the live database, trust the database and
correct this file — don't silently work around a mismatch.

## 1. Open the run log — `blog_auto_runs` (Step 0, per client, before any research)

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type` check constraint accepts **only** `'scheduled'` or `'manual'` — **not** `'cron'`. Use
  `'manual'` only for an ad hoc single-`client_id` invocation (routine.md Step 0b); every normal hourly
  tick is `'scheduled'`.
- `status` check constraint accepts **only** `'running'`, `'completed'`, `'failed'`.
- Keep the returned `id` as `<run_id>` for the rest of this client's processing.

## 2. Close the run log — `blog_auto_runs` (Publish step, always, exactly once)

Draft created:

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',
    articles_generated = 1,
    articles_passed_gate = 1,
    topics_generated = <0 or 1, 1 if Step 4b generated a new topic>,
    research_summary = '<jsonb: sources cited, keyword, intent, etc.>'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';
```

No draft (QA fail / no fuel / no strategy row / saturated / doctrine halt):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed',
    research_summary = '<jsonb: what was checked this tick>'::jsonb,
    completed_at = now(),
    error_log = '<why the branch stopped>'
WHERE id = '<run_id>';
```

`articles_flagged` increments only when a doctrine escalation trigger fires (see routine.md QA step).

## 3. Mark the topic used — `blog_topics`

```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

This means "turned into a draft post," not "live" — the table's existing convention. If Step 4b
generated the topic this run, it was already inserted with `status = 'in_progress'`; this UPDATE is what
closes it out once the Duda draft exists.

## 4. Insert the article — `blog_posts` (only when a draft was created)

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt, focus_keyword,
  topic_category, target_keywords, featured_image_url, featured_image_alt,
  status, duda_page_id, duda_publish_url, topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<slug>', '<meta_description>', '<full HTML content>', '<short excerpt>',
  '<primary target keyword>', '<category, e.g. how_to>', ARRAY['<kw1>','<kw2>'],
  '<featured image URL>', '<alt text>',
  'draft', '<duda blog post id>', '<duda preview/editor URL>', '<topic_id>', '<run_id>',
  '<jsonb: doctrine QA verdict, per-rule pass/fail, word count, H2 count, link count, sources>'::jsonb
);
```

- `status` **must** be `'draft'` — never `'scheduled'` or `'published'`; that transition is Mila's call,
  made in her own review flow, not the routine's.
- `topic_id` links back to the `blog_topics` row used/generated in Step 4.
- `auto_run_id` links back to the `blog_auto_runs` row opened in Step 0.

## 5. Notify Mila — `tasks` + `task_assignees` (only when a draft was created)

Look up Mila's user id (in `public.profiles`, not `auth.users`) and the client's `to_do` column id:

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
-- id = 4b9df73e-bc38-49ef-840e-75fdb0b29c29 (Mila Uriarte) as of this writing; re-look-up each run
-- in case it changes, don't hardcode it.
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

If no row matches by email in `public.profiles`, don't skip the task silently — surface it in the
end-of-run report so Garrett can fix Mila's profile row or the lookup.

```sql
INSERT INTO public.tasks (client_id, title, description, status, column_id, assignee_user_id)
VALUES (
  '<client_id>', 'Review blog draft: <post title>',
  'Auto Blogger drafted this post for review. Duda preview: <duda_preview_url>. Doctrine QA: SHIP.',
  'to_do', '<to_do_column_id>', '<mila_user_id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id_from_above>', '<mila_user_id>');
```

`tasks.status` is a `task_status` enum defaulting to `'to_do'` — set it explicitly to match the column.
