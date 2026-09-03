---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS (Supabase) column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back — blog_topics, blog_posts, blog_auto_runs, tasks, and task_assignees. Read during Draft → QA → Publish and at run-log open/close, per routine.md.
---

# SpearlanceOS Blog Pipeline — write-back reference

Project id **`chikljxwgiskyjsnjelf`** (production — confirmed via `list_branches`: `main` /
`chikljxwgiskyjsnjelf` has `is_default: true`). Everything here is `public` schema.

> **Provenance:** this file was reconstructed from live `information_schema` introspection plus the
> actual rows written by prior Auto Blogger runs (queried 2026-09-03). It did not exist anywhere in
> the repo's git history — `routine.md` has referenced it since the multi-client rewrite, but at
> least one earlier run (the CLE Remodeling / "Fall Home Maintenance Checklist" run, 2026-09-02) hit
> the same gap and noted it in `blog_auto_runs.research_summary`. If real column defaults or enum
> values ever drift from what's below, re-derive from `information_schema.columns` and `enum_range()`
> rather than trusting this file blindly — it's a convenience cache, not the source of truth.

## 1. Open the run log — `blog_auto_runs`

Insert **before** any research/drafting for the client, at the very start of Step 0 (per client):

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type`: `'scheduled'` for the normal hourly cron run. Use `'manual'` only when invoked ad hoc
  for a single `client_id` (routine.md Step 0b dry-run case).
- Keep the returned `id` as `<run_id>` — every other write this client-run makes references it.
- Columns: `id` (uuid, default `gen_random_uuid()`), `client_id` (uuid, required), `triggered_at`
  (timestamptz, default `now()`), `trigger_type` (text, required), `status` (text, default
  `'running'`), `topics_generated`/`articles_generated`/`articles_passed_gate`/`articles_flagged`
  (int, default 0 each — increment as you go or set on close), `research_summary` (jsonb, nullable),
  `completed_at` (timestamptz, nullable), `error_log` (text, nullable), `created_at` (timestamptz,
  default `now()`).

## 2. Topic selection / generation — `blog_topics`

Columns actually in use: `id`, `client_id`, `topic_title`, `description`, `summary` (both
`description` and `summary` exist — `routine.md`'s SQL uses `summary`; either is fine, prefer
`summary` for consistency with prior runs), `keywords` (text[]), `avatar_id`, `priority` (text,
default `'medium'`), `status` (text, default `'idea'`), `blog_post_id`, `ai_generated` (bool),
`created_at`, `strategy_batch_id`, `suggested_publish_date` (date), `category`.

**Live `status` values** (observed distribution): `idea` (backlog, not yet used), `in_progress`
(claimed by a run this cycle), `published` (turned into a post — not necessarily live, see §3).
There is no `'failed'` topic status; a topic that doesn't pan out just stays `in_progress` or gets a
fresh row next time — don't invent a new enum value here.

**4a — using an existing calendar topic:** mark it claimed immediately so a concurrent/later tick
doesn't double-pick it:

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
```

**4b — generating a new topic**, insert then keep the id:

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES
  ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>']::text[], '<category>',
   true, 'in_progress', current_date)
RETURNING id;
```

**On successful publish (draft created in Duda)**, close the topic out:

```sql
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```

## 3. The article record — `blog_posts`

This is Mila's actual draft/schedule/publish queue — one row per article created in Duda.

Columns: `id`, `client_id`, `title`, `slug`, `meta_description`, `content` (the HTML, optional to
store in full), `excerpt`, `focus_keyword`, `seo_score`, `readability_score`, `avatar_id`,
`topic_category`, `target_keywords` (text[]), `featured_image_url`, `featured_image_alt`,
`body_images` (jsonb), `status` (text, default `'draft'`), `scheduled_for`, `published_at`,
`duda_page_id`, `duda_publish_url`, `ai_model` (default `'claude-sonnet-4-5'` — set to whatever the
running Claude Routine model actually is, or leave the default), `generation_prompt`,
`generation_metadata` (jsonb), `created_by`, `created_at`/`updated_at` (default `now()`),
`last_edited_by`, `topic_id`, `scheduled_publish_date`, `images_generated` (bool), `auto_run_id`,
`quality_scores` (jsonb — put the full doctrine QA verdict here), `rejection_reason`,
`revision_count` (default 0).

**Live `status` values**: `draft` (268 rows — what the routine always inserts), `scheduled` (7),
`published` (2) — both of the latter are Mila's doing, never the routine's. **Always insert
`status = 'draft'`.**

```sql
INSERT INTO public.blog_posts
  (client_id, title, slug, meta_description, focus_keyword, target_keywords, status,
   duda_page_id, duda_publish_url, topic_id, auto_run_id, quality_scores)
