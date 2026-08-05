---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back sequence (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Use this during Auto Blogger routine.md's Publish / Record-the-run steps for every client, and when opening the per-client blog_auto_runs log at the start of processing.
---

# SpearlanceOS blog pipeline write-back

Project ref (production, confirmed via `list_branches` — `is_default: true`): **`chikljxwgiskyjsnjelf`**.
Schema below was pulled live via `mcp__Supabase__list_tables(verbose=true)` / `information_schema` — re-verify with
`list_tables` if a write errors on an unknown column, schema drift is possible.

This file has repeatedly failed to survive on `main` — many prior Auto Blogger sessions wrote it to their own
throwaway branch and pushed, but no PR was ever opened (each scheduled run gets a fresh session/branch, so the
fix never merges and the next run rediscovers the same gap from a clean `main` checkout). If you are a human
reading this: there is a fix sitting on some `claude/awesome-fermi-*` branch — check `list_pull_requests` /
open branches and merge one.

## 1. Open the run log (start of each client's processing, before Step 1)

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')  -- trigger_type='manual' only for an ad hoc single-client invocation
RETURNING id;
```

Keep the returned `id` as `<run_id>` for the rest of this client's processing. `status` defaults to `'running'`;
you only ever write it once more, at close-out (below).

## 2. Topic selection write-back (Step 4)

**4a — using an existing calendar topic:** mark it in progress the moment you commit to it:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**4b — generating a new topic:** insert it, capture the new id:

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, priority, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', 'medium', true, 'in_progress', current_date)
RETURNING id;
```

Notes on `blog_topics` columns (do not guess these — they differ from what routine.md's prose implies):
- The description/brief column is **`summary`** (there is also a separate, mostly-unused `description` column —
  use `summary`).
- `priority` is **`text`**, not an enum (`'low'|'medium'|'high'` by convention; default `'medium'`).
- `status` is **`text`**, default `'idea'`. Values used by the pipeline: `'idea'` (untouched), `'in_progress'`
  (claimed this run), `'published'` (turned into a post — not necessarily live, see below).
- `blog_post_id` links back once a post exists (set implicitly via the `blog_posts.topic_id` FK from the other
  side; you don't need to UPDATE `blog_topics.blog_post_id` directly — `blog_posts.topic_id` is the FK of record,
  confirmed via `list_tables` foreign_key_constraints. Still fine to also set `blog_topics.blog_post_id` for
  convenience if you want it queryable from the topic row.)

## 3. If the draft clears QA: create the Duda post, then write back in this order

### 3a. `blog_topics` — close out the topic

```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

("published" here means "turned into a post", matching the table's existing convention — it does **not** mean
the Duda post is live. The routine never publishes live; every Duda post is left unpublished/draft.)

### 3b. `blog_posts` — the actual draft/review record Mila works from

```sql
INSERT INTO public.blog_posts
  (client_id, topic_id, auto_run_id, title, slug, meta_description, content, excerpt, focus_keyword,
   topic_category, target_keywords, featured_image_url, featured_image_alt, status, duda_page_id,
   duda_publish_url, quality_scores, ai_model)
VALUES
  ('<client_id>', '<topic_id>', '<run_id>', '<title>', '<url-slug>', '<meta description>', '<full HTML content>',
   '<short excerpt>', '<focus keyword>', '<category>', ARRAY['kw1','kw2'], '<thumbnail/main image URL used>',
   '<alt text>', 'draft', '<duda post id from create_blog_post response>', '<duda preview/editor URL>',
   '<doctrine QA verdict as jsonb>'::jsonb, 'claude-sonnet-4-5')
RETURNING id;
```

Hard rule: `status` is always **`'draft'`** on insert — never `'scheduled'` or `'published'`, that's Mila's call.
`quality_scores` is `jsonb`, free-form — put the doctrine compliance verdict there (e.g.
`{"verdict":"SHIP","word_count":1650,"h2_count":7,"sources_cited":[...],"hard_gates":{...}}`); later runs read this
back for context, so keep it structured and useful, not just `"SHIP"`.

### 3c. `blog_auto_runs` — close out the run log opened in step 1

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = 1, articles_generated = 1, articles_passed_gate = 1, articles_flagged = 0
WHERE id = '<run_id>';
```

If the branch was stopped instead (QA fail, no fuel, no strategy row, doctrine halt, escalation, genuinely
saturated topic space, etc.):

```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, specific enough to read cold next run>'
WHERE id = '<run_id>';
```

Every opened run log gets closed exactly once, one way or the other, before moving to the next client. Write a
real, specific `error_log` even on a routine "nothing to do" outcome — the next run (and Garrett, via the
report) reads this table as the system of record for what each tick actually did. Vague logs like `"declined"`
are useless; state what was checked, what was found, and why it didn't clear the bar.

### 3d. `tasks` + `task_assignees` — only when a draft was actually created

Mila's user id (`profiles.email = 'mila@spearlance.com'`) — look it up fresh rather than hardcoding, in case it
ever changes:

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Each client has its own `task_columns` rows (not shared/global) with a default `'to_do'` `key`:

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

Then:

```sql
INSERT INTO public.tasks (client_id, title, description, status, column_id, related_asset_ids)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger created a draft for review. Duda preview: <duda preview url>. Keyword: <keyword>. Sources: <n> cited. Doctrine verdict: SHIP.',
  'to_do',
  '<to_do task_columns.id from above>',
  ARRAY['<blog_posts.id>']::uuid[]
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task id from above>', '<mila profile id>');
```

`tasks.status` is an **enum** (`task_status`: `to_do | in_progress | done`), unlike `blog_topics.status` /
`blog_posts.status` which are plain `text` — don't assume the same type across tables.
`tasks.priority` is also an enum (`task_priority`: `low | normal | high | urgent`, default `'normal'`) — leave
it at the default unless there's a specific reason to escalate.

## Quick reference — full column list (as of this vendoring)

**`blog_topics`**: id, client_id, topic_title, description, keywords[], avatar_id, priority (text), status
(text), blog_post_id, ai_generated, created_at, strategy_batch_id, summary, suggested_publish_date, category.

**`blog_posts`**: id, client_id, title, slug, meta_description, content, excerpt, focus_keyword, seo_score,
readability_score, avatar_id, topic_category, target_keywords[], featured_image_url, featured_image_alt,
body_images (jsonb), status (text, default 'draft'), scheduled_for, published_at, duda_page_id,
duda_publish_url, ai_model (default 'claude-sonnet-4-5'), generation_prompt, generation_metadata (jsonb),
created_by, created_at, updated_at, last_edited_by, topic_id, scheduled_publish_date, images_generated,
auto_run_id, quality_scores (jsonb), rejection_reason, revision_count.

**`blog_auto_runs`**: id, client_id, triggered_at, trigger_type, status (text, default 'running'),
topics_generated, articles_generated, articles_passed_gate, articles_flagged, research_summary (jsonb),
completed_at, error_log, created_at.

**`tasks`** (only the columns this pipeline touches — many more exist for the general task-board feature):
id, client_id, title, description, status (enum), priority (enum), column_id, related_asset_ids[], created_at,
updated_at.

**`task_assignees`**: id, task_id, user_id, created_at.

**`task_columns`**: id, client_id, name, key, mapped_status (enum), is_default, display_order — per-client,
not global; always filter by `client_id`.
