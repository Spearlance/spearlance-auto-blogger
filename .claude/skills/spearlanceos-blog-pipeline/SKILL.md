---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase write-back for the Auto Blogger — the real column names, status values, and copy-pasteable SQL for blog_topics, blog_posts, blog_auto_runs, tasks, and task_assignees. Read this before writing back a topic status, a drafted article, a run log, or Mila's review task. Also documents the one-draft-per-client-per-day DB guard that every run MUST check before spending any research/sourcing calls.
---

# SpearlanceOS Blog Pipeline — write-back reference

All writes target project `chikljxwgiskyjsnjelf` (production — see `supabase-prod-guard`; this is a
standing, scoped exception: the Auto Blogger routine is durably authorized to write to exactly the five
tables below and nothing else). Schema below was read directly from `information_schema.columns` on
2026-08-26 — reread it yourself if a write ever errors on an unknown column; don't assume this file is
still byte-accurate forever.

## Hard gate: check the one-draft-per-day guard BEFORE any research

`public.blog_posts` has a live trigger:

```
trg_one_blog_draft_per_client_per_day BEFORE INSERT ON public.blog_posts
  FOR EACH ROW EXECUTE FUNCTION guard_one_blog_draft_per_client_per_day()
```

It rejects a second `blog_posts` insert for the same `client_id` on the same calendar day. **Run this
check immediately after opening the run log, before Step 3/Step 4 (reactive check, topic selection,
research, sourcing) for a due client:**

```sql
SELECT id, title, duda_page_id, created_at
FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at::date = current_date;
```

- **Row found →** this client already has today's draft. Do **not** run research, sourcing, drafting, or
  QA — every Tavily/DataForSEO call spent here is wasted, because any resulting Duda post can never be
  logged to `blog_posts` (the insert will be rejected) and becomes an orphaned, untracked draft sitting
  in Duda with no `tasks` row pointing Mila to it reliably. Close the run log for this client immediately
  (`status = 'completed'`, `articles_generated = 0`, `error_log` explaining the skip) and move to the next
  due client, if any, under this tick's `POSTS_PER_RUN` budget.
- **No row →** proceed normally.

