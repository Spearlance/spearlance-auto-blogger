---
name: spearlanceos-blog-pipeline
description: Exact column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back to SpearlanceOS (blog_auto_runs, blog_topics, blog_posts, tasks, task_assignees). Read by routine.md's Step 0 (open the run log) and the Publish step (write-back). Project chikljxwgiskyjsnjelf.
---

# SpearlanceOS Blog Pipeline — write-back reference

This file has historically gone missing from `main` (every scheduled run works off a fresh
disposable branch, and none had been merged back — so routine.md's references to this skill kept
resolving to nothing, and every run fell back to deriving the schema live via
`information_schema.columns`). This version was (re)written from that live introspection on
2026-09-11 against project `chikljxwgiskyjsnjelf` (confirmed production via `list_branches`: `main`
branch, `project_ref = chikljxwgiskyjsnjelf`, `is_default: true`). **If this file goes missing again,
re-derive it the same way** — query `information_schema.columns` for the five tables below before
guessing at column names.

**If you are a scheduled run and this file exists**: the missing-file problem may be fixed. Don't
assume it's stayed fixed — a fresh disposable branch that never merges will make it vanish again from
whatever `main` looks like to you.

## Known hard constraint: one draft per client per day

`public.blog_posts` has a trigger:

```
CREATE TRIGGER trg_one_blog_draft_per_client_per_day BEFORE INSERT ON public.blog_posts
FOR EACH ROW EXECUTE FUNCTION guard_one_blog_draft_per_client_per_day()
```

It rejects a second `blog_posts` insert for the same `client_id` on the same day. **routine.md's Step
0c/0d due-check does not check this** — it only checks `selected_days`. Consequence: if the hourly cron
fires more than once per hour (repeatedly observed — sometimes ~3-4x the intended rate), every tick
after a due client's first successful draft that day gets its `blog_posts` insert rejected. Two
concrete failure modes to guard against, both seen repeatedly in `blog_auto_runs.error_log`:

1. **Wasted spend**: research + drafting happens, then the insert fails. Avoid this by checking
   `blog_posts` for an existing row `WHERE client_id = ? AND created_at::date = current_date` *before*
   spending any Tavily/DataForSEO calls, not after drafting.
2. **Orphaned Duda draft**: worse — a full draft gets created in Duda (`create_blog_post` succeeds)
   and *then* the `blog_posts` insert is rejected by the trigger, leaving a real, unpublished Duda post
   with no `blog_posts` row, no `topic_id` link, and no Mila task. It's invisible to Mila's review
   queue. If you hit this, do the `blog_posts` insert (and topic/task write-back) for that Duda post
   the moment a same-day slot is free, rather than starting a new draft on top of it. If you can't
   verify the orphaned draft's actual body content (the Duda MCP's `get_blog_post` does not return
   post body/HTML — only metadata), don't fabricate a QA verdict for it; leave it and flag it for
   Garrett/Mila to review directly in the Duda editor.

Check before starting work on a client:

```sql
SELECT id, title, duda_page_id FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at::date = current_date;
```

## 1. Open the run log — `blog_auto_runs`

Insert at the very start of a client's processing, before any research:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type`: `'scheduled'` for the normal hourly cron; `'manual'` only when invoked ad hoc for a
  single `client_id`.
- `status` default is `'running'`; you close it out at the end (see below) as `'completed'` or
  `'failed'`.
- Columns: `id` (uuid, pk), `client_id` (uuid, required), `triggered_at` (timestamptz, default now()),
  `trigger_type` (text), `status` (text), `topics_generated` (int, default 0),
  `articles_generated` (int, default 0), `articles_passed_gate` (int, default 0),
  `articles_flagged` (int, default 0), `research_summary` (jsonb), `completed_at` (timestamptz),
  `error_log` (text), `created_at` (timestamptz).

## 2. Mark the topic used — `blog_topics`

When a draft is actually produced from a calendar or generated topic:

```sql
UPDATE public.blog_topics SET status = 'published' WHERE id = '<topic_id>';
```

`status = 'published'` here means "turned into a post" (per the existing table convention), not that
the post is live — every Duda draft stays unpublished regardless. Other observed `status` values:
`'idea'` (default, open/unused), `'in_progress'` (claimed by a run this tick but not yet written back
— set this if you start drafting from a calendar topic, so a concurrent/overlapping tick doesn't also
pick it up), `'published'` (done). Columns: `id`, `client_id`, `topic_title`, `description`, `keywords`
(array), `avatar_id`, `priority` (text, default `'medium'`), `status` (text, default `'idea'`),
`blog_post_id` (uuid, set once a post exists), `ai_generated` (bool), `created_at`, `strategy_batch_id`,
`summary`, `suggested_publish_date` (date), `category`.

