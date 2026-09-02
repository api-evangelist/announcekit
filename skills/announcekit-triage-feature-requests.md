---
name: announcekit-triage-feature-requests
description: Work an AnnounceKit feature request board — read what users asked for, reply to them, and promote accepted requests onto the public roadmap.
api: AnnounceKit
generated: '2026-09-02'
method: generated
source: >-
  Grounded in the real MCP tool names published in announcekit-mcp 0.1.0 and the
  GraphQL root fields they call, verified against the live introspected schema
  at https://announcekit.app/gq/v2 (2026-09-02).
surfaces:
  mcp: https://mcp.announcekit.app/mcp
  graphql: https://announcekit.app/gq/v2
operations:
  mcp:
    - list_projects
    - list_feature_requests
    - list_feedback
    - list_roadmap
    - create_feature_request
    - comment_feature_request
    - reply_feature_request
    - create_roadmap_item
    - create_roadmap_status
    - save_label
  graphql:
    - featureRequests
    - feedbacks
    - feedbackCounts
    - statuses
    - saveFeatureRequest
    - commentFeatureRequest
    - replyFeatureRequest
    - saveIssue
    - saveStatus
    - saveLabel
scope_required: write
---

# Triage feature requests

## 1. Read the board

`list_feature_requests` (GraphQL `featureRequests`) takes `project_id`,
`sort_by` and `page`. `sort_by` is the `FeatureRequestSortBy` enum — exactly
three values, `TOP`, `TRENDING`, `NEW`. Anything else is a validation error.

Each row carries vote and comment counts via `FeatureRequestStats`. Pagination is
1-based page numbers; the response gives you `page`, `pages` and `count`. There
is no page-size argument — page size is fixed server-side and undocumented.

`list_feedback` (GraphQL `feedbacks` + `feedbackCounts`) is the adjacent signal:
reactions and comments left on published posts rather than standalone requests.

## 2. Understand the roadmap shape before you write to it

`list_roadmap` (GraphQL `statuses`) returns the roadmap as **status columns**,
not as a flat list of items. `Issue` records hang off a `Status`. So creating a
roadmap item requires a `status_id`, which means reading the roadmap first.

If the column you need does not exist, `create_roadmap_status` (GraphQL
`saveStatus`) creates one with a name and a colour.

## 3. Respond

Two different actions, and the difference matters:

- `comment_feature_request` (GraphQL `commentFeatureRequest`) — adds a comment,
  and optionally sets the request's status. Quiet.
- `reply_feature_request` (GraphQL `replyFeatureRequest`) — **notifies the
  request's subscribers.** This sends mail to real people.

Use `comment_feature_request` for internal notes and status changes. Only use
`reply_feature_request` when the human has asked you to tell the requesters
something, and confirm the wording first. `deleteFeatureRequestComment` exists
in the GraphQL API but is not an MCP tool, and deleting a reply does not unsend
the notifications it triggered.

## 4. Promote to the roadmap

`create_roadmap_item` (GraphQL `saveIssue`) takes `project_id`, `status_id`,
`title` and optionally `summary`, `labels` and `due_at`.

The link back to the request is `FeatureRequest.issue_id`. That field is the
hinge of the whole product — it is what turns "users voted for this" into "this
is on the public roadmap". Set it via `create_feature_request` with both
`feature_request_id` and `issue_id` (the tool upserts when an id is supplied).

## 5. Close the loop

When the item ships, the release note that announces it can reference the
request through the post's `related_feature_requests`. See
`announcekit-publish-release-note`.

## Conventions that will bite you

- **Upsert, not create.** `create_feature_request`, `create_roadmap_item`,
  `create_roadmap_status` and `save_label` all take an optional id and update
  when it is present. Passing the id makes a retry safe; omitting it on a retry
  creates a duplicate. There is no idempotency key.
- **Archive is the soft path.** `saveFeatureRequest` with `is_archived: true`
  is reversible. `deleteFeatureRequest` is a hard delete with no documented
  retention window — and it is not exposed as an MCP tool at all.
- **Segment filters are opaque.** `segment_filters` on both `FeatureRequest` and
  `Post` is a `JSONObject` with no published shape. Do not construct one from
  guesswork; read an existing record if you need the format.
- **Errors are HTTP 200.** Check the `errors` array, not the status code.
