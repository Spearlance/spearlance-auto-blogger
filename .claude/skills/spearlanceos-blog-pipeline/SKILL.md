---
name: spearlanceos-blog-pipeline
description: Exact SpearlanceOS table/column map and copy-pasteable SQL for the Auto Blogger's write-back step (blog_topics, blog_posts, blog_auto_runs, tasks, task_assignees). Read this during the Publish and Record-the-run steps of the run playbook, whenever it says to write to one of those five tables.
---

# SpearlanceOS blog pipeline — write-back reference

Project id **`chikljxwgiskyjsnjelf`**. The routine writes to exactly five tables:
`blog_topics`, `blog_posts`, `blog_auto_runs`, `tasks`, `task_assignees`. Everything else is
read-only. Never call SpearlanceOS's `blog-*` edge functions.

## Hard guardrail: one `blog_posts` draft per client per day

A DB trigger, `guard_one_blog_draft_per_client_per_day`, **rejects a second `blog_posts` INSERT**
for a client that already has one dated today. This is real and has already caused repeated wasted
research/draft/QA cycles (each one burning Tavily/DataForSEO calls, creating a full doctrine-QA'd
draft, then failing at the write-back and leaving an **orphaned, untracked Duda draft** with no
`blog_posts`/`tasks` row).

**Check for this BEFORE doing any research, not after drafting:**

```sql
SELECT id, duda_page_id, created_at
FROM public.blog_posts
WHERE client_id = '<client_id>' AND created_at::date = current_date;
```

If a row comes back, this client is **not eligible for a new draft today** — skip it (see "Logging
a skip" below). Do not start sourcing, drafting, or QA for it. If several clients are due this tick,
run this check for all of them before picking which one (if any) to actually work.

If you ever do reach the write-back with a rejected INSERT (e.g. a race with another tick), do not
delete the Duda draft — deletion needs live human confirmation. Leave it unpublished, note it in the
run log as orphaned, and still file a Mila task for it if practical so it doesn't get lost.

## 1. `blog_topics` — mark the topic used

```sql
UPDATE public.blog_topics
SET status = 'published', blog_post_id = '<blog_posts.id>'
WHERE id = '<topic_id>';
```
`status='published'` here means "turned into a post," not "live" — matches the table's existing
convention. If the topic was newly generated this run (Step 4b), it was already inserted with
`status='in_progress'`; this UPDATE is what closes it out.

## 2. `blog_posts` — the review/schedule record

Always `status = 'draft'` — never `'scheduled'` or `'published'`; that is Mila's call.

```sql
INSERT INTO public.blog_posts (
  client_id, title, slug, meta_description, content, excerpt,
  focus_keyword, topic_category, target_keywords,
  featured_image_url, featured_image_alt, status,
  duda_page_id, duda_publish_url, ai_model,
  topic_id, auto_run_id, quality_scores
) VALUES (
  '<client_id>', '<title>', '<slug>', '<meta_description>', '<content_html>', '<excerpt>',
  '<focus_keyword>', '<category>', ARRAY['<kw1>', '<kw2>'],
  '<featured_image_url>', '<featured_image_alt>', 'draft',
  '<duda_post_id>', '<duda_preview_or_editor_url>', 'claude',
  '<topic_id_or_null>', '<auto_run_id>', '<quality_scores_jsonb>'
)
RETURNING id;
```

`quality_scores` is a JSONB blob of the doctrine QA verdict. Observed real shape (match this):

```json
{
  "verdict": "SHIP",
  "word_count": 1654,
  "h2_count": 9,
  "h1_in_body": 0,
  "internal_links": 12,
  "external_sources": 3,
  "sources": ["https://...", "https://..."],
  "sourcing_mode": "tavily_primary" ,
  "doctrine_checks": {
    "title": "PASS", "description": "PASS", "headings": "PASS",
    "content": "PASS", "interlinking": "PASS",
    "schema": "PASS - Article auto-applied by Duda; FAQ section present, FAQ JSON-LD needs manual add via Duda Settings > Custom HTML",
    "intent": "PASS", "escalation": "none"
  }
}
```
Use `"sourcing_mode": "degraded_dataforseo_only_tavily_down"` when Tavily was unavailable and you
fell back to DataForSEO-only sourcing per Step 0a.

## 3. `blog_auto_runs` — one row per client per tick

Open at the **start** of a client's processing:

```sql
INSERT INTO public.blog_auto_runs (client_id, trigger_type, status)
VALUES ('<client_id>', 'scheduled', 'running')
RETURNING id;
```
Use `trigger_type = 'manual'` only for an ad hoc single-`client_id` invocation.

Close the same row at the **end** of that client's processing (always update, never insert twice):

```sql
UPDATE public.blog_auto_runs
SET status = '<completed|failed>',
    completed_at = now(),
    topics_generated = <0_or_1>,
    articles_generated = <0_or_1>,
    articles_passed_gate = <0_or_1>,
    articles_flagged = <0_or_1>,
    error_log = '<null_if_completed_with_a_draft__otherwise_the_reason>'
WHERE id = '<run_id>';
```
`status='completed'` covers both "drafted a post" and "correctly determined there was no eligible
work this tick" (e.g. the one-draft-per-day guard already tripped, or no calendar topic was usable).
Reserve `status='failed'` for an actual break in the pipeline (doctrine files unreadable, a
connector down mid-run, an unexpected write-back rejection). Put a real, specific explanation in
`error_log` either way — it is the audit trail the next tick (and Garrett) reads.

## 4. `tasks` + `task_assignees` — only when a new draft was created

Look up Mila and the client's "To Do" column:

```sql
SELECT id FROM auth.users WHERE email = 'mila@spearlance.com';
SELECT id FROM public.task_columns WHERE client_id = '<client_id>' AND key = 'to_do';
```

```sql
INSERT INTO public.tasks (client_id, title, description, status, column_id)
VALUES (
  '<client_id>',
  'Review Duda draft: <short_post_title>',
  'Auto Blogger drafted this post. Duda preview: <preview_or_editor_url>. Keyword: <focus_keyword>. QA verdict: SHIP.',
  'to_do',
  '<to_do_column_id>'
)
RETURNING id;

INSERT INTO public.task_assignees (task_id, user_id)
VALUES ('<task_id>', '<mila_user_id>');
```

Skip this step entirely when no new `blog_posts` row was created this tick (including when the
one-draft-per-day guard blocked it) — don't file a review task for work that didn't happen.
