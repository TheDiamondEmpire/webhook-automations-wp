# Webhook Automations for WordPress: Webhook Reference

<p align="center"><img src="assets/banner.svg" alt="Webhook Automations for WordPress" width="100%"></p>

This page documents what the plugin sends and how a receiver should check it. Everything here describes behaviour that is covered by the automated tests in this repository (see the README, Verification section).

## Delivery guarantee

Delivery is **at-least-once**, not exactly-once. A request can be sent more than once when the receiver processes it but the response is lost (timeout, dropped connection, a 5xx after processing), or when the site process stops between sending and recording the result. Every delivery therefore carries a stable **idempotency key**. A receiver that stores the keys it has processed gets exactly-once **effects**. Experiment E1 measures this with a simulated receiver; see the research report.

## Request

`POST` to the endpoint URL with a JSON body (UTF-8) and these headers:

| Header | Meaning |
|---|---|
| `Content-Type` | `application/json; charset=utf-8` |
| `User-Agent` | `WebhookAutomationsWP/<version>` |
| `X-Wawp-Event` | Event name, for example `post.published` (convenience copy; the signed body also has it) |
| `X-Wawp-Delivery` | Delivery id (UUID). Same for every attempt of one delivery. A manual replay gets a new one. |
| `X-Wawp-Idempotency-Key` | Stable key of the event occurrence (convenience copy of `idempotency_key` in the body) |
| `X-Wawp-Attempt` | 1 for the first attempt, 2 for the first retry, and so on |
| `X-Wawp-Timestamp` | Unix time in seconds when this attempt was signed. Changes on every attempt. |
| `X-Wawp-Signature` | `v1=<hex>`; two comma-separated `v1=` values while a rotated secret is in its grace period |

Only the body and the timestamp are covered by the signature. The `X-Wawp-*` copies of the event, delivery id and idempotency key are not signed: read those values from the signed body.

### Body (the "envelope")

```json
{
  "id": "2be2fc8c-ec62-4bd7-9c4f-aaba8ab3fc83",
  "idempotency_key": "evt_0ea4bcf0d979a81c0d83b995",
  "event": "post.published",
  "created_at": "2026-10-23T17:49:04Z",
  "site_url": "https://example.com",
  "data": { }
}
```

`replay_of` (the id of the original delivery) is added on manual replays. The machine-readable schemas for the envelope and for the `data` of every event are in [event-schemas.json](event-schemas.json).

## Signature scheme (version `v1`)

```
signed_string = X-Wawp-Timestamp + "." + raw request body bytes
signature     = hex( HMAC-SHA256( key = signing secret, message = signed_string ) )
X-Wawp-Signature: v1=<signature>[,v1=<signature with the previous secret>]
```

The signing secret is the full string shown once when the endpoint is created or its secret is rotated (it starts with `whsec_`); use it as-is as the key, UTF-8 encoded.

A receiver must:

1. Read the **raw** body bytes. Re-serialised JSON will not match.
2. Recompute the HMAC for each secret it accepts, and compare with every `v1=` value in **constant time**.
3. Reject the request if `X-Wawp-Timestamp` is more than its tolerance away from the receiver's clock (300 seconds is a sensible default; the reference verifier uses 300 and treats exactly 300 as acceptable).
4. Optionally remember accepted signatures for the tolerance window and reject an exact repeat (replay protection). A legitimate retry is re-signed with a new timestamp, so it has a different signature.
5. De-duplicate effects by `idempotency_key`.

Check a signature by hand (this is how the known-answer test vector in `tests/verifier.unit.test.mjs` was computed independently of the Node code):

```
printf '%s' '1700000000.{"hello":"world"}' | openssl dgst -sha256 -hmac 'whsec_test_secret'
```

### Reference verifier (Node)

`verifier/webhook-verifier.mjs` has no dependencies.

```js
import { verify, ReplayStore } from './verifier/webhook-verifier.mjs';

const store = new ReplayStore();
const result = verify({
  secrets: [currentSecret, previousSecretIfRotating],
  headers: req.headers,
  rawBody,                 // Buffer with the exact bytes received
  toleranceSeconds: 300,
  replayStore: store,
});
// result = { ok: true|false, reason: 'ok' | 'missing_headers' | 'malformed_timestamp' | 'bad_signature' | 'stale_timestamp' | 'replayed' }
```

### Secret rotation

Rotating creates a new secret (shown once). For the grace period (default 24 hours, setting `secret_grace`) each request carries two `v1=` values, one per secret, so a receiver can switch without dropping deliveries. After the grace period only the new secret signs.

## Idempotency keys and replay

* The key identifies one event occurrence. It is identical across all attempts of a delivery and across endpoints that received the same occurrence. It is generated per occurrence, so two separate publishes of the same post have different keys.
* A manual replay creates a **new delivery** (new `id`, `replay_of` set) with the **same** key. A receiver that already processed the event will de-duplicate it; a receiver that lost it will process it.
* Replay needs the stored raw payload. It is kept for `body_retention_days` (default 7) after the last change; after that only the redacted copy remains and replay is refused.

## Retry policy

| Outcome | Action |
|---|---|
| HTTP 2xx | Delivered |
| Timeout, connection error, HTTP 408, 425, 429, 5xx | Retry with backoff until `max_attempts` (default 6), then dead-letter |
| HTTP 3xx | Not followed; dead-letter (`redirect_not_followed`) |
| Other HTTP 4xx | Dead-letter immediately |
| URL refused by the safety check at send time | Dead-letter immediately |

