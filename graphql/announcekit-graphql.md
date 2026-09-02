---
generated: '2026-09-02'
method: probed
source: https://announcekit.app/gq/v2
provider: AnnounceKit
providerId: announcekit
schema_file: graphql/announcekit-schema.graphql
introspected: '2026-09-02'
introspection_open: true
---

# AnnounceKit GraphQL API

AnnounceKit's machine-readable contract is a GraphQL schema. There is no
OpenAPI — `/openapi.json`, `/openapi.yaml`, `/swagger.json` and `/api-docs` all
return 404 on `announcekit.app` (probed 2026-09-02).

**Endpoint:** `https://announcekit.app/gq/v2`

**Documentation:** https://announcekit.app/docs/graphql-api

**Schema:** [`graphql/announcekit-schema.graphql`](announcekit-schema.graphql) —
1,885 lines of SDL, rendered from a full introspection of the live endpoint on
2026-09-02.

## Introspection is open

A `POST` of the standard introspection query to `https://announcekit.app/gq/v2`
returns the complete schema with no credential. That is unusual and it is the
reason this repo now holds a real contract rather than a prose stub: the schema
is self-describing and anyone can read it.

Executing most queries is a different matter — `{ posts(project_id: 1) { ... } }`
returns `"Access Denied."` unauthenticated. Reading the shape is public;
reading the data is not.

## Surface

| | Count |
|---|---|
| Types (excluding introspection) | 171 |
| Object types | 142 |
| Input object types | 16 |
| Enums | 6 |
| Custom scalars | 2 (`Date`, `JSONObject`) |
| `Query` fields | 93 |
| `Mutation` fields | 150 |

`Subscription` is declared in the schema block but resolves to the subscription
billing object, not to live-event subscriptions — AnnounceKit's event surface is
webhooks, not GraphQL subscriptions.

## Authentication

A Basic Authentication token in the request header, or the dashboard login
cookies (`sesid`, `sesid.sig`). Signed into the dashboard, the GraphiQL IDE
works with neither. For agents, an `ak_pat_` bearer token or the OAuth flow
against the MCP server. See
[`authentication/announcekit-authentication.yml`](../authentication/announcekit-authentication.yml).

## Rate limiting

60 requests per minute per **IP address** — not per key, not per account, so it
is shared by every integration behind the same egress address.
`X-RateLimit-Limit` and `X-RateLimit-Remaining` are returned on every response
(observed live). A breach returns HTTP 429 with `extensions.code: RATE_LIMITED`.

## Errors

Everything except the rate limit returns **HTTP 200** with a GraphQL `errors`
array. Only `RATE_LIMITED` carries a machine-readable `extensions.code`; the
rest are bare message strings. See
[`errors/announcekit-problem-types.yml`](../errors/announcekit-problem-types.yml).

## Related artifacts

- [`mcp/announcekit-mcp.yml`](../mcp/announcekit-mcp.yml) — the 29-tool MCP server that wraps this schema
- [`mcp/announcekit-tool-crosswalk.yml`](../mcp/announcekit-tool-crosswalk.yml) — every tool bound to the GraphQL field it calls
- [`data-model/announcekit-data-model.yml`](../data-model/announcekit-data-model.yml) — the entity graph derived from this schema
- [`conventions/announcekit-conventions.yml`](../conventions/announcekit-conventions.yml) — pagination, idempotency, reversibility
- [`asyncapi/announcekit-webhooks.yml`](../asyncapi/announcekit-webhooks.yml) — the 11 webhook events
