---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase write-back schema (column names, status values, copy-pasteable SQL) for the Auto Blogger's per-client run log, drafted-article record, and Mila review task. Read this before writing back to blog_auto_runs, blog_posts, tasks, or task_assignees.
---

# SpearlanceOS Blog Pipeline — Write-Back Reference

Project id: `chikljxwgiskyjsnjelf` (production — see `supabase-prod-guard`; this routine's writes to
these four tables are pre-authorized standing operation, not a one-off write needing fresh approval).

> Schema below was derived live via `information_schema.columns` on 2026-09-04 because this file was
> missing from the repo (referenced by the live Auto Blogger run prompt but never checked in — several
> prior ticks logged the same gap back to at least 2026-09-02). If the live schedule's routine prompt and
> this repo's `routine.md`/`CLAUDE.md` are ever reconciled, re-verify this file against whichever table
> set the reconciled version actually writes — the repo's current `routine.md` only documents a
> single-table (`blog_topics`-only) write-back, which predates what's below.

## 1. Open the run log — `blog_auto_runs` (one row per client processed, opened before any research)

Columns: `id` (uuid, pk), `client_id` (uuid, required), `triggered_at` (timestamptz, default now()),
`trigger_type` (text, required — `'scheduled'` for cron ticks, `'manual'` for a single-`client_id`
dry run), `status` (text, default `'running'`), `topics_generated` (int, default 0),
`articles_generated` (int, default 0), `articles_passed_gate` (int, default 0), `articles_flagged`
(int, default 0), `research_summary` (jsonb, nullable), `completed_at` (timestamptz, nullable),
`error_log` (text, nullable), `created_at` (timestamptz, default now()).

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id, created_at;
```

Keep the returned `id` as `<run_id>` — every subsequent write for this client references it. Close the
same row exactly once, whether or not a draft was produced:

```sql
-- success
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = <0 or 1>, articles_generated = 1, articles_passed_gate = 1, articles_flagged = 0,
    research_summary = '<jsonb: topic, sources_cited[], duda_post_id, blog_posts_id, task_id, notes>'::jsonb
WHERE id = '<run_id>';

-- stopped branch (QA fail / no fuel / no strategy / topic exhaustion / doctrine halt / escalation)
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, specific and reproducible>'
WHERE id = '<run_id>';
```

## 2. Mark the topic used — `blog_topics`

Relevant columns: `id`, `client_id`, `topic_title`, `description`, `summary`, `keywords` (text[]),
`category`, `priority` (text, default `'medium'` — free text in practice, e.g. `'high'`, `'5'`, `'normal'`
have all been seen; don't assume an enum), `status` (text, default `'idea'`), `blog_post_id` (uuid,
nullable — not reliably populated by older rows, don't depend on it for dedup), `ai_generated` (bool),
`suggested_publish_date` (date), `strategy_batch_id` (uuid, nullable).

Selecting the next usable topic (per routine Step 4a):

```sql
SELECT id, topic_title, summary, keywords, category, priority, suggested_publish_date, status, blog_post_id
FROM public.blog_topics
WHERE client_id = '<client_id>' AND status NOT IN ('published', 'in_progress')
  AND suggested_publish_date <= current_date
ORDER BY priority DESC, suggested_publish_date ASC;
```

`status = 'in_progress'` rows are excluded by that filter on purpose — they're either mid-process this
hour or orphaned from a prior tick that never closed them out; don't reselect them blind. If you generate
a brand-new topic instead (Step 4b), insert it with `status = 'in_progress'` and `ai_generated = true`.

When a topic turns into an actual drafted post, mark it done (this means "turned into a post," not
"live" — the table's existing convention):

```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

## 3. Record the draft — `blog_posts` (one row per article; Mila's review/schedule/publish queue)

Columns: `id`, `client_id`, `title`, `slug`, `meta_description`, `content` (full HTML — mirror what was
sent to Duda's `create_blog_post`), `excerpt`, `focus_keyword`, `target_keywords` (text[]),
`featured_image_url`, `featured_image_alt`, `body_images` (jsonb, nullable), `status` (text, default
`'draft'` — **always insert as `'draft'`**, never `'scheduled'`/`'published'`, that's Mila's call),
`scheduled_for` / `scheduled_publish_date` (nullable, hers to set), `published_at` (nullable),
`duda_page_id`, `duda_publish_url`, `ai_model` (text, default `'claude-sonnet-4-5'` — set to whatever
model actually generated the draft), `generation_prompt` / `generation_metadata` (jsonb, optional),
`topic_id` (uuid), `auto_run_id` (uuid — the `<run_id>` from step 1), `images_generated` (bool, default
false — images are Duda-stock for now per project doctrine), `quality_scores` (jsonb — the doctrine QA
verdict), `rejection_reason` (text, nullable), `revision_count` (int, default 0).

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt, focus_keyword, target_keywords,
  featured_image_url, featured_image_alt, status, duda_page_id, duda_publish_url,
  ai_model, topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<slug matching the Duda post path>', '<meta description>',
  '<full HTML body>', '<1-2 sentence excerpt/summary>', '<primary keyword>',
  ARRAY['<keyword1>','<keyword2>', ...],
  '<main image URL used in Duda>', '<alt text>',
  'draft', '<duda post id>', '<full public URL, live-or-not>',
  'claude-sonnet-5', '<topic_id>', '<run_id>',
  '<jsonb: verdict, word_count, h2_count, internal_links, external_source_links, sources[], notes>'::jsonb
) RETURNING id;
```

## 4. Assign Mila a review task — `tasks` + `task_assignees` (only when a draft was created)

`tasks` columns of note: `id`, `client_id`, `title`, `description`, `status` (enum `task_status`,
default `'to_do'`), `priority` (enum `task_priority`, default `'normal'`), `assignee_user_id`,
`creator_user_id`, `column_id` (uuid — FK to `task_columns`, drives which Kanban column it renders in),
`related_asset_ids` (uuid[], default `{}` — link the new `blog_posts.id` here).

`task_columns` is **per-client** — look up that client's `'to_do'` column, don't hardcode an id:

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

Look up Mila's user id once per run (stable across clients, cheap to re-check):

```sql
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';
```

Then:

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id, related_asset_ids)
VALUES (
  '<client_id>', 'Review draft: <post title>',
  '<Duda preview/editor link, post id + path, QA verdict summary, any flags (e.g. missing FAQ schema needs manual Duda Custom HTML add, or an orphaned duplicate draft found alongside this one)>',
  'to_do', 'normal', '<mila_user_id>', '<to_do_column_id>', ARRAY['<blog_posts.id>']::uuid[]
) RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id) VALUES ('<task id>', '<mila_user_id>');
```

## Known data-quality issues to watch for (not this skill's job to fix, but worth flagging when seen)

- **Orphaned Duda drafts**: a post exists in Duda (`list_blog_posts`) with no matching `blog_posts` row
  (`duda_page_id` never recorded) and its `blog_topics` row is still `status = 'idea'`. This happens when
  a tick creates the Duda post but a DB write (or the whole tick) fails before the write-back completes.
  `get_blog_post` only returns metadata, not the HTML body, so an orphan's content can't be verified
  without either publishing it (don't) or regenerating it. Flag it in the run's `blog_auto_runs.error_log`
  and the Mila task rather than silently trusting or silently deleting it — `delete_blog_post` requires
  explicit human confirmation per its own tool contract, which an unattended run cannot supply.
- **`priority` and `status` are free text, not enforced enums** on `blog_topics` — don't write code that
  assumes a fixed value set.