Backoff: after the n-th failed attempt, `nominal(n) = min(max_delay, base_delay * 2^(n-1))` and `delay(n) = round(nominal(n) * (1 + jitter * (2u - 1)))` with `u` uniform in [0, 1), at least 1 second. Defaults: `base_delay` 30 s, `max_delay` 21600 s, `jitter` 0.2, so the first retries wait about 30, 60, 120, 240 and 480 seconds (plus or minus 20%). The cap applies to the nominal value, so a jittered delay can exceed `max_delay` by up to the jitter fraction. A `Retry-After` header (seconds or HTTP date) on a retryable response raises the delay, up to `max_delay`.

## Events

| Event | Fired when |
|---|---|
| `post.published`, `page.published` | A post or page becomes published |
| `post.updated`, `page.updated` | A published post or page is updated |
| `post.trashed`, `page.trashed` | Moved to the trash |
| `post.deleted`, `page.deleted` | Permanently deleted |
| `user.registered` | A user is created |
| `comment.created` | A comment is inserted (spam and trash are not sent) |
| `comment.approved` | A comment becomes approved |
| `custom.<name>` | Your code runs `do_action( 'wawp_trigger', '<name>', $data )` or `wawp_trigger_event( '<name>', $data )` |
| `webhook.test` | "Send test" in the admin; sent to the chosen endpoint regardless of its subscriptions |

Subscriptions can be exact names, a group wildcard (`post.*`, `page.*`, `user.*`, `comment.*`, `custom.*`) or `*`. Revisions and drafts do not produce events (tested); autosaves and auto-drafts are excluded in the code but were not tested separately. Only the `post` and `page` post types are covered by default (filter `wawp_post_types`); other plugins, including WooCommerce and form plugins, are not integrated and were not tested: send their events yourself through the custom hook.

### Custom events

```php
do_action( 'wawp_trigger', 'order_shipped', array( 'order_id' => 123 ) );
// or
wawp_trigger_event( 'order_shipped', array( 'order_id' => 123 ) );
```

Names use lowercase letters, digits, dot, dash and underscore (1 to 64 characters). The delivery event is `custom.order_shipped` and `data` is `{ "custom": { "name": "order_shipped", "data": { ... } } }`.

## Payload templates

An endpoint can carry a JSON template that replaces `data` entirely. Strings may contain `{{ path.to.value }}` placeholders resolved against the event context (`post`, `user`, `comment` or `custom`, plus `event` and `site`). A string that is exactly one placeholder keeps the value's type. Filters follow a pipe: `upper`, `lower`, `trim`, `length`, `json`, `default:"text"`. The post `content` is available to templates but is not part of the default payload. Templates are data; nothing in them is executed.

```json
{ "id": "{{ post.id }}", "heading": "New: {{ post.title | upper }}", "who": "{{ post.author.display_name | default:\"unknown\" }}" }
```

## Hooks for developers

| Hook | Type | Purpose |
|---|---|---|
| `wawp_trigger` | action | Fire a custom event (name, data) |
| `wawp_event_context` | filter | Change the template context of an event |
| `wawp_should_deliver` | filter | Return false to skip delivering an event to an endpoint |
| `wawp_payload` | filter | Change the final `data` for an endpoint |
| `wawp_post_types` | filter | Post types that produce `post.*` / `page.*` events |
| `wawp_resolved_ips` | filter | Override the DNS answer used by the URL safety check (testing) |

## REST API

Namespace `wawp/v1`. Every route requires an administrator (`manage_options`) with the standard `X-WP-Nonce` header, except `POST /queue/run`, which also accepts the header `X-Wawp-Cron-Token`.

| Route | Purpose |
|---|---|
| `GET /events` | Event catalogue |
| `GET, POST /endpoints` | List and create (the response to create has the one-time `secret`) |
| `GET, PUT, DELETE /endpoints/{id}` | Read, update, delete |
| `POST /endpoints/{id}/rotate-secret` | New secret (one-time `secret` in the response) |
| `POST /endpoints/{id}/test` | Send a test delivery now |
| `GET /deliveries` | Log (filters `status`, `endpoint_id`, `event`, `q`; `page`, `per_page`) |
| `GET /deliveries/{id}` | One delivery with its attempt log and the redacted payload |
| `POST /deliveries/{id}/replay` | Replay (new delivery, same idempotency key) |
| `POST /deliveries/{id}/retry-now` | Make a waiting delivery due now |
| `GET /health` | Queue counts, queue age, stalled flag, cron information |
| `POST /queue/run` | Run the queue now |
| `POST /cron-token` | Create a cron token (shown once, stored hashed) |
| `GET, PUT /settings` | Settings |

## Running the queue from a real cron job

WP-Cron only runs when the site receives requests, so a quiet site can leave deliveries waiting. The Queue health screen shows the age of the oldest due delivery and flags the queue as stalled when it passes `stale_queue_seconds`. For dependable timing create a token in Settings and call, every minute:

```
curl -X POST -H "X-Wawp-Cron-Token: <token>" https://example.com/wp-json/wawp/v1/queue/run
```

PHP code can call `wawp_run_queue( $limit )` directly. No WP-CLI command is provided.

## Safety checks on endpoint URLs

By default the URL must be `https`, must not contain credentials, and the host and every address it resolves to must be public (private, loopback, link-local, carrier-grade NAT, multicast and reserved ranges are refused, as are numeric host tricks such as `2130706433`). The check runs when an endpoint is saved and again before every send. Redirects are never followed, the response read is capped by a transport setting and the HTTP timeout is capped at 30 seconds. Setting `allow_private_urls` is an explicit local-testing opt-in that also allows plain http. Known limit: a hostile DNS server could answer differently between the check and the connection (DNS rebinding).