VALUES
  ('<client_id>', '<title>', '<slug>', '<meta_description>', '<primary_keyword>',
   ARRAY['<kw1>','<kw2>']::text[], 'draft',
   '<duda_post_id>', '<duda_preview_or_editor_url>', '<topic_id>', '<run_id>',
   '<jsonb QA verdict>'::jsonb)
RETURNING id;
```

`quality_scores` shape used by prior runs (free-form jsonb, but keep this structure so Mila's
tooling and future runs can grep it consistently):

```json
{
  "verdict": "SHIP",
  "word_count": 1637,
  "h2_count": 8,
  "internal_links": 7,
  "no_h1_in_body": true,
  "unsourced_stats": 0,
  "external_sources_cited": 5,
  "sources": ["https://...", "..."],
  "faq_schema_note": "Article schema auto-applied by Duda; FAQ JSON-LD needs manual add via Duda Settings > Custom HTML"
}
```

## 4. Review task for Mila — `tasks` + `task_assignees`

Only when a draft was actually created. Mila's profile (`public.profiles`):
`id = '4b9df73e-bc38-49ef-840e-75fdb0b29c29'`, `email = 'mila@spearlance.com'`. Re-look-up by email
rather than hardcoding, in case the id ever changes:

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';
```

Find the client's **`to_do`** column (`task_columns` is per-client, keyed `to_do` / `in_progress` /
`done`):

```sql
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

`tasks` columns actually populated by the routine: `client_id`, `title`, `description`, `status`
(task_status enum: `to_do` | `in_progress` | `done` — default `to_do`), `priority` (task_priority
enum: `low` | `normal` | `high` | `urgent` — routine always uses `normal`), `assignee_user_id` (set
directly to Mila's id, in addition to the `task_assignees` row below — prior runs set both),
`creator_user_id` (Mila's id, or leave null), `column_id` (the client's `to_do` column id from
above). Title/description convention observed across every prior run — keep it:

```sql
INSERT INTO public.tasks
  (client_id, title, description, status, priority, assignee_user_id, column_id)
VALUES (
  '<client_id>',
  'Review draft: <post title>',
  'Auto Blogger created a new draft for <client name>. Duda draft: <preview_or_editor_url> (post id <duda_post_id>, site <site_id>, unpublished). Doctrine QA verdict: <SHIP>. Target keyword: <keyword>. Sources cited: <short source list>. Follow-up: FAQ JSON-LD needs to be added manually via Duda Settings > Custom HTML (Duda cannot auto-apply FAQ schema via the API).',
  'to_do',
  'normal',
  '<mila_profile_id>',
  '<client_to_do_column_id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_profile_id>');
```

## 5. Close the run log — `blog_auto_runs`

Exactly one close per opened row, whichever branch the client ended on:

**Draft created:**

```sql
UPDATE public.blog_auto_runs
SET status = 'completed',
    topics_generated = <0 or 1>,
    articles_generated = 1,
    articles_passed_gate = 1,
    research_summary = '<jsonb: topic, target keyword/intent, sources, duda_post_id, duda_preview_url, notes>'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';
```

**Branch stopped instead (QA fail, no fuel, no strategy row, doctrine halt, escalation, duplicate
topic, etc.):**

```sql
UPDATE public.blog_auto_runs
SET status = 'failed',
    articles_flagged = 1,
    error_log = '<why: e.g. "QA FAIL: 2 unsourced stats in draft, cut and still below 1000 words">',
    research_summary = '<jsonb: whatever was evaluated, even though nothing shipped>'::jsonb,
    completed_at = now()
WHERE id = '<run_id>';
```

Live `status` values in `blog_auto_runs`: only `completed` and `failed` are ever set on close (
`running` is the transient open state). Don't invent a third value — a "nothing was due" tick never
opens a run log row at all (Step 0d ends the run before Step 0's "open the run log" step), so it
never needs one of these closes either.

`trigger_type` seen in production: `scheduled` (788 of 788 rows as of 2026-09-03 — `manual` exists
as a valid value per routine.md but hasn't been used yet in this project).
