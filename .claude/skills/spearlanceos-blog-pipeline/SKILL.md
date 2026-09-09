---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back tables (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read during the Publish and "open the run log" steps of routine.md before writing anything back to SpearlanceOS.
---

# SpearlanceOS blog pipeline: write-back reference

Project id: `chikljxwgiskyjsnjelf` (production -- see `supabase-prod-guard`; the Auto Blogger's writes
to exactly these five tables, in the patterns below, are the routine's pre-authorized, designed
behavior per `routine.md` / `CLAUDE.md`. Nothing else in SpearlanceOS is writable from this routine.)

Schema below was verified against `information_schema.columns` / `pg_enum` on 2026-09-09. Re-verify if
a write ever fails on an unexpected column -- do not silently guess around an error.

## Why this file exists (read this before "fixing" it again)

Every scheduled run works on its own disposable branch. If this file only gets fixed on that branch and
the branch is never merged to `main`, the *next* scheduled run reclones stale `main`, finds the file
missing again, and "fixes" it again on yet another throwaway branch -- forever. As of 2026-09-09 there
were 56 unmerged `claude/*` branches on the remote and 15+ prior recreations of this exact file, none of
which reached `main`. **If you are recreating this file, that is a symptom, not the fix.** The fix is
for a human to merge one of these branches into `main`. Say so plainly in the end-of-run report.

## 1. `blog_auto_runs` -- one row per client per tick

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, auto (`gen_random_uuid()`) |
| `client_id` | uuid | NOT NULL |
| `triggered_at` | timestamptz | default `now()` |
| `trigger_type` | text | NOT NULL, no default/constraint observed. Use `'scheduled'` for the normal hourly Routine; `'manual'` only for an ad hoc single-`client_id` invocation (Step 0b). Historically only `'scheduled'` has been used. |
| `status` | text | default `'running'`. Set to `'completed'` (draft created) or `'failed'` (branch stopped) when you close the row. |
| `topics_generated` | integer | default 0 |
| `articles_generated` | integer | default 0 |
| `articles_passed_gate` | integer | default 0 |
| `articles_flagged` | integer | default 0 |
| `research_summary` | jsonb | optional free-form summary of sources/keywords used |
| `completed_at` | timestamptz | set when you close the row |
| `error_log` | text | required when `status='failed'`; explain why the branch stopped |
| `created_at` | timestamptz | default `now()` |

Open (top of client processing):

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type)
VALUES ('<client_id>', 'scheduled')
RETURNING id;
```

Close on success:

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = <n>, articles_generated = <n>, articles_passed_gate = <n>,
    research_summary = '<jsonb>'::jsonb
WHERE id = '<run_id>';
```

