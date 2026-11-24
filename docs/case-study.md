<p align="center"><img src="assets/banner.svg" alt="Webhook Automations for WordPress" width="100%"></p>

# Webhook Automations for WordPress: Client Case Study

> Integrations that survive outages, can be verified by the receiver and can be debugged in minutes.

> **Status of this document.** This is a reference case study: it sets out the business problem, the solution and the value model. The plugin exists and is tested against a simulated receiver (see the [research report](research-report.md)), but it has not been used in a client engagement. The results table is intentionally blank and is filled in only from measured data (a pilot or a real engagement). No outcome figures are claimed here.

## Snapshot

|  |  |
|---|---|
| **Who it is for** | Agencies and site owners who connect WordPress to CRMs, chat tools, spreadsheets and internal systems. |
| **Solution** | Send signed, reliable webhooks from WordPress events, with retries, replay protection and a delivery log. |
| **Track** | Business · Advanced |
| **Delivered by** | The Diamond Empire |

## The challenge

Integrations break silently when a receiver is down, nobody can verify that a request really came from the site, and debugging means reading server logs.

## The solution

A WordPress plugin that turns site events into signed webhooks to any service. It queues deliveries, retries failures with backoff and jitter, gives every delivery an idempotency key so receivers can de-duplicate, signs every request so receivers can verify it, and keeps a searchable, redacting delivery log with manual replay. Delivery is at-least-once: exactly-once effects need de-duplication on the receiving side.

What the client gets in practice:

- Event catalogue: post and page lifecycle, user registration, comments, and any custom action hook
- Payload templates (JSON with placeholders and a few filters) and per-endpoint rate limits; developer filters for further control
- HMAC-SHA256 request signatures with a timestamp and a replay window; secret rotation
- Delivery queue with exponential backoff and jitter, per-endpoint rate limits and a dead-letter queue
- Idempotency keys so receivers can safely de-duplicate retried deliveries

## How it creates value

| Value lever | How the solution delivers it | How we measure it |
|---|---|---|
| Lost events | Queued, retried and replayable deliveries | Events lost per month |
| Integration trust | Signed requests with replay protection | Unverified requests accepted by receivers |
| Debugging time | A searchable log with one-click replay | Hours to diagnose a failed integration |
| Integration cost | One plugin replaces custom code or a middleman subscription | Monthly spend on integration middleware |

## How an engagement runs

| Phase | What happens |
|---|---|
| Map | Events and destinations |
| Build | Endpoints, payloads and signing |
| Test | Failure drills against the client’s receivers |
| Operate | Monitor queue health and failures |

## What the client receives

- Installable plugin ZIP (GPL-2.0-or-later). A licence-key or update service is not part of the product as built; any commercial terms would be agreed separately
- Endpoints and payload mappings built with the client
- A precise description of the signature scheme and a reference verifier in Node (other languages implement the same three lines of HMAC)
- A queue health screen with queue age and a stalled-queue warning (no e-mail or chat alerting is built; alerting would be set up outside the plugin, for example by monitoring the health route)
- Short operations guide

## Results

Baseline measurements are taken **before** the solution goes live, and the same measures are taken after, so every change in the table is attributable and honest.

| Measure | How measured | Baseline | After | Change |
|---|---|---|---|---|
| Lost events | Events lost per month | — | — | — |
| Integration trust | Unverified requests accepted by receivers | — | — | — |
| Debugging time | Hours to diagnose a failed integration | — | — | — |
| Integration cost | Monthly spend on integration middleware | — | — | — |

> **To fill in:** replace each — with measured values and add the measurement window and sample size beneath the table.

## How quality and risk are managed

- Failure-injection tests against a simulated receiver: errors, timeouts, dropped connections, lost acknowledgements, redirects, oversized responses
- Signature verification tests: tampered body, wrong secret, stale timestamp, replayed delivery
- Event-coverage test that triggers every supported event in WordPress and checks its payload against the documented schema
- Clean uninstall verified; no secrets in logs

## Is this a fit?

Best suited to: agencies and site owners who connect WordPress to CRMs, chat tools, spreadsheets and internal systems.

## Talk to me

Want this adapted to your business? Get in touch via **[thedempire.shop](https://thedempire.shop/)**.

---

[README](../README.md) · [Client case study](case-study.md) · [Research report](research-report.md)
