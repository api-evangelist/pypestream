---
name: pypestream-embed-and-run-a-chat
description: >-
  Start a Pypestream engagement for an end user and exchange messages with the microapp over the
  Engagement API and its Phoenix Channels WebSocket, respecting the chat:ready precondition.
api: pypestream:engagement-api
generated: '2026-08-26'
method: generated
source: >-
  openapi/pypestream-engagement-api-openapi.json,
  https://developers.pypestream.com/reference/engagement-overview,
  https://developers.pypestream.com/reference/engagement-api-websocket
base_url: https://engagement-api.pypestream.com/
sandbox_url: https://engagement-api-sandbox.pypestream.com/
operations:
  - anonymous_session
  - start
  - message
  - ping
  - type
  - snapshot
  - end
websocket_events:
  - chat:start
  - chat:ready
  - new:ping
  - new:pong
  - msg:send
  - incoming:msg
  - incoming:notice
  - chat:end
  - chat:snapshot
  - chat:snapshot_response
---

# Embed and run a Pypestream chat

This flow is **half REST, half WebSocket**. Calling only the REST operations will not work: the
messages the microapp sends back arrive on the socket, and one REST call is gated on a socket event.

## Before you start

You need `app_id`, `pype_id` and `stream_id` from your Pypestream environment, and a `consumer`
identifier for the end user. Pypestream does not publish a self-serve way to obtain these; they come
from your Pypestream environment.

## Steps

1. **Create the user.** `POST /messaging/v1/consumers/anonymous_session` (`anonymous_session`).
   Keep the `access_token` from the response — it authenticates both the remaining REST calls
   (HTTP Bearer) and the WebSocket (as the `token` connection parameter).

2. **Open the socket before you start the engagement.**
   Connect to `wss://engagement-api.pypestream.com/socket/websocket` with
   `params: {token: <access_token>}`, using a Phoenix Channels client.
   Join the topic `chat:{CHAT_ID}` *before* step 3, or you will miss the microapp's opening
   messages.

3. **Start the engagement.** `POST /messaging/v1/chats/{chat_id}/start` (`start`), and emit the
   `chat:start` event with `app_id`, `consumer`, `gateway: pypestream_widget`, `pype_id`,
   `stream_id`, `user_id`, `version: "1"` and `access_token`.
   A `409` here means an engagement is already active for this consumer (`ExistingActiveConversation`)
   — end it or resume it rather than retrying.

4. **Wait for `chat:ready`.** Do not skip this. Sending a message before `chat:ready` arrives is
   rejected with **HTTP 428** (`PreconditionRequired`). This is the most common first-integration
   failure on this API and it is documented on the WebSocket page, not on the REST operation.

5. **Send messages.** `POST /messaging/v1/chats/{chat_id}/message` (`message`), or emit `msg:send`
   on the socket. Replies arrive as `incoming:msg`; out-of-band notices arrive as
   `incoming:notice`.

6. **Keep the connection alive.** Emit `new:ping` every **20 seconds** with `seq`, `user_id` and
   `access_token`. Pypestream replies `new:pong` with the matching `seq`. If you stop, the session
   times out and the socket closes with WebSocket code **1000**.

7. **Optionally snapshot.** `POST /messaging/v1/chats/{chat_id}/snapshot` (`snapshot`) or emit
   `chat:snapshot`; the state comes back as `chat:snapshot_response`. Use this to rehydrate after a
   reconnect rather than replaying the conversation.

8. **End the engagement.** `POST /messaging/v1/chats/{chat_id}/end` (`end`), or emit `chat:end`.

## Rules an agent must not get wrong

- **A sent message cannot be recalled.** There is no unsend, edit or delete operation on any
  Pypestream contract. Treat `message` as irreversible and confirm content before sending.
- **`end` is not an undo.** It terminates the engagement; the conversation and its transcript
  persist and stay readable through the Reporting API for 30 days.
- **There is no idempotency key.** If `start` or `message` times out, retrying may duplicate. Use
  `snapshot` to check state before retrying rather than blindly re-sending.
- **Errors are `{message, errors[{source, type, message}]}`.** The permitted values of `type` are
  not published, so branch on HTTP status, not on `type`.
- **WebSocket 1007** means your frame was malformed; **HTTP 403** on connect means the token is
  invalid — get a new one from `anonymous_session`.
