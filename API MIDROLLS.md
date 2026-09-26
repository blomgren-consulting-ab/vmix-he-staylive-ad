# Livestream Messages — Mid-roll Ad Triggers

Programmatic mid-roll ad insertion for an active Staylive livestream. The
endpoints documented here are used by the Staylive Dashboard today and are
exposed to production companies that want to drive ad breaks from their own
broadcast tooling.

A `POST` triggers an ad on every viewer currently watching the livestream.
The trigger reaches each viewer's player in real time, so latency from API
call to ad playback is typically well under one second.

Two things are worth knowing before you integrate:

* **Real-time delivery is at-most-once.** A broadcast goes to every viewer
watching at the instant of the `POST`. Viewers whose connection has
briefly dropped at that moment will not receive the trigger and are not
caught up afterwards. The `failed` counter in the response tells you how
many viewers were missed.
* **Broadcast history is queryable.** Broadcasts are stored server-side and
the most recent 1,000 are returned by `GET /livestreams/{id}/messages`.
Cancelling a broadcast with `DELETE` also removes it from this history.
Use the history to see what has been triggered on a stream, or to
reconcile state if your own client process restarts. For delivery
reporting, capture `sentTo` and `failed` from each `POST` response — the
stored history only records how many viewers were connected at broadcast
time (`viewer\\\\\\\_count`), not how many the trigger reached.

\---

## Base URL

```
https://api.staylive.tv
```

All paths below are relative to this base URL.

## Authentication

`POST` and `DELETE` requests must include a Staylive JWT in the
`Authorization` header:

```
Authorization: Bearer <token>
```

Tokens are generated from the **API keys** section of the Staylive
Dashboard. Create the API key with one of the following access levels:

**Owner**, **Administrator**, **Producer**, or **Regular user**.

The key can be created on the channel owning the livestream, or on a
collection or platform containing that channel — access carries down the
hierarchy, so a collection- or platform-level key is valid for every
channel under it.

Calls without a valid token are rejected with `401 Unauthorized`. Calls
made with a valid token by a user who does not hold one of these roles are
rejected with `403 Forbidden`. A `403` is also what you get for a
livestream ID that does not exist — no role can be resolved against an
unknown livestream — so double-check the ID before suspecting the token.

`GET /livestreams/{id}/messages` requires no authentication.

## Response envelope

All success responses are wrapped in the standard Staylive API envelope:

```json
{
  "message": { /\\\\\\\* endpoint-specific payload \\\\\\\*/ },
  "statuscode": 201,
  "success": true,
  "data": null
}
```

