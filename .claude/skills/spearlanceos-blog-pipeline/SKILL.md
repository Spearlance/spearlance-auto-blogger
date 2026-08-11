---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS Supabase column names, status values, and copy-pasteable SQL for the Auto Blogger's write-back tables (blog_auto_runs, blog_topics, blog_posts, tasks, task_assignees). Read whenever routine.md's Publish or "Open the run log" steps say to write to SpearlanceOS. Project id chikljxwgiskyjsnjelf.
---

# SpearlanceOS Blog Pipeline — Write-Back Reference

This skill exists so the Auto Blogger doesn't have to re-derive table schemas from
`information_schema.columns` every run. It documents the five write-back operations `routine.md`
performs, in order, with the exact columns and constraints as of 2026-08-11. If a write fails on a
constraint not listed here, the schema has likely changed — re-check with `execute_sql` against
`information_schema.columns` / `pg_constraint` and update this file.

Project id: **`chikljxwgiskyjsnjelf`**. All writes are scoped to the five tables below; everything else
in SpearlanceOS is read-only for this routine (see `routine.md` and `CLAUDE.md`).

## 1. Open the run log — `blog_auto_runs`

Insert this **before** any research or drafting, one row per client processed this tick.

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```

- `trigger_type` check constraint: `'scheduled'` | `'manual'` (manual only for ad hoc single-client runs).
- `status` check constraint: `'running'` | `'completed'` | `'failed'` — **no `'in_progress'` or other value.**
- Keep the returned `id` as `<run_id>` for the rest of this client's processing.

Other columns (fill in when closing out, step 4 below): `topics_generated`, `articles_generated`,
`articles_passed_gate`, `articles_flagged` (all `integer`), `research_summary` (`jsonb`, freeform —
useful for stashing sourcing mode, Duda post id/URL, dedup notes), `completed_at` (`timestamptz`),
`error_log` (`text`).

## 2. Topic — `blog_topics`

If generating a new topic (routine.md Step 4b):

```sql
INSERT INTO public.blog_topics (client_id, topic_title, summary, keywords, category, priority, status, ai_generated, suggested_publish_date, blog_post_id)
VALUES ('<client_id>', '<title>', '<summary>', ARRAY['kw1','kw2'], '<category>', 'medium', 'in_progress', true, current_date, NULL)
RETURNING id;
```

- `status` default `'idea'`; routine uses `'in_progress'` while drafting, `'published'` once turned into
  a post (means "turned into a post," not "live on the web" — the table's existing convention).
- `priority` is `text`, default `'medium'` (not an enum in the DB — `'low'`/`'medium'`/`'high'` seen in
  practice).
- There is both a `summary` (`text`) and a `description` (`text`) column; routine.md's convention is to
  fill `summary`. `keywords` is a `text[]` — use `ARRAY[...]`, not a JSON array literal.
- If reusing an existing due topic (Step 4a) instead, skip this insert and just `UPDATE ... SET status =
  'in_progress' WHERE id = '<topic_id>'`.

After the `blog_posts` row exists (step 3), come back and:

```sql
UPDATE public.blog_topics SET blog_post_id = '<blog_posts.id>', status = 'published'
WHERE id = '<topic_id>';
```

## 3. Article — `blog_posts`

One row per drafted article, **always** `status = 'draft'` — never `'scheduled'` or `'published'`; that
transition is Mila's call, made in her own tooling, not this routine's.

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, focus_keyword, topic_category,
  target_keywords, status, duda_page_id, duda_publish_url, topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<duda-post-path-slug>', '<meta description>',
  '<full HTML or a pointer to the Duda post id if content is large>',
  '<primary keyword>', '<category>', ARRAY['kw1','kw2'], 'draft',
  '<duda create_blog_post response id>', '<full public URL from Duda>',
  '<blog_topics.id>', '<blog_auto_runs.id>',
  '{"verdict": "SHIP", "word_count": 1234, ...}'::jsonb
)
RETURNING id;
```

