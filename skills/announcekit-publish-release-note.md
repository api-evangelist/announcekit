---
name: announcekit-publish-release-note
description: Draft, review and publish (or schedule) a release note in AnnounceKit, including localized copy and labels.
api: AnnounceKit
generated: '2026-09-02'
method: generated
source: >-
  Grounded in the real MCP tool names published in announcekit-mcp 0.1.0 and the
  GraphQL root fields they call, verified against the live introspected schema
  at https://announcekit.app/gq/v2 (2026-09-02). No operation named here was
  invented.
surfaces:
  mcp: https://mcp.announcekit.app/mcp
  graphql: https://announcekit.app/gq/v2
operations:
  mcp:
    - list_projects
    - list_labels
    - list_post_templates
    - generate_post_draft
    - improve_text
    - create_post
    - update_post
    - update_post_locale
    - publish_post
    - schedule_post
    - get_post
  graphql:
    - me
    - labels
    - postTemplates
    - autoGeneratePostContents
    - aiActions
    - savePost
    - updatePostLocale
    - post
scope_required: write
---

# Publish a release note

The whole flow runs against one GraphQL mutation, `savePost`. The MCP server
splits it into `create_post`, `update_post`, `publish_post` and `schedule_post`
because post state is not an enum — it is computed from `is_draft` plus the
`visible_at` / `expire_at` dates. Keep that in mind: these are four framings of
the same write.

## 1. Resolve the project

`list_projects` (GraphQL `me`) returns the projects the token can reach. Every
subsequent call needs `project_id`. A token is limited to the projects its
creating member can access, at that member's role — if a call comes back
`Access Denied.`, the token, not the query, is usually wrong.

## 2. Resolve the labels

`list_labels` (GraphQL `labels(project_id:)`) returns `{ id, name }`. Pass label
IDs, never names, into `create_post`. One `Label` serves posts, feature requests
and the roadmap; which surfaces it applies to lives in its options.

## 3. Draft

Two options:

- `list_post_templates` and start from a reusable template, or
- `generate_post_draft` (GraphQL `autoGeneratePostContents`) to have AnnounceKit's
  built-in AI write the body from a short prompt and a tone.

`improve_text` (GraphQL `aiActions`) rewrites a snippet in place — grammar, tone,
length — and is useful between drafting and publishing.

## 4. Create as a draft

`create_post` takes `project_id`, `title`, `body`, and optionally `summary`,
`labels`, `locale`, `visible_at`, `is_draft`. **It defaults to a draft.** Leave
it that way. Body accepts basic HTML.

This is the closest thing AnnounceKit has to a dry run: the draft is a real
persisted record, but nothing has been delivered to readers yet. Use `get_post`
to read it back before going further, and `previewPost` on the GraphQL side to
render it.

## 5. Localize (optional)

`update_post_locale` sets the title and body for one locale. A post carries one
`PostContent` per locale with a `defaultContent` shortcut, so add locales one
call at a time rather than trying to send them all through `create_post`.

## 6. Publish or schedule

- `publish_post` — live now.
- `schedule_post` — live at a future `visible_at`.

Prefer `schedule_post` when you can. It is the ONLY step in this flow with a
real reversal window: a scheduled post can still be edited or pulled back to
draft right up until `visible_at` passes.

## Reversal — read this before you publish

Publishing is a delivery action. AnnounceKit fans a published post out to
in-app widgets, email, Slack and RSS.

- Unpublishing is possible in the API (`savePost` with `is_draft: true`; the
  `post.unpublish` webhook event confirms it is a first-class transition), but
  **there is no `unpublish_post` MCP tool**. An agent can publish and cannot
  un-publish without a human in the dashboard or a direct GraphQL call.
- There is no `delete_post` tool either. The MCP server ships no deletes at all,
  by design.
- **Nothing in AnnounceKit's documentation states that unpublishing recalls or
  suppresses already-delivered emails, Slack messages or RSS entries.** Do not
  tell a user it does. Assume delivery is one-way.

So: confirm with the human before calling `publish_post`. Schedule if there is
any doubt.

## Conventions that will bite you

- **No idempotency.** There is no idempotency key. `create_post` with no
  `post_id` creates a NEW post every time. If a create times out, do not blind
  retry — call `list_posts` and check first, or you will publish twice.
- **`update_post` is a read-modify-write**, not a patch. The server reads the
  post and re-sends merged content. Concurrent edits can clobber each other.
- **Errors come back HTTP 200.** A GraphQL `errors` array with `"Access Denied."`
  arrives with a 200 status. Branch on the response body, never on the status
  code. The one exception is rate limiting, which really is a 429.
- **Rate limit is 60 requests/minute per IP**, shared by everything behind that
  address. Watch `X-RateLimit-Remaining`. There is no reset header and no
  `Retry-After`, so back off exponentially rather than computing a wake time.
