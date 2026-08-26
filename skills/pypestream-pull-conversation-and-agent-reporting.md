---
name: pypestream-pull-conversation-and-agent-reporting
description: >-
  Extract Pypestream conversation metadata, transcripts and agent activity into a data warehouse
  using the Reporting API, honouring its window limits and 30-day retention.
api: pypestream:reporting-api
generated: '2026-08-26'
method: generated
source: >-
  openapi/pypestream-reporting-api-openapi.json,
  https://developers.pypestream.com/reference/reporting-api-overview,
  https://developers.pypestream.com/reference/authentication
base_url: https://reporting.pypestream.com/api/v2/
sandbox_url: https://reporting-sandbox.pypestream.com/api/v2/
operations:
  - GET /single-agent/{agent_id}
  - GET /multi-agent
  - GET /single-chat/{chat_id}
  - GET /multi-chat
  - GET /single-chat-transcript/{chat_id}
  - GET /multi-chat-transcript
---

# Pull Pypestream reporting data

> **This contract declares no `operationId` on any operation.** Operations are addressed by method
> and path throughout. A generated client will not have named methods.

## Access

The Reporting API is **not on by default**: Pypestream states it is "available upon request on a
Pype per Pype basis". Authentication is HTTP Basic, where the username is a **Client ID** and the
password a **Client Secret**, created by a Conversation Manager in the Pypestream platform under
Conversations → Admin → Pypestream API → Create API Key. HTTPS is required.

## What you can pull

| Purpose | Call |
|---|---|
| One agent | `GET /single-agent/{agent_id}` |
| Agents in a window | `GET /multi-agent?from=&to=` |
| One session's metadata | `GET /single-chat/{chat_id}` |
| Sessions in a window | `GET /multi-chat?from=&to=` |
| One transcript | `GET /single-chat-transcript/{chat_id}` |
| Transcripts in a window | `GET /multi-chat-transcript?from=&to=` |

`from` and `to` are **RFC3339** (`YYYY-MM-DDTHH:MM:SSZ`), e.g. `2021-12-20T00:00:00Z`.

## The limits that will bite you

- **Window maxima differ per endpoint.** `multi-agent` accepts a **24-hour** frame.
  `multi-chat` and `multi-chat-transcript` accept **no more than one hour**. Exceeding the maximum
  returns **HTTP 403** — not 400 — so do not treat 403 as an auth failure here.
- **Retention is 30 days.** A request for data older than 30 days returns an **empty array with a
  200**, not an error. An extractor that only checks status codes will silently record zero rows.
  Check the array length against the window you asked for.
- **Rate limit: 400 requests/hour and 5 concurrent requests.** Exhaustion is documented as requests
  being "ignored"; **no 429 is declared and no `RateLimit-*` or `Retry-After` header is returned**,
  so you must count your own calls. At a one-hour window and 400 requests/hour you can backfill
  roughly 16 days of transcripts per hour of wall clock — budget accordingly.
- **No pagination.** There is no cursor and no page size; the time window *is* the pagination.

## Rules an agent must not get wrong

- **Do not point a dashboard at this API.** Pypestream states directly: collect and process into
  your own data warehouse, then serve your dashboards from there.
- **Error responses have no body.** All 18 declared 4xx responses across these six operations carry
  a description and **no schema and no media type**. Branch on status code only.
- **This surface is read-only** — reversibility, idempotency and dry-run are all `na`.
- **Transcripts contain end-user conversation content.** Pypestream's own guidance is to issue a
  separate key per developer so requests are attributable, and never to share keys.