- `slug` is `NOT NULL` — use the `path` Duda returns (from `create_blog_post`'s response, or
  `get_blog_post`), not a slug you invent yourself; they must match for `duda_publish_url` to resolve.
- `content` is `text`, not size-limited by a DB constraint, but the full HTML can be large; storing it
  verbatim is fine — just don't let it block on Duda API quirks (e.g. base64-encoded content in some
  tool responses — store the human-readable HTML you authored, not the encoded blob).
- `quality_scores` (`jsonb`) is a good place to log the doctrine QA verdict, word count, link count,
  sourcing mode (normal / degraded), and any external citation URLs — the QA record for this specific
  draft.
- `rejection_reason` (`text`) — fill this if a draft is created but fails QA and can't be auto-fixed or
  deleted (no Duda content-edit tool exists here, and deleting requires human confirmation unattended
  runs don't have). Leave the Duda draft in place, write the exact fix list here, and flag it for Mila.

## 4. Close the run log — `blog_auto_runs`

```sql
UPDATE public.blog_auto_runs SET
  status = 'completed',              -- or 'failed' if the branch was stopped instead
  topics_generated = 1,              -- 0 if an existing topic was reused
  articles_generated = 1,            -- 0 if no draft was created — this drives the Step 0c due-check
  articles_passed_gate = 1,          -- 0 if QA failed
  articles_flagged = 0,
  completed_at = now(),
  error_log = NULL,                  -- set to the exact reason if status = 'failed'
  research_summary = '{"...": "..."}'::jsonb
WHERE id = '<run_id>';
```

**`articles_generated` must accurately reflect whether a draft was actually created.** `routine.md`'s
Step 0c "already served today" gate queries this exact field to decide whether an hourly-due client has
already gotten its slot today — get this wrong and either the runaway-duplication bug (over-counting
never happens, but under-counting to 0 when a draft *was* made would let the same client redraft again
next tick) or false starvation (a client wrongly marked served) results.

## 5. Review task for Mila — `tasks` + `task_assignees`

Only when a draft was actually created.

```sql
SELECT u.id FROM auth.users u JOIN public.profiles p ON p.id = u.id WHERE u.email = 'mila@spearlance.com';
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';

WITH new_task AS (
  INSERT INTO public.tasks (client_id, title, description, status, priority, column_id, assignee_user_id)
  VALUES ('<client_id>', 'Review draft: <post title> (<client name>)', '<description with QA summary + Duda links>',
          'to_do', 'normal', '<task_columns.id>', '<mila_user_id>')
  RETURNING id
)
INSERT INTO public.task_assignees (task_id, user_id)
SELECT id, '<mila_user_id>' FROM new_task;
```

- `tasks.status` and `tasks.priority` are enum types (`task_status`, `task_priority`), not free text —
  use `'to_do'` / `'normal'` as shown; don't guess other spellings.
- Mila's user id as of 2026-08-11: `4b9df73e-bc38-49ef-840e-75fdb0b29c29` (`mila@spearlance.com`) — still
  worth resolving by email each run rather than hardcoding, in case it ever changes.
- `task_columns` is per-client (`client_id` + `key = 'to_do'`); don't assume a global column id.

## Known gotchas

- **`clients.website_url` can point to the wrong site.** A client's live Duda domain
  (`get_site_details(site_name).canonical_url`) sometimes differs from `clients.website_url` in
  SpearlanceOS — they can be entirely different sites (e.g. a parent company's marketing site vs. the
  specific brand site this `site_id` actually publishes to). Always trust `canonical_url` for sitemap /
  internal-link discovery.
- **Topic saturation.** A client with a long-running, frequent cadence can exhaust its realistic topic
  space (dozens of near-variant titles already published). Check `list_blog_posts` titles for close
  overlap, not just exact-title dedup, before generating a new topic.
