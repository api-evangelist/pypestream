---
name: pypestream-escalate-to-a-live-agent
description: >-
  Check contact-center agent availability, escalate a Pypestream conversation to a human agent,
  pass context, and exchange messages and typing indicators over the Contact Center API.
api: pypestream:contact-center-api
generated: '2026-08-26'
method: generated
source: >-
  openapi/pypestream-contact-center-api-openapi.json,
  https://developers.pypestream.com/reference/contact-center-api-overview,
  https://developers.pypestream.com/reference/contact-center-webhooks
base_url: https://middleware.pypestream.com/
sandbox_url: https://middleware-sandbox.pypestream.com/
operations:
  - listIntegrations
  - createIntegration
  - getIntegration
  - updateIntegration
  - patchIntegration
  - deleteIntegration
  - getSettings
  - setSettings
  - conversationHistory
  - getMetadata
  - updateMetadata
  - sendMessage
  - sendTypingIndicator
  - sendEnd
webhook_events:
  - agents/availability
  - agents/waitTime
  - conversations/{conversationId}/escalate
  - conversations/{conversationId}/type
  - conversations/{conversationId}/messages/{messageId}
  - conversations/{conversationId}/end
---

# Escalate a Pypestream conversation to a live agent

This API is **webhook-driven**. Pypestream pushes the events; your integration reacts. Polling is
not the intended shape.

## Set up once

1. **Register the integration.** `POST /contactCenter/v2/integrations` (`createIntegration`).
   `listIntegrations`, `getIntegration`, `updateIntegration` (PUT, full replace) and
   `patchIntegration` (PATCH, partial) manage it afterwards.
   `deleteIntegration` removes it — this is the reversal, and it is immediate and permanent.
2. **Read and set contact-center configuration** with `getSettings` / `setSettings` on
   `/contactCenter/v1/settings`.
3. **Stand up a webhook endpoint** on a publicly reachable HTTPS URL and return `2xx` per event.
   Pypestream documents **no signing secret and no signature header**, so you cannot verify a
   delivery came from Pypestream from the request alone — restrict by network path and treat
   payload identifiers as untrusted until re-read through the API.

## The escalation flow

1. **`agents/availability` arrives** when an end user asks for an agent. It carries `available`,
   `estimatedWaitTime` (seconds), `hoursOfOperation`, `queueDepth` and `status`
   (`online` / `busy` / `offline`). Decide here whether to escalate or offer an alternative.
2. **`agents/waitTime`** arrives when agents are busy; surface `estimatedWaitTime` to the user.
3. **`conversations/{conversationId}/escalate`** arrives when the end user is escalated. Route to a
   skilled agent group.
4. **Give the agent context.** `GET /contactCenter/v2/conversations/{conversationId}/history`
   (`conversationHistory`) returns the automated conversation so far; `getMetadata` on
   `/contactCenter/v1/conversations/{conversationId}/metadata` returns end-user metadata and
   `updateMetadata` (PATCH) updates it from the agent side.
5. **Exchange messages.**
   `PUT /contactCenter/v1/conversations/{conversationId}/messages/{messageId}` (`sendMessage`) —
   note this is a **PUT with a caller-supplied `messageId`**, which is the closest thing this API
   has to idempotency: reusing the same `messageId` addresses the same message.
   `POST .../type` (`sendTypingIndicator`) sets the typing status.
6. **Close out.** `POST /contactCenter/v1/conversations/{conversationId}/end` (`sendEnd`), or handle
   the inbound `conversations/{conversationId}/end`.

## Rules an agent must not get wrong

- **Version paths are mixed in one contract.** Integrations and history are on
  `/contactCenter/v2/`; conversations, metadata, typing, end and settings are on
  `/contactCenter/v1/`. Do not assume one prefix.
- **The published servers say `http://`.** Use `https://middleware.pypestream.com/` — the host
  redirects, and Pypestream's own docs require HTTPS. Never send Basic credentials over the
  plaintext URL as published.
- **Auth is HTTP Basic (Client ID / Client Secret) or the `X-Pypestream-Token` header.** Keys are
  created by a Conversation Manager in the platform. Pypestream asks that each developer hold a
  separate key and that keys are never shared.
- **The event list is explicitly incomplete.** Pypestream states it may add event types at any time,
  so ignore unknown events rather than failing on them.
- **No retry or ordering guarantee is published** for webhook delivery. Make your handler
  idempotent on `messageId` and `conversationId`.
- **A message sent to a customer cannot be recalled.**
