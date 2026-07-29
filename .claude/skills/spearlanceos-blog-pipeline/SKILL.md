---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase write-back conventions (column names, status values, copy-pasteable SQL) for the Auto Blogger's per-client run: opening/closing blog_auto_runs, marking blog_topics done, inserting the blog_posts review record, and creating Mila's review task. Read this whenever the Auto Blogger playbook says "write back to SpearlanceOS" so the writes use the right tables/columns on the first try.
---

# SpearlanceOS blog pipeline — write-back reference

Reconstructed 2026-07-29 after this file was found missing from the repo (a prior run apparently
created it in a since-reclaimed ephemeral container but never committed it — several
`blog_auto_runs.error_log` entries from 2026-07-28 reference exactly this skill and Mila-review
workflow, confirming it existed before). **Recreate/update this file whenever its schema drifts from
reality** — don't let it go stale silently again. Project id: `chikljxwgiskyjsnjelf`.

This is the *current* write model: every Duda post is created as an **unpublished draft**, reviewed by
Mila, and scheduled/published by her — not the older `auto_blog_mode` queue/auto_publish + single
`blog_topics`-only write-back described in this repo's `routine.md`/`CLAUDE.md`. If you're running from
a scheduler-stored prompt that matches the "never publish live, Mila reviews everything" model, follow
*this* file; if `routine.md`'s older model is what you were actually handed, follow that instead and
flag the mismatch in your report — don't silently reconcile the two.

## Tables you write to (and only these)

`blog_topics`, `blog_posts`, `blog_auto_runs`, `tasks`, `task_assignees`. Everything else in
SpearlanceOS is read-only. Never call its `blog-*` edge functions.

## 1. Open the run log — first thing, before any research

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')  -- trigger_type = 'manual' only for an ad hoc single-client invocation
RETURNING id;  -- keep as <run_id>, used to close the row later
```
`status` defaults to `'running'`. Columns available for the close-out: `topics_generated`,
`articles_generated`, `articles_passed_gate`, `articles_flagged` (all default `0`), `research_summary`
(jsonb — free-form: topic, Duda post id/URL, sourcing mode notes), `error_log` (text, only on failure),
`completed_at`.

## 2. Look up Mila's user id and the client's "To Do" task column (once per client)

```sql
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```
`tasks.status` and `tasks.priority` are Postgres enum types (`task_status`, `task_priority`), not free
text — use the same string values already in use (`'to_do'`, `'normal'`, etc.) rather than guessing.

## 3. After the Duda draft is created — the write-back sequence

Do these four in order, every time a draft is produced. If the branch was stopped instead (QA fail, no
fuel, no strategy row, doctrine halt), skip straight to closing `blog_auto_runs` with `status='failed'`
and a descriptive `error_log`; don't insert `blog_topics`/`blog_posts`/`tasks` for a post that doesn't
exist.

### 3a. `blog_topics` — mark the topic used/created as done
If you generated the topic this run, `INSERT` it directly with `status='published'` (this table's
convention: `'published'` means "turned into a post," not "live on the client's site" — Duda's own
publish state is tracked separately in `blog_posts`/on the post itself). If you pulled an existing
`blog_topics` row in Step 4a of the playbook, `UPDATE` that row's `status` to `'published'` instead of
inserting a new one.

```sql
INSERT INTO public.blog_topics
  (client_id, topic_title, summary, keywords, category, priority, status, ai_generated, suggested_publish_date)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', 'medium', 'published', true, current_date)
RETURNING id;  -- keep as <topic_id>
```

### 3b. `blog_posts` — the actual record Mila works from
One row per drafted article. `status` is **always `'draft'`** here — never `'scheduled'` or
`'published'`; that transition is Mila's call, made in her own tooling, not this routine's.

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, excerpt, focus_keyword,
  topic_category, target_keywords, featured_image_url, featured_image_alt,
  status, duda_page_id, duda_publish_url, topic_id, auto_run_id,
  images_generated, quality_scores
) VALUES (
  '<client_id>', '<title>', '<duda-path-slug>', '<meta description>', '<1-2 sentence excerpt>',
  '<primary keyword phrase>', '<category>', ARRAY['kw1','kw2'], '<main_image url>', '<alt text>',
  'draft', '<duda post id>', '<full https URL the post will have once published>',
  '<topic_id>', '<run_id>', true,
  '{"verdict":"SHIP","word_count":0,"h2_count":0,"internal_links":0,"sourced_external_urls":0,
    "sources":[],"sourcing_mode":"tavily_primary|dataforseo_only_degraded",
    "doctrine_checklist":{"title":"PASS","description":"PASS","headings":"PASS","content":"PASS",
      "interlinking":"PASS","schema":"PASS_with_manual_faq_jsonld_followup","intent":"PASS","escalation":"none"}}'::jsonb
)
RETURNING id;
```
Other useful columns if you have the data: `avatar_id`, `readability_score`, `seo_score`,
`generation_prompt`, `generation_metadata`, `scheduled_publish_date`. `slug` and `title` are `NOT NULL`;
everything else in the insert above is optional but worth filling when known.

### 3c. `tasks` + `task_assignees` — put it in front of Mila
```sql
WITH new_task AS (
  INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id, related_asset_ids)
  VALUES (
    '<client_id>',
    'Review draft: <post title>',
    'Auto Blogger drafted this post for <client name>. Duda editor: <edit_site_url> — Post ID <duda post id>. <any manual follow-ups, e.g. FAQ JSON-LD>. Ready for review/schedule/publish.',
    'to_do', 'normal', '<mila_user_id>', '<to_do_column_id>', ARRAY['<blog_posts.id>']::uuid[]
  )
  RETURNING id
)
INSERT INTO public.task_assignees (task_id, user_id)
SELECT id, '<mila_user_id>' FROM new_task
RETURNING task_id;
```
`related_asset_ids` is a `uuid[]` — point it at the `blog_posts.id` you just created so the task links
straight to the review record.

### 3d. Close `blog_auto_runs`
```sql
UPDATE public.blog_auto_runs
SET status = 'completed', completed_at = now(),
    topics_generated = 1, articles_generated = 1, articles_passed_gate = 1, articles_flagged = 0,
    research_summary = '{"topic":"...","duda_post_id":"...","duda_preview_url":"...","sourcing_mode":"..."}'::jsonb
WHERE id = '<run_id>';
```
On a held/failed branch instead:
```sql
UPDATE public.blog_auto_runs
SET status = 'failed', completed_at = now(), error_log = '<why: QA fail / no fuel / no strategy row / etc.>'
WHERE id = '<run_id>';
```

## Notes from real runs

- `blog_posts.status` must stay `'draft'` even in `auto_publish` mode under the current (Mila-reviews-
  everything) model — see the header note above about which model you're actually operating under.
- Tavily can run out of its plan quota (`432` on both `tavily_search` and `tavily_extract`). When that
  happens and DataForSEO still answers, proceed sourcing via DataForSEO's `serp_organic_live_advanced` +
  `on_page_content_parsing` alone (record `sourcing_mode: "dataforseo_only_degraded"` in
  `research_summary`/`quality_scores`) rather than aborting the client.
- Don't hold a draft just because a client already has a large unpublished/backlog of drafts sitting in
  Duda — under the current playbook that's explicitly normal (Mila's queue to work through), not a signal
  to stop. Only hold for an actual doctrine QA failure, missing fuel, or a genuine data-integrity bug
  (e.g. a confirmed duplicate post) — and if it's the latter, flag it prominently rather than silently
  holding every subsequent tick without escalating loudly.