This does not change *due-ness* (Step 0c's calendar test is still the only due-check) — it's a pre-work
gate that runs after a client is already known to be due, so a missed slot still reads as "already served
today," not "not due."

Known history (2026-08-24 through 2026-08-26): skipping this check before it was written into this file
caused roughly 15+ wasted full research/draft/QA cycles across CLE Remodeling, Invictus Snowfighters, and
Sunshine State, each leaving an orphaned `UNPUBLISHED` Duda draft with no `blog_posts` row. Those orphans
were left in place (no unattended-delete authority) and flagged for Mila/Garrett to clean up manually —
don't delete them yourself either; that's a judgment call for a human, not an automated cleanup.

## `blog_topics` — the topic queue

| column | type | notes |
|---|---|---|
| id | uuid | pk |
| client_id | uuid | |
| topic_title | text | required |
| summary | text | short summary shown to Mila / used for planning |
| description | text | rarely used; `summary` is the one the routine writes |
| keywords | text[] | |
| category | text | should match a `content_mix` key |
| priority | text | default `'medium'` |
| status | text | default `'idea'`; routine uses `'idea'` → `'in_progress'` → `'published'` (meaning "turned into a
  post," not "live" — matches the table's existing convention) |
| blog_post_id | uuid | set once a `blog_posts` row exists for this topic |
| ai_generated | boolean | `true` for routine-generated topics |
| suggested_publish_date | date | |
| avatar_id | uuid | optional, leave null unless known |
| strategy_batch_id | uuid | optional, leave null unless known |

**Generate + save a new topic (Step 4b):**

```sql
INSERT INTO public.blog_topics (client_id, topic_title, summary, keywords, category, ai_generated, status, suggested_publish_date)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['<kw1>','<kw2>']::text[], '<category>', true, 'in_progress', current_date)
RETURNING id;
```

**Mark a calendar topic in progress (Step 4a) / done after publish:**

```sql
UPDATE public.blog_topics SET status = 'in_progress' WHERE id = '<topic_id>';
-- after the draft is created:
UPDATE public.blog_topics SET status = 'published', blog_post_id = '<blog_post_id>' WHERE id = '<topic_id>';
```

## `blog_posts` — the drafted article / Mila's queue

Required: `client_id`, `title`, `slug`. Everything else is optional but the routine should fill what it
has. Relevant columns beyond the obvious: `topic_id`, `auto_run_id`, `quality_scores` (jsonb — store the
doctrine QA verdict), `status` (always `'draft'` from this routine — never `'scheduled'`/`'published'`),
`duda_page_id`, `duda_publish_url`, `focus_keyword`, `target_keywords` (text[]), `meta_description`,
`excerpt`, `topic_category`, `featured_image_url`, `featured_image_alt`.

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, excerpt, focus_keyword, target_keywords,
  topic_category, featured_image_url, featured_image_alt, status, topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<slug>', '<meta_description>', '<excerpt>', '<focus_keyword>',
  ARRAY['<kw1>','<kw2>']::text[], '<category>', '<image_url>', '<alt_text>', 'draft',
  '<topic_id>', '<auto_run_id>',
  '{"verdict":"SHIP","word_count":1500,"h2_count":6,"internal_links":8,"sourced_stats":3}'::jsonb
)
RETURNING id;
```

If the guard trigger rejects this insert (client already has a row today — shouldn't happen if you ran
the pre-check above, but treat it as authoritative if it fires anyway), do **not** retry or work around
it. Log the Duda draft in the `tasks` row instead (see below) so it isn't orphaned, and close the run log
noting the guard blocked the `blog_posts` write.

## `blog_auto_runs` — one row per client per tick

Open at the start of processing a due client:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

(`trigger_type = 'manual'` only for an ad hoc single-`client_id` invocation.) Close it once, at the end of
this client's processing:

```sql
UPDATE public.blog_auto_runs
SET status = 'completed', -- or 'failed' if the branch was stopped (QA fail, no fuel, no strategy, doctrine halt, escalation, or the pre-work guard skip above)
    topics_generated = <0 or 1>,
    articles_generated = <0 or 1>,
    articles_passed_gate = <0 or 1>,
    articles_flagged = <0 or 1>,
    error_log = '<why, if not a normal success>',
    completed_at = now()
WHERE id = '<run_id>';
```

Note: prior runs have used `status = 'completed'` (not `'failed'`) for the "already served today, guard
would block it" case, since that's a correct no-op, not an error — the tick did its job by recognizing
there was nothing to do. Keep doing that; reserve `'failed'` for an actual blocked/aborted branch.

## `tasks` + `task_assignees` — Mila's review notification

Only when a draft was actually created (in `blog_posts`, or as a Duda-only orphan the guard blocked from
being logged — see above). Look up Mila and the client's `to_do` column first:

```sql
SELECT id FROM public.profiles WHERE email = 'mila@spearlance.com';  -- 4b9df73e-bc38-49ef-840e-75fdb0b29c29 as of 2026-08-26
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

`task_columns.id` is **per-client** — do not hardcode a value from one client onto another.

```sql
INSERT INTO public.tasks (client_id, title, description, status, priority, assignee_user_id, column_id, external_source, external_ref)
VALUES (
  '<client_id>',
  'Review Auto Blogger draft: <title>',
  '<description with Duda editor/preview links, Duda post id, QA verdict summary, sources cited, and any follow-up (e.g. FAQ JSON-LD needs manual Duda Custom HTML add)>',
  'to_do', 'normal', '4b9df73e-bc38-49ef-840e-75fdb0b29c29', '<column_id>', 'auto_blogger', '<duda_post_id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '4b9df73e-bc38-49ef-840e-75fdb0b29c29');
```

`related_asset_ids` (uuid[]) may instead carry the `blog_posts.id` when one exists, in place of / in
addition to `external_ref`. Either linking style has been used; prefer `related_asset_ids` when a
`blog_posts` row exists, `external_source`/`external_ref` when the draft is Duda-only (guard-blocked).
