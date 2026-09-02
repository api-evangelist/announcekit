---
name: announcekit-report-post-performance
description: Pull reach, engagement and NPS numbers for AnnounceKit announcements over a date range.
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
    - list_posts
    - get_post
    - get_post_stats
    - get_post_status_summary
    - list_activities
    - list_feedback
    - get_nps
    - list_external_users
    - list_segments
    - list_feeds
  graphql:
    - me
    - posts
    - post
    - getPostStatistics
    - getPostStatus
    - activities
    - feedbacks
    - feedbackCounts
    - getNps
    - externalUsers
    - externalUserCount
    - segments
    - feeds
scope_required: read
---

# Report on announcement performance

This is the read-only flow. A `read`-scoped `ak_pat_` token is enough — all
eleven tools below are in the 15-tool read set. Prefer a read token for
reporting work; it makes it structurally impossible to publish by accident.

## 1. Frame the window

`get_post_status_summary` (GraphQL `getPostStatus`) takes `project_id` and a
`DateRange` (`start_date`, `end_date`) and answers the shape question first: how
many posts created in this window are live, scheduled, or still draft. Start
here — it tells you whether a quiet month is a reach problem or a publishing
problem.

## 2. Pull the posts

`list_posts` (GraphQL `posts`) takes `project_id`, `page` and a free-text
`query`. The response wrapper carries `list`, `count`, `page` and `pages`.

Note the naming inconsistency you will hit immediately: the item array is called
`list` on `Posts`, `FeatureRequests`, `Issues` and `RoadmapItems`, but `items`
on the `PageOf*` types (`PageOfActivities`, `PageOfFeedback`,
`PageOfExternalUsers`). Same envelope, two field names. Handle both.

## 3. Per-post numbers

`get_post_stats` (GraphQL `getPostStatistics`, via an `AnalyticsInput`) takes
`project_id`, `post_id`, `start_date` and `end_date` and returns views, link
clicks, reactions and feedback.

`get_post` fills in the context those numbers need: status, `visible_at`,
`expire_at`, labels and the localized contents.

## 4. Engagement detail

- `list_feedback` — reaction counts plus the actual comments, per post or across
  the project.
- `list_activities` (GraphQL `activities`) — the raw event stream: views,
  clicks, feedback, votes. The `ActionSource` enum tells you WHERE each event
  came from: `widget`, `email`, `feed`, `nps`. That is the channel breakdown,
  and it is the most useful cut in the whole dataset for a multi-channel
  product.
- `get_nps` (GraphQL `getNps`) — NPS score and response counts for a post's
  survey.

## 5. Audience

- `list_external_users` — the audience, paged, with a total from
  `externalUserCount`. These are the customer's end users, NOT AnnounceKit
  account holders.
- `list_segments` — the segment fields and saved profiles you can slice by.
- `list_feeds` — the changelog pages posts are published to.

## Conventions that will bite you

- **Rate limit is 60 requests per minute per IP.** A per-post stats loop over a
  few hundred posts will hit it. Batch by date range where the tool allows it
  (`get_post_stats` takes a range), watch `X-RateLimit-Remaining`, and back off
  exponentially — there is no reset header and no `Retry-After` to read.
- **Pagination is offset-based**, 1-based `page`, no page-size control. Read
  `pages` from the first response and loop, rather than guessing a limit.
- **Unauthenticated reads can silently return null** rather than erroring —
  `{ me { id } }` returns `{"data":{"me":null}}` with HTTP 200 when no
  credential is supplied. Null-check before assuming an empty result means
  "nothing there".
- **Analytics filter arguments are `JSONObject`** (`user_filter`,
  `segmentProfile`, `filters`) with no published shape. Their format is not in
  the schema and is not documented.