Close on stop (no draft produced):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why>'
WHERE id = '<run_id>';
```

If the whole client branch resolves in one step (e.g. guard-blocked before any research), a single
INSERT with `status`/`completed_at`/`error_log` already set is fine -- no need for a separate open/close
round trip when nothing happened in between.

## 2. `blog_posts` -- the draft/schedule/publish record Mila works from

**Hard gate:** `trg_one_blog_draft_per_client_per_day` (BEFORE INSERT trigger) blocks a second
`blog_posts` row for the same `client_id` on the same calendar day. **Check for an existing row dated
today before doing any research**, not just before the INSERT -- finding out after drafting wastes the
whole client branch:

```sql
SELECT count(*) FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at::date = current_date;
-- if > 0, this client already has today's draft. Stop the branch (status='failed' in blog_auto_runs,
-- error_log explaining the guard), do not draft.
```

Required (NOT NULL, no default) columns: `client_id`, `title`, `slug`. Everything else is nullable.
`slug` has no generator here -- derive one yourself (kebab-case the title) since Duda's own post path is
a separate field on the Duda side, not copied automatically.

| column | type | notes |
|---|---|---|
| `id` | uuid | PK, auto |
| `client_id` | uuid | NOT NULL |
| `title` | text | NOT NULL |
| `slug` | text | NOT NULL -- kebab-case the title yourself |
| `meta_description` | text | |
| `content` | text | the drafted HTML/body |
| `excerpt` | text | |
| `focus_keyword` | text | the target keyword from Step 4c |
| `seo_score` / `readability_score` | numeric | optional, if you scored the draft |
| `avatar_id` | uuid | leave null unless you resolved a specific target-avatar row |
| `topic_category` | text | matches `blog_topics.category` |
| `target_keywords` | text[] | |
| `featured_image_url` / `featured_image_alt` | text | |
| `body_images` | jsonb | optional |
| `status` | text | default `'draft'` -- **always leave as `'draft'`**. Never insert `'scheduled'` or `'published'`; that's Mila's call. |
| `scheduled_for` / `published_at` | timestamptz | leave null (Mila sets these) |
| `duda_page_id` | text | the Duda post id from `create_blog_post`'s response |
| `duda_publish_url` | text | the Duda preview/editor URL |
| `ai_model` | text | default `'claude-sonnet-4-5'`; leave default unless you know otherwise |
| `generation_prompt` / `generation_metadata` | text / jsonb | optional provenance |
| `topic_id` | uuid | FK to the `blog_topics` row used/created in Step 4 |
| `scheduled_publish_date` | date | the topic's `suggested_publish_date`, if useful |
| `auto_run_id` | uuid | FK to this client's `blog_auto_runs.id` |
| `quality_scores` | jsonb | the doctrine QA verdict, e.g. `{"verdict":"SHIP","word_count":1650,"h2_count":5,"internal_links":7,"sourced_stats":4,"notes":"..."}` |
| `rejection_reason` | text | only if you're logging a non-shipped attempt |
| `revision_count` | integer | default 0 |

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, excerpt, focus_keyword,
   topic_category, target_keywords, featured_image_url, featured_image_alt,
   duda_page_id, duda_publish_url, topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<kebab-case-slug>', '<meta description>', '<html content>',
   '<excerpt>', '<focus keyword>', '<category>', ARRAY['<kw1>','<kw2>'],
   '<featured image url>', '<alt text>',
   '<duda_post_id>', '<duda preview url>', '<topic_id>', '<run_id>',
   '{"verdict":"SHIP", "...": "..."}'::jsonb)
RETURNING id;
```

## 3. `blog_topics` -- mark the used/generated topic done

`status` observed values: `'idea'` (default), `'in_progress'`, `'published'`. There is no schema
constraint enforcing this list, but stick to it -- these are the values the rest of the pipeline (and
past runs) expect. `'published'` here means "turned into a post" (a `blog_posts` draft exists), not
"live" -- the table's existing convention, unrelated to Duda's publish state.

Mark a used/generated topic done once its `blog_posts` row is created:

```sql
UPDATE public.blog_topics
SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

If a topic was picked up (`in_progress`) but the branch stopped before a draft was created, revert it
so a future run can reuse it rather than leaving it stuck:

```sql
UPDATE public.blog_topics SET status = 'idea' WHERE id = '<topic_id>';
```

## 4. `tasks` + `task_assignees` -- Mila's review task (only when a draft was created)

`task_columns` is per-client; look up that client's `'to_do'` column id (do not hardcode a global id):

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

`tasks.status` is the enum `task_status`: `to_do | in_progress | blocked | done | cancelled` (default
`to_do`). `tasks.priority` is enum `task_priority`: `low | normal | high | urgent` (default `normal`).
Mila's user id (`auth.users`, looked up by email once per run -- do not hardcode, re-resolve in case it
ever changes): `SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';`

```sql
INSERT INTO public.tasks (client_id, title, description, column_id, assignee_user_id, related_asset_ids)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger created a draft for review. Duda: <duda preview url>. Topic: <topic_title>. Keyword: <focus_keyword>.',
  '<to_do column_id for this client>',
  '<mila_user_id>',
  ARRAY['<blog_posts.id>']::uuid[]
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_user_id>');
```

(`tasks.assignee_user_id` and the `task_assignees` row are both populated above for compatibility with
whichever the SpearlanceOS UI reads; do not skip either.)

## Sequencing recap (per Publish step in routine.md)

1. `blog_topics` -> mark used/generated topic `'published'` with `blog_post_id`.
2. `blog_posts` -> insert the draft row, always `status='draft'`.
3. `blog_auto_runs` -> close the row opened at the top of this client's processing.
4. `tasks` + `task_assignees` -> only if a draft was created.

Write nothing else to SpearlanceOS. Never call its `blog-*` edge functions.