If you generated a brand-new topic (Step 4b), insert it first, then use its `id` throughout:

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>'], '<category>', true, 'in_progress', current_date)
RETURNING id;
```

## 3. Insert the article — `blog_posts`

One row per drafted article, always `status = 'draft'` (never `'scheduled'` or `'published'` — that's
Mila's call):

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, content, focus_keyword, topic_category,
   target_keywords, featured_image_url, featured_image_alt, status, topic_id, auto_run_id,
   duda_page_id, duda_publish_url, quality_scores)
VALUES
  ('<client_id>', '<title>', '<url-slug>', '<meta description>', '<full HTML content>',
   '<primary keyword>', '<category>', ARRAY['<kw1>','<kw2>'], '<image url>', '<alt text>',
   'draft', '<topic_id>', '<auto_run_id>', '<duda post id>', '<duda preview/editor url>',
   '<jsonb QA verdict, e.g. {"doctrine_verdict":"SHIP","word_count":1650,"h2_count":6,"internal_links":7,"sourced_claims":4}'::jsonb)
RETURNING id;
```

Required (NOT NULL): `client_id`, `title`, `slug`. Everything else nullable. Other columns available
if useful: `excerpt`, `seo_score`, `readability_score`, `avatar_id`, `body_images` (jsonb),
`scheduled_for`, `published_at`, `ai_model` (defaults `'claude-sonnet-4-5'` — fine to leave as default
or set to whatever's actually running), `generation_prompt`, `generation_metadata` (jsonb),
`scheduled_publish_date`, `images_generated` (bool), `rejection_reason`, `revision_count`.

Remember the `id` this returns — it's `<blog_post_id>` below.

Also link the topic back, if not already inserted with `topic_id` above:

```sql
UPDATE public.blog_topics SET blog_post_id = '<blog_post_id>' WHERE id = '<topic_id>';
```

## 4. Close the run log — `blog_auto_runs`

Exactly once per opened row, whether or not a draft resulted:

```sql
-- success
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    articles_generated = 1, articles_passed_gate = 1,
    research_summary = '<jsonb summary of sources/keywords used>'::jsonb
WHERE id = '<run_id>';

-- stopped branch (QA fail, no fuel, no strategy row, doctrine halt, saturation/dedup decline, escalation)
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why, specific and re-checkable>'
WHERE id = '<run_id>';
```

Note: in practice, "failed" has been used for both genuine problems *and* benign, doctrine-correct
declines (guard-blocked because a draft already exists today, vertical saturated, no due topic). That
overloads the word — a human skimming `blog_auto_runs.status = 'failed'` can't tell "real failure" from
"correctly did nothing." Consider giving `error_log` a one-word prefix (`OK:`, `BLOCKED:`, `DECLINED:`,
`ERROR:`) until/unless a dedicated status value is added to the schema.

## 5. Create Mila's review task — `tasks` + `task_assignees`

Only when a draft was actually created (i.e., after step 3 succeeds). Look up Mila's user id and the
client's `to_do` column id (both are stable but re-verify per client — `task_columns` is per-client):

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

```sql
INSERT INTO public.tasks (client_id, title, description, status, assignee_user_id, column_id)
VALUES (
  '<client_id>',
  'Review blog draft: <post title>',
  'Auto Blogger drafted this post for review. Duda preview: <duda preview/editor url>. Doctrine verdict: <SHIP/etc>. Sources: <list>.',
  'to_do',
  '<mila_profile_id>',
  '<to_do_column_id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_profile_id>');
```

`tasks.status` and `tasks.priority` are enum types (`task_status`, `task_priority`), not free text —
stick to observed values (`'to_do'`, `'in_progress'`, `'done'`, `'cancelled'` for status). `tasks` has
many more optional columns (due_date, related_asset_ids, recurrence, qa_* fields, etc.) that don't apply
to this workflow — leave them at their defaults.

## Schema drift warning

This was hand-verified once (2026-09-11). Tables in a live product database change. If an insert fails
with an unknown-column or type error, re-run the introspection query below rather than guessing:

```sql
SELECT table_name, column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name IN ('blog_auto_runs','blog_posts','blog_topics','tasks','task_assignees')
ORDER BY table_name, ordinal_position;
```