Errors follow the standard [Boom](https://hapi.dev/module/boom/) shape, e.g.:

```json
{
  "statusCode": 401,
  "error": "Unauthorized",
  "message": "Missing authentication"
}
```

One quirk to be aware of: field names inside `message` use camelCase in the
`POST` response (`messageId`, `viewerCount`) but snake\_case in the history
returned by `GET` (`time\\\\\\\_sent`, `viewer\\\\\\\_count`). This is expected — match
each endpoint's documented field names exactly rather than assuming one
convention.

\---

## POST /livestreams/{id}/messages

Broadcasts a message to every viewer currently watching the livestream. The
only supported action today is `PLAY\\\\\\\_AD`, which instructs the player to start
a mid-roll ad break.

### Path parameters

|Name|Type|Description|
|-|-|-|
|`id`|integer|Numeric Staylive livestream ID.|

### Request body

`Content-Type: application/json`

|Field|Type|Required|Description|
|-|-|-|-|
|`action`|string|yes|Action to broadcast. Use `"PLAY\\\\\\\_AD"` to trigger a mid-roll ad.|
|`ad\\\\\\\_url`|string|yes (when `action = "PLAY\\\\\\\_AD"`)|A VAST 3.0/4.x ad tag URL. The viewer's player will fetch this tag and play the returned ad.|
|`playback\\\\\\\_timestamp`|integer|yes (when `action = "PLAY\\\\\\\_AD"`)|The wall-clock moment, in Unix epoch milliseconds, that the ad should appear from the producer's perspective. See [Choosing the playback timestamp](#choosing-the-playback-timestamp) below.|

Additional fields may be included in the payload and will be forwarded to
viewers untouched, but they are not currently used by the Staylive player.

### Example request

```bash
curl -X POST "https://api.staylive.tv/livestreams/8451/messages" \\\\\\\\
  -H "Authorization: Bearer $STAYLIVE\\\\\\\_TOKEN" \\\\\\\\
  -H "Content-Type: application/json" \\\\\\\\
  -d '{
    "action": "PLAY\\\\\\\_AD",
    "ad\\\\\\\_url": "https://ads.example.com/vast?tag=midroll-promo-1",
    "playback\\\\\\\_timestamp": 1748177500000
  }'
```

### Example response — `201 Created`

```json
{
  "message": {
    "success": true,
    "messageId": "7f9c1b2e-4d3a-4f6b-9e8d-2a1c5b7d9e0f",
    "viewerCount": 1843,
    "sentTo": 1843,
    "failed": 0
  },
  "statuscode": 201,
  "success": true,
  "data": null
}
```

|Field|Description|
|-|-|
|`messageId`|Identifier for the broadcast (UUID). Pass to `DELETE` to stop the ad early.|
|`viewerCount`|Number of WebSocket viewer sessions connected at the moment of broadcast.|
|`sentTo`|Number of viewers the trigger was successfully delivered to.|
|`failed`|Number of viewers the trigger failed to reach (e.g. socket dropped between accept and send).|

Any additional fields you include in the request body are forwarded to
viewers as-is, so you can pass campaign metadata, tracking IDs, etc. if
your player needs them.

### Error responses

|Status|When|
|-|-|
|`400 Bad Request`|Payload fails validation (missing `action`, `ad\\\\\\\_url` or `playback\\\\\\\_timestamp` for `PLAY\\\\\\\_AD`, or non-integer/non-positive `playback\\\\\\\_timestamp`).|
|`401 Unauthorized`|Missing, expired, or invalid token.|
|`403 Forbidden`|Token is valid but the user does not hold an accepted role on the livestream — also returned when the livestream ID does not exist.|
|`502 Bad Gateway`|Staylive's messaging service is unreachable or the broadcast failed. This usually means no viewer received the trigger, but it is not a guarantee — see [Retrying safely](#operational-notes) before re-posting.|

### Choosing the playback timestamp

`playback\\\\\\\_timestamp` is the moment, on the producer's clock, when the ad
should appear to play. Because every viewer experiences a different delay
between the source feed and their own playback head, the Staylive player
uses this timestamp together with the live stream's Program-Date-Time (PDT)
to align the ad on each viewer's local timeline.

Two common patterns:

* **On-site production (producer is at the venue):** the producer sees the
action with negligible delay, so pass `Date.now()` from the system
triggering the ad. The ad will appear on each viewer with their normal
stream delay relative to live.
* **Off-site production (producer is watching the encoded stream):** the
producer is themselves behind by the stream's end-to-end delay. Pass the
PDT of the frame the producer is currently watching (in epoch
milliseconds) so that all viewers see the ad break aligned to the same
moment of broadcast content the producer was looking at when they
triggered it.

If `playback\\\\\\\_timestamp` is far in the past (more than a few seconds), most
viewers will play the ad immediately on receipt rather than catching up.

\---

## GET /livestreams/{id}/messages

Returns up to the most recent 1,000 messages broadcast on this livestream,
oldest first. Includes messages whose ads have already finished playing.

Useful for:

* Showing operators which ad breaks have been triggered on the stream.
* Reconciling your own state if your client process restarts.

Two limitations to be aware of:

* `viewer\\\\\\\_count` is the number of viewers connected at broadcast time, not
the number the trigger was delivered to. Delivery counts (`sentTo`,
`failed`) are only returned in the `POST` response and are not stored, so
capture them there if you need delivery reporting.
* There is no status field, and the API does not know when an ad finishes —
ad length is determined by the VAST creative in each viewer's player. The
history tells you what was triggered and when (`time\\\\\\\_sent`), not whether
an ad is still playing.

### Path parameters

|Name|Type|Description|
|-|-|-|
|`id`|integer|Numeric Staylive livestream ID.|

### Example response — `200 OK`

```json
{
  "message": {
    "count": 1,
    "messages": \\\\\\\[
      {
        "id": "7f9c1b2e-4d3a-4f6b-9e8d-2a1c5b7d9e0f",
        "action": "PLAY\\\\\\\_AD",
        "ad\\\\\\\_url": "https://ads.example.com/vast?tag=midroll-promo-1",
        "playback\\\\\\\_timestamp": 1748177500000,
        "time\\\\\\\_sent": 1748177500142,
        "viewer\\\\\\\_count": 1843
      }
    ]
  },
  "statuscode": 200,
  "success": true,
  "data": null
}
```

|Field|Description|
|-|-|
|`count`|Number of messages in the response.|
|`messages`|Array of broadcast messages, sorted ascending by `time\\\\\\\_sent`.|
|`messages\\\\\\\[].id`|Stable identifier for the broadcast, matching the `messageId` returned by `POST`.|
|`messages\\\\\\\[].time\\\\\\\_sent`|Server timestamp (Unix ms) when the broadcast was emitted.|
|`messages\\\\\\\[].viewer\\\\\\\_count`|Number of viewers connected at broadcast time.|

### Error responses

|Status|When|
|-|-|
|`400 Bad Request`|`id` is not a positive integer.|
|`502 Bad Gateway`|Messaging service temporarily unreachable. Safe to retry.|

`GET` requires no authentication, so there are no `401`/`403` responses.
An unknown livestream ID is not an error either — it returns `200 OK` with
an empty `messages` array, so a mistyped ID looks identical to a livestream
that has had no broadcasts.

\---

## DELETE /livestreams/{id}/messages/{messageId}

Cancels an active ad break. Viewers' players tear down the ad and resume
live video on the next frame.

Use this to:

* End an ad break early (e.g. play resumed sooner than expected).
* Cancel an ad that was triggered by mistake.

Note that the deleted message is also removed from the history returned by
`GET /livestreams/{id}/messages`. If you use that history for reporting,
record anything you need about the broadcast before cancelling it.

Cancellation is delivered with the same at-most-once semantics as the
original broadcast: a viewer whose connection has briefly dropped at the
moment of the `DELETE` will not receive the cancellation and will play the
ad to completion. Unlike `POST`, the `DELETE` response carries no delivery
counts, so there is no way to see how many viewers the cancellation
reached.

### Path parameters

|Name|Type|Description|
|-|-|-|
|`id`|integer|Numeric Staylive livestream ID.|
|`messageId`|string|The `messageId` returned by the original `POST`.|

### Authentication

Same as `POST` — JWT in the `Authorization` header, with one of the
elevated roles listed under [Authentication](#authentication).

### Example request

```bash
curl -X DELETE "https://api.staylive.tv/livestreams/8451/messages/7f9c1b2e-4d3a-4f6b-9e8d-2a1c5b7d9e0f" \\\\\\\\
  -H "Authorization: Bearer $STAYLIVE\\\\\\\_TOKEN"
```

### Example response — `200 OK`

```json
{
  "message": { "success": true },
  "statuscode": 200,
  "success": true,
  "data": null
}
```

### Error responses

|Status|When|
|-|-|
|`401 Unauthorized`|Missing, expired, or invalid token.|
|`403 Forbidden`|Token does not have an elevated role on the livestream — also returned when the livestream ID does not exist.|
|`404 Not Found`|No message with the given `messageId` exists for this livestream.|
|`502 Bad Gateway`|Messaging service temporarily unreachable. Safe to retry.|

\---

## Operational notes

* **Delivery is at-most-once.** A broadcast reaches every viewer connected
at the instant of the `POST`. Viewers whose connection has briefly
dropped at that moment will not receive the trigger and are not caught
up afterwards. The `failed` counter in the `POST` response reflects per-
viewer send failures. The same applies to `DELETE` cancellations: a
viewer who misses the cancellation plays the ad to completion.
* **Missed messages are not replayed on reconnect.** `GET` will always
return the broadcast in its history, but viewers who were offline at the
moment of the trigger do not see the ad when they reconnect.
* **No automatic retries.** A successful `POST` should not be reissued —
viewers who already received the first trigger will receive a duplicate.
Use `DELETE` to cancel, then `POST` a fresh trigger if you need to retry.
* **Retrying safely after a `502`.** A `502` usually means the trigger was
not broadcast, but it is not a guarantee — the failure can occur after
delivery (for example, the connection drops after viewers were reached).
Blindly re-posting therefore risks a double ad break. To retry safely,
include a unique field of your own in every `PLAY\\\\\\\_AD` payload (e.g.
`"client\\\\\\\_reference": "<uuid>"` — extra fields are stored with the
message), and on a `502` call `GET /livestreams/{id}/messages` first: if
a message with your reference is already in the history, the broadcast
went out and you should not re-post.
* **Ad break length is determined by the VAST creative**, not by this API.
Staylive does not synthesize an end-of-ad event; the player resumes live
playback when the VAST creative finishes or when `DELETE` is called.

