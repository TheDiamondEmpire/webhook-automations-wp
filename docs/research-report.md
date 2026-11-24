# Webhook Automations for WordPress: Research Report

> **Where the code is.** The experiment scripts that produced these results are part of the source package, which is distributed separately (see [SOURCE-ACCESS.md](../SOURCE-ACCESS.md)). The raw results they wrote are included in this repository under [`results/`](../results/).

<p align="center"><img src="assets/banner.svg" alt="Webhook Automations for WordPress" width="100%"></p>

> **Status: protocol v0.2, results recorded.** Hypotheses, methods and the statistical plan (sections 2 to 6) were fixed before the final run; the deviations since v0.1 are listed in the change log. All results below come from files in [results/](../results/) produced by scripts in this repository. **The receiver and every failure are SIMULATED by a local Node test receiver; WordPress, the plugin and the HTTP requests are real.** Results describe the plugin under these simulated conditions on one loaded machine. They are not field measurements.

## Abstract

This study asks: how reliably does the plugin deliver webhook events under receiver failures, does its retry schedule behave as specified, and does its signature scheme resist tampering and replay? A WordPress plugin (WordPress 7.1.3 under WordPress Playground, PHP 8.3, SQLite layer) was exercised by a Node harness against a local receiver that injects seeded failures. With seeded failures on 2,880 events, the queue delivered 95.3% to 100% of events (retry budget 3 or 6) against 67.2% to 93.8% for a naive single-attempt sender, at failure probabilities of 0.5 down to 0.1; events were still lost whenever all attempts failed, at the rate a simple model predicts. Retries produced duplicate effects (4 to 28 per 100 events) at a receiver that ignores the idempotency key and none at a receiver that stores it. The retry schedule matched its specification in all 2,160 recorded intervals (accelerated time, not wall-clock). The reference verifier gave the expected result in all 1,100 attack and control trials on 100 real signed deliveries, and all 13 documented events produced a schema-valid, signed payload in all 3 repetitions. The receiver and every failure are simulated.

## 1. Research question

How reliably does the plugin deliver webhook events under receiver failures, does its retry schedule behave as specified, and does its signature scheme resist tampering and replay?

## 2. Hypotheses

- H1: Under injected receiver failures (5xx errors, timeouts, connection drops), every queued event is delivered at least once within its retry budget, and with idempotency-key de-duplication on the receiver, every event has exactly one effect.
- H2: The measured retry schedule matches the specified exponential backoff with jitter bounds.
- H3: Tampered bodies, wrong secrets, stale timestamps and replayed deliveries are all rejected by a reference verifier; valid deliveries are always accepted.
- H4: Every supported event type produces a correct payload that matches its documented schema.

H1 contains two claims that need different handling. "Delivered at least once within its retry budget" cannot hold for an event whose every attempt fails; the study therefore reports the lost fraction and compares it with the value the failure model predicts, rather than treating any loss as a refutation. "Exactly one effect with de-duplication" is a property of the stable idempotency key and is checked directly.

## 3. Background

Webhook senders converge on the same ingredients: a signature that covers a timestamp and the raw body, a tolerance window against replay, retries with exponential backoff and jitter, and receiver-side de-duplication by an event or idempotency identifier.

- Stripe documents signing with HMAC-SHA256 over `timestamp + "." + payload`, a default tolerance of five minutes, a fresh timestamp and signature on every retry, one signature per active secret while a secret is rolled, and de-duplication by logging processed event ids [1]. The plugin's scheme follows this shape (header names and encoding differ).
- Standard Webhooks is an open project that states that webhook implementations must protect against SSRF, spoofing and replay [2]. Its detailed header names were not verified for this report.
- HMAC is specified in RFC 2104 (Krawczyk, Bellare and Canetti, 1997, informational) [3].
- Marc Brooker's analysis of exponential backoff with jitter found that jitter greatly reduces contention compared with plain exponential backoff and that "equal jitter" performed worse than "full jitter" and decorrelated jitter in his simulation [4]. The plugin uses symmetric jitter around the nominal delay (plus or minus a fraction, default 20%), which is closer to "equal jitter" than to full jitter. This study checks conformance to the plugin's own specification; it does not claim the schedule is optimal for contention.
- An Internet-Draft of the IETF HTTPAPI working group proposes an `Idempotency-Key` request header so that non-idempotent requests can be retried safely [5]. The plugin sends `X-Wawp-Idempotency-Key` and the same value inside the signed body; it does not implement that draft.
- The OWASP SSRF prevention cheat sheet recommends disabling redirects, blocking private, loopback, link-local and metadata ranges, and, against DNS rebinding, connecting only to a validated address [6]. The plugin blocks the ranges and redirects but does not pin the connection to the validated address, which is a known residual risk.
- The WordPress plugin handbook states that WP-Cron runs only when a page loads, so tasks cannot be guaranteed to run at an exact time [7].

## 4. Method

### 4.1 System under test

A WordPress plugin that turns site events into signed webhooks to endpoints the site owner configures. It queues deliveries in a custom table, retries failures with exponential backoff and jitter, gives every delivery a stable idempotency key, signs every request (HMAC-SHA256 over timestamp and body), and keeps a redacting delivery log with manual replay. Delivery is at-least-once. Version 0.1.0.

### 4.2 Environment

A real WordPress (7.1.3) in WordPress Playground CLI 3.1.60 (WordPress and PHP compiled to WebAssembly, under Node 25), PHP 8.3, SQLite database layer, two request workers. The plugin sends real `wp_remote_post` requests to a local Node receiver on 127.0.0.1; no transport workaround was needed. The receiver verifies signatures with the reference verifier and injects failures on a seeded schedule. WP-Cron's automatic spawning is switched off during experiments and the harness drives the queue explicitly (a separate test shows that WP-Cron itself does run the queue). Retry timing is observed in **accelerated time**: a test-only clock offset that the plugin honours only when the constant `WAWP_TEST_CLOCK` is defined by a must-use plugin (see the README for the guard), advanced by the harness to the next due time instead of waiting. The machine was shared and heavily loaded.

### 4.3 Baselines

- A naive single-attempt `wp_remote_post` with no queue and no signature (a test fixture, not product code), measured in E1.
- Queue without receiver-side use of the idempotency key (the "queue-only" view of the same traffic), measured in E1.
- Unsigned deliveries (a receiver with no verifier), measured in E3. This baseline is definitional: it accepts everything.

### 4.4 Independent variables

- E1: failure rate p in {0.1, 0.3, 0.5}; retry budget (maximum attempts) in {1 (naive), 3, 6}; seed (8 seeds); 40 events per run.
- E2: jitter in {0, 0.2}; seed (3); 40 deliveries per run; 10 attempts per delivery; base delay 30 s; cap 600 s.
- E3: tamper and replay case; 100 captured real deliveries.
- E4: event type (13 documented events and 2 negative controls); 3 repetitions.

### 4.5 Metrics

- E1: events with at least one effect at the receiver ("delivered"), events with none ("lost"), duplicate effects per 100 events with and without receiver-side de-duplication, requests per event, plugin-reported status.
- E2: nominal delay correctness, delay within jitter bounds, attempt never before its scheduled time, lateness, jitter distribution.
- E3: verifier accept and reject correctness per case.
- E4: exactly one delivery per event, schema validity, signature validity, silence of negative controls.

### 4.6 Failure model (E1)

For each attempt the receiver draws from a seeded generator keyed by seed, event and attempt index. With probability p the attempt fails, as one of: HTTP 500 before processing (35%), connection reset before processing (25%), HTTP 500 after processing (20%), reset after processing (10%), no answer within the sender's 1 s timeout (10%, the receiver closes the connection after 1.6 s). Attempts that fail "after processing" apply the effect but the sender never learns it: these produce duplicates on retry. Naive and queue conditions see the same fate for the first attempt of the same event and seed.

"Queue-only" and "full" are two receiver policies applied to the same plugin traffic in the same run: queue-only applies every request it receives; full applies a request only the first time it sees the idempotency key. The plugin always sends the key.

## 5. Experiment matrix

| ID | Experiment | Manipulation | Measured |
|---|---|---|---|
| E1 | Delivery under failure | Seeded failure schedules against naive, queue-only and full configurations | Delivered, duplicates, lost |
| E2 | Retry schedule conformance | Record attempts for failing endpoints in accelerated time | Intervals versus specification |
| E3 | Signature and replay checks | Tamper, wrong secret, stale and replayed cases | Accept and reject correctness |
| E4 | Event coverage | Trigger each supported event | Payload schema conformance |

## 6. Statistical plan

- **Replication:** run counts are fixed in the experiment scripts before data collection (E1: 8 seeds per condition; E2: 3 seeds per condition; E3: 100 deliveries; E4: 3 repetitions) and reported.
- **Summaries:** means over runs with percentile-bootstrap 95% confidence intervals (10,000 resamples, seeded); pooled proportions with Wilson 95% intervals. For E1 and E2 the unit of replication is the run (seed), because events within a run share a queue.
- **Comparisons:** paired by seed, exact two-sided Wilcoxon signed-rank test on the per-run delivered fractions, queue versus naive at the same failure rate (zero differences dropped).
- **Effect sizes:** Cliff's delta alongside p-values.
- **Multiple comparisons:** Holm-Bonferroni across the six E1 comparisons.
- **Uniformity (E2):** Kolmogorov-Smirnov distance of the jitter ratio against the uniform distribution on [1 - jitter, 1 + jitter], using only intervals whose nominal delay is at least 240 s so that rounding to whole seconds is negligible.
- **No silent exclusions:** no run or event was excluded.
- **Negative results are results:** where a hypothesis is not supported, the report says so.

## 7. Results

All numbers are produced by `npm run experiments` and printed by `npm run report`. Raw per-run data: `results/e1-raw.jsonl`, `results/e2-raw.jsonl`, `results/e3-raw.jsonl`, `results/e4-raw.jsonl`; summaries: `results/e*-summary.json`; run record: `results/run-metadata.json`. **Receiver and failures are simulated in every experiment.**

### E1. Delivery under failure

**Manipulation:** seeded failure schedules (failure probability p per attempt) against a naive single-attempt sender and the plugin's queue with a retry budget of 3 or 6 attempts. **Measured:** delivered, lost, duplicate effects with and without receiver-side de-duplication. 8 seeds per condition, 40 events per run, so 320 events per condition (2,880 events in total). Source: `results/e1-raw.jsonl`, `results/e1-summary.json`. **Simulated receiver and failures.**

| Condition (p = failure probability) | n (runs x events) | Delivered, mean over runs | 95% CI (bootstrap) | Lost events (pooled) and Wilson 95% | Model-predicted lost | Notes |
|---|---|---|---|---|---|---|
| naive, p=0.1 | 8 x 40 | 93.8% | [91.3%, 96.3%] | 20/320 = 6.3% [4.1%, 9.5%] | 7.0% | single attempt, 1.00 requests per event |
| queue B=3, p=0.1 | 8 x 40 | 100.0% | [100.0%, 100.0%] | 0/320 [0%, 1.2%] | 0.03% | 1.106 requests per event |
| queue B=6, p=0.1 | 8 x 40 | 100.0% | [100.0%, 100.0%] | 0/320 [0%, 1.2%] | 0.00% | 1.106 requests per event |
| naive, p=0.3 | 8 x 40 | 81.3% | [77.5%, 85.3%] | 60/320 = 18.8% [14.9%, 23.4%] | 21.0% | |
| queue B=3, p=0.3 | 8 x 40 | 99.1% | [98.1%, 99.7%] | 3/320 = 0.9% [0.3%, 2.7%] | 0.93% | 1.350 requests per event |
| queue B=6, p=0.3 | 8 x 40 | 100.0% | [100.0%, 100.0%] | 0/320 [0%, 1.2%] | 0.01% | 1.372 requests per event |
| naive, p=0.5 | 8 x 40 | 67.2% | [63.4%, 70.9%] | 105/320 = 32.8% [27.9%, 38.1%] | 35.0% | |
| queue B=3, p=0.5 | 8 x 40 | 95.3% | [92.5%, 97.5%] | 15/320 = 4.7% [2.9%, 7.6%] | 4.3% | 1.741 requests per event |
| queue B=6, p=0.5 | 8 x 40 | 99.4% | [98.4%, 100.0%] | 2/320 = 0.6% [0.2%, 2.3%] | 0.18% | 1.934 requests per event |

"Model-predicted lost" is the closed-form value for independent attempts, in which an attempt leaves no effect with probability 0.7 p (the three failure kinds that act before processing): 0.7 p for the naive sender and (0.7 p)^B for the queue. Every model value lies inside the Wilson interval of the matching observed pooled loss (the closest is 0.18% against [0.17%, 2.25%] for the last queue row).

Duplicate effects (events applied more than once) per 100 events, mean over runs with bootstrap 95% CI:

| Condition | Receiver ignores the key ("queue-only") | Receiver de-duplicates by key ("full") | Events with a duplicate effect (of 320), without / with de-duplication |
|---|---|---|---|
| naive (any p) | 0 [0, 0] | not applicable (no key is sent) | 0 / 0 |
| queue B=3 or 6, p=0.1 | 4.06 [1.88, 6.25] | 0 | 13 / 0 |
| queue B=3, p=0.3 | 10.94 [6.56, 14.69] | 0 | 34 / 0 |
| queue B=6, p=0.3 | 12.19 [8.44, 15.31] | 0 | 36 / 0 |
| queue B=3, p=0.5 | 20.63 [14.38, 26.88] | 0 | 62 / 0 |
| queue B=6, p=0.5 | 27.81 [18.75, 38.13] | 0 | 73 / 0 |

Paired comparisons, delivered fraction per seed, queue versus naive at the same p (exact Wilcoxon signed-rank, 8 pairs, Holm-adjusted over the six comparisons): every comparison has raw p = 0.0156 (p=0.1) or 0.0078 (p=0.3, 0.5), Holm-adjusted p = 0.0469, Cliff's delta between 0.875 and 1.0; median differences in delivered fraction: p=0.1: +0.0625 (B=3 and B=6); p=0.3: +0.175 (B=3), +0.1875 (B=6); p=0.5: +0.275 (B=3), +0.325 (B=6). With 8 pairs the smallest attainable two-sided exact p is 0.0078, so these are the strongest results the design can return.

What the plugin itself reported (queue conditions, all runs reached an idle queue, 0 deliveries left waiting): delivered 320, 320 (p=0.1); 313, 320 (p=0.3); 285, 312 (p=0.5) for B=3, B=6; dead-lettered 0, 0; 7, 0; 35, 8. Of the dead-lettered deliveries, 0, 0; 4, 0; 20, 6 had in fact been **applied by the receiver** (the final attempt's response was lost), so "dead" does not always mean "not delivered".

Receiver-side signature checks: in the B=6 conditions at p=0.3 and 0.5 the harness receiver rejected 2 and 28 requests as unverifiable although the plugin's signatures were correct; no failures occurred anywhere else. This is a harness effect of accelerated time: the receiver's check used the real clock while the plugin's clock had been advanced by the accumulated retry delays (more than 300 s before the fifth attempt). The rejection reasons were not recorded in this run, so this explanation is consistent with the data (it appears only where the cumulative delay exceeds the 300 s tolerance, and 28 is close to the roughly 30 requests expected at the fifth and sixth attempt at p=0.5, 320 x (0.5^4 + 0.5^5)) but was not directly confirmed. It did not change any delivery outcome, because the simulated receiver applies a request whether or not verification succeeds.

### E2. Retry schedule conformance (accelerated time)

**Manipulation:** an always-failing (HTTP 500) simulated receiver; 40 deliveries per run, 10 attempts per delivery; base delay 30 s, cap 600 s; jitter 0 or 0.2; 3 seeds each. Time was accelerated by the guarded test clock. **Measured:** the recorded schedule against the specification. Source: `results/e2-raw.jsonl`, `results/e2-summary.json`. **Simulated receiver; accelerated time.**

| Condition | n (retry intervals) | Result | 95% CI | Notes |
|---|---|---|---|---|
| jitter 0: nominal delay equals min(600, 30 x 2^(n-1)) | 1080 | 1080/1080 | Wilson [99.6%, 100%] | 480 intervals sit on the 600 s cap |
| jitter 0: delay within bounds (exact) | 1080 | 1080/1080 | Wilson [99.6%, 100%] | ratio delay/nominal = 1 for every interval |
| jitter 0.2: nominal delay correct | 1080 | 1080/1080 | Wilson [99.6%, 100%] | |
| jitter 0.2: delay within [0.8, 1.2] x nominal | 1080 | 1080/1080 | Wilson [99.6%, 100%] | observed ratio min 0.80, max 1.20 |
| jitter 0.2: mean of delay/nominal | 1080 | 1.0016 | bootstrap [0.9945, 1.0086] | expected 1.0 for symmetric jitter |
| jitter 0.2: Kolmogorov-Smirnov distance to uniform | 720 (nominal >= 240 s) | d = 0.0278 | critical value at 5% = 0.0506 | uniform not rejected |
| Attempt never before its scheduled time | 2160 (both jitter settings) | 2160/2160 | Wilson [99.8%, 100%] | |
| Deliveries dead-lettered after exactly the maximum attempts | 240 (120 per jitter setting) | 240/240 | Wilson [98.4%, 100%] | no retry scheduled after the last attempt |
| Lateness (actual gap minus scheduled delay) | 2160 | median 0 s | p95 2 s (jitter 0), 1 s (jitter 0.2); max 3 s (jitter 0), 1 s (jitter 0.2) | caused by serial processing in the harness, not by the schedule |

What this does **not** show: real waiting. The clock was jumped, so nothing here measures WP-Cron's real latency, behaviour on an idle site, or clock skew.

### E3. Signature and replay checks

**Manipulation:** 100 real signed deliveries produced by the plugin were captured and then attacked or re-presented. **Measured:** whether the reference verifier (and, for one replay case, the live receiver) gave the expected outcome. Source: `results/e3-raw.jsonl`, `results/e3-summary.json`. **The attacks are applied by the harness to real captured requests; the receiver is simulated.**

| Condition | n | Result (correct outcome) | 95% CI (Wilson) | Notes |
|---|---|---|---|---|
| Valid control accepted | 100 | 100/100 | [96.3%, 100%] | all 100 were also accepted by the live receiver on first arrival; Node recomputed all 100 PHP signatures identically |
| One byte of the body flipped, rejected | 100 | 100/100 | [96.3%, 100%] | position chosen by a seeded generator |
| Body truncated by one byte, rejected | 100 | 100/100 | [96.3%, 100%] | |
| Wrong secret, rejected | 100 | 100/100 | [96.3%, 100%] | |
| Timestamp header altered, rejected | 100 | 100/100 | [96.3%, 100%] | rejected as a bad signature |
| Signature header altered, rejected | 100 | 100/100 | [96.3%, 100%] | |
| Valid signature, timestamp 301 s old, rejected as stale | 100 | 100/100 | [96.3%, 100%] | |
| Valid signature, timestamp 301 s in the future, rejected | 100 | 100/100 | [96.3%, 100%] | |
| Valid signature, timestamp exactly 300 s old, accepted (boundary) | 100 | 100/100 | [96.3%, 100%] | tolerance is inclusive |
| Second presentation of the same request to a local replay store, rejected | 100 | 100/100 | [96.3%, 100%] | |
| Same request re-sent over HTTP to the live receiver, rejected as replayed | 100 | 100/100 | [96.3%, 100%] | |
| Unsigned baseline (no verifier) rejects any of the nine attack cases above | 900 | 0/900 | not applicable | definitional: with no verifier everything is accepted |

All outcomes were as expected; no case produced an unexpected result. A 100% observed rate over 100 trials still leaves a Wilson lower bound of 96.3%.

### E4. Event coverage

**Manipulation:** each documented event was triggered by real WordPress actions (create, update, trash and delete a post and a page; register a user; add and approve a comment; fire the custom hook; send a test delivery), three repetitions, plus two negative controls. **Measured:** exactly one delivery per trigger, payload validity against `docs/event-schemas.json` (Ajv), and signature validity. Source: `results/e4-raw.jsonl`, `results/e4-summary.json`. **Simulated receiver.**

| Condition | n (repetitions) | Result | 95% CI (Wilson) | Notes |
|---|---|---|---|---|
| post.published, post.updated, post.trashed, post.deleted | 3 each | 3/3 each | [43.9%, 100%] | exactly one delivery, schema-valid, signature verified |
| page.published, page.updated, page.trashed, page.deleted | 3 each | 3/3 each | [43.9%, 100%] | |
| user.registered | 3 | 3/3 | [43.9%, 100%] | |
| comment.created, comment.approved | 3 each | 3/3 each | [43.9%, 100%] | |
| custom.example (generic hook) | 3 | 3/3 | [43.9%, 100%] | |
| webhook.test (admin test delivery) | 3 | 3/3 | [43.9%, 100%] | |
| Negative: creating a draft | 3 | 3/3 silent | [43.9%, 100%] | no delivery |
| Negative: spam comment | 3 | 3/3 silent | [43.9%, 100%] | no delivery |

13 of 13 documented events were covered; 0 unexpected deliveries in total. Only three repetitions were run, so the confidence interval for each event is wide.

### Other recorded checks (not experiments)

- `php -l`: 18 of 18 PHP files pass on PHP 7.4.33 and on PHP 8.5.10 (`results/php-lint.json`).
- Automated tests against real WordPress, 66 test cases each, 0 failures: PHP 8.3 with WordPress 7.1.3; PHP 7.4 with WordPress 7.1.3; PHP 7.4 with WordPress 6.0.17 (`results/test-run-*.json`). Unit and static tests without WordPress: 25 cases pass.
- Installable ZIP uploaded through wp-admin in a fresh WordPress, activated, exercised, deactivated and deleted, with uninstall removing tables, options and scheduled events: 5 of 5 cases pass.

## 8. Discussion

**H1 is partly supported and partly not, as stated.** The queue improves delivery over a single attempt in every tested condition (all six paired comparisons have Holm-adjusted p = 0.047 and Cliff's delta of at least 0.875; with 8 pairs this is the strongest result the design can give). But "every queued event is delivered at least once within its retry budget" is false whenever an event's attempts all fail: 3/320 events were lost at p=0.3 with a budget of 3, 15/320 at p=0.5 with a budget of 3, and 2/320 at p=0.5 with a budget of 6. These losses are not a defect of the retry loop; they are what the failure model predicts ((0.7 p)^B), and the dead-letter queue keeps the delivery for replay. The more useful finding is that "dead" is not the same as "lost": in 30 of the 50 dead-lettered deliveries in the B=3 and B=6 conditions at p >= 0.3 (4 of 7, 20 of 35 and 6 of 8 in the three conditions where any were dead-lettered) the receiver had applied the event but the final acknowledgement never arrived. A replay of such a delivery is safe only if the receiver de-duplicates by key, which is exactly what the replay design (same key, new delivery) assumes. The second half of H1 (one effect per event with de-duplication) held in every run: 0 of 1,920 queued events had more than one effect at a receiver that stores the keys, while 13 to 73 events per 320 did at a receiver that ignores them. This is expected from the design, but it shows that the key is stable across attempts, which is the property that makes de-duplication possible. The practical reading is: at-least-once delivery really does duplicate (up to 28 duplicate effects per 100 events here), and exactly-once effects require receiver-side de-duplication.

The naive sender never duplicated and never signed or logged anything; its cost was loss, at 0.7 p as modelled. Retrying raised traffic by 10% to 93% (requests per event 1.11 to 1.93), a cost that grows with the failure rate and the budget.

**H2 is supported in accelerated time.** All 2,160 recorded intervals had the specified nominal delay, stayed within the jitter bounds, and began no earlier than scheduled; the cap was reached and respected (960 intervals on the 600 s cap across both jitter settings); the jitter ratio had mean 1.0016 (bootstrap CI [0.9945, 1.0086]) and the Kolmogorov-Smirnov test did not reject uniformity (d = 0.0278 against a 5% critical value of 0.0506). Because the clock was moved rather than waited for, this establishes the schedule logic and the due-time selection, not real-world timing. The lateness figures (median 0, at most 3 s) are a property of the serial harness.

**H3 is supported.** All 11 trial types gave the expected outcome in 100 of 100 trials, including the inclusive 300 s tolerance boundary, a replay sent to a live receiver, and cross-language agreement (Node reproduced all 100 PHP-computed signatures). The unsigned baseline accepted all 900 attack trials by construction, so the comparison only illustrates what the verifier adds. The lower confidence bound of 96.3% is a reminder that 100 trials cannot demonstrate a rate near 100%. The verifier and the signer were written by the same author, which E3 cannot detect; the known-answer test vector, computed with `openssl`, covers the scheme but not the implementation of header parsing choices that both sides might share.

**H4 is supported.** All 13 documented events delivered exactly once with a schema-valid, correctly signed payload in 3 of 3 repetitions, and the two negative controls (a draft, a spam comment) stayed silent. Because every update of a published post also creates a revision, the exactly-once result also shows that revisions do not fire events. Three repetitions are few; the value of E4 is mainly regression protection.

**Surprises and process lessons.** (1) The first complete run of E1 was wrong because of a harness mistake: the loop that drives accelerated retries stopped after budget + 2 rounds, leaving up to 137 of 320 deliveries still waiting and counting them as lost (for example 27.5% lost at p=0.5, B=3 where the model predicts 4.3%). The mismatch with the model exposed the problem; the harness was fixed and every experiment was rerun (change log). (2) The harness receiver's signature check used the real clock while WordPress's clock was accelerated, which produced receiver-side verification failures in long retry chains (E1 section); this affected only a diagnostic counter. (3) The size cap on responses could not be demonstrated: in the Playground environment a 600 KB response with a 2 KB cap ended in a read timeout rather than a truncated read, so the cap is passed to the transport but its effect on bytes read is unverified.

**Relation to the background.** The plugin follows the practice documented by Stripe (timestamped HMAC over the raw body, a new signature per retry, de-duplication by identifier). Its symmetric jitter is the "equal jitter" style that Brooker's simulation found to do worse than full jitter on contention; this study did not test contention and does not claim an optimal schedule.

**Guidance for readers.** Treat webhooks from this plugin as at-least-once; store idempotency keys on the receiving side; check the signature over the raw bytes with a timestamp tolerance; and run the queue from a real cron job on quiet sites. Do not read the loss and duplicate rates above as expectations for production: they come from failure probabilities chosen by the author against a simulated receiver.

## 9. Threats to validity

- **Simulated receiver and failures.** The local receiver behaves more simply than real services (no TLS, no latency distribution, no rate limiting, no proxies, failures independent per attempt). The failure model was chosen by the author, so absolute loss and duplicate figures say nothing about real receivers; only the relationships between configurations are informative.
- **Accelerated time hides real timing effects.** E2 shows the schedule the plugin computes and that nothing is attempted early. It does not show real wall-clock waiting, WP-Cron spawn latency, behaviour on a site with no traffic, or clock skew between sender and receiver. The measured "lateness" is dominated by the serial test runner, not by the algorithm.
- **WP-Cron.** Its timing depends on site traffic; the harness drives the queue explicitly during experiments and reports this. Separate tests show that natural WP-Cron, a direct `wp-cron.php` request and the token route each ran the queue.
- **Test-clock guard.** The clock override exists in the product code behind a constant. It was shown inert without the constant in two WordPress instances, but it remains code that a reviewer must trust.
- **Environment.** One machine, shared and heavily loaded by other processes (CPU near 100% at times); at most two cores were used. Wall-clock durations are not reported as results. Contention can only cause late attempts (which E2 reports), real timeouts of the 1 s sender timeout on slow requests (which would show up as extra retries in E1), and slow runs.
- **Database layer.** Everything ran on SQLite through WordPress Playground. MySQL and MariaDB behaviour (for example row-claim semantics under concurrent runners) was not tested; concurrency was not stress-tested at all.
- **Statistical power.** Eight seeds per E1 condition give wide intervals for rare outcomes; E3 and E4 report proportions of 100 and 3 trials respectively, where 100% correct still has a Wilson lower bound below 100%.
- **Researcher degrees of freedom.** Analysis choices are fixed in this protocol; deviations are in the change log. The author wrote both the system and the verifier, so a shared misunderstanding of the scheme is possible; the known-answer HMAC vector was computed with `openssl`, independent of both.
- **Environment drift.** WordPress and Playground downloads are pinned only to "latest" for WordPress (7.1.3 at run time) and to an exact Playground CLI version.

## 10. Reproducibility checklist

- [ ] Code tagged at the commit used for the reported results (the folder is not a git repository here and nothing was committed or tagged)
- [x] Environment pinned: `package-lock.json` pins Node dependencies including `@wp-playground/cli` 3.1.60; WordPress 7.1.3 and PHP versions are recorded in `results/` (WordPress itself is fetched as "latest", so it is recorded, not pinned; no container image is used)
- [x] Random seeds and run counts recorded (in the experiment scripts and in each raw record)
- [x] Raw results published as JSONL alongside the summaries (no analysis notebook; summaries are produced by the scripts)
- [x] Hardware or service configuration described (`results/run-metadata.json`; shared loaded machine)
- [x] One command reproduces every table: `npm run experiments`, then `npm run report`
- [x] Data sources, licences and versions listed (no external data; dependencies in `package-lock.json`)

## 11. References

References were checked against their sources on the dates of this work; items marked partially verified are explained.

1. Stripe. "Receive Stripe events in your webhook endpoint" (webhook documentation: manual signature verification, `t=` and `v1=` scheme, five-minute tolerance, secret rolling, duplicate events). https://docs.stripe.com/webhooks. Verified by reading the page.
2. Standard Webhooks. https://www.standardwebhooks.com/. Partially verified: the landing page states the goal (protect against SSRF, spoofing and replay); the specific header names and algorithm were not confirmed from the page and are not relied on.
3. Krawczyk, H., Bellare, M. and Canetti, R. (1997). HMAC: Keyed-Hashing for Message Authentication. RFC 2104, IETF (Informational; updated by RFC 6151). https://www.rfc-editor.org/rfc/rfc2104. Verified (title, authors, date, status).
4. Brooker, M. (2015). "Exponential Backoff And Jitter." AWS Architecture Blog, 4 March 2015. https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/. Verified (title, author, date, conclusions as summarised above).
5. Jena, J., Dalal, S. and Tishkin, D. "The Idempotency-Key HTTP Header Field." Internet-Draft draft-ietf-httpapi-idempotency-key-header (version 08 when checked), work in progress. https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/. Verified as an active draft; it may have changed since.
6. OWASP. "Server-Side Request Forgery Prevention Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html. Verified by reading the page.
7. WordPress Developer Resources. "Cron" (Plugin Handbook). https://developer.wordpress.org/plugins/cron/. Verified by reading the page.

## 12. Change log

| Date (UTC) | Version | Change |
|---|---|---|
| (before the build) | v0.1 | Protocol drafted; no results yet. |
| 2026-10-23 to 2026-10-27 | v0.2 | Protocol finalised with the built system. Changes from v0.1, all made before the reported run: H1 clarified into two claims (section 2); "queue-only" and "full" defined as two receiver policies applied to the same plugin traffic (section 4.6); the failure model, run counts (E1 8 seeds x 40 events, E2 3 seeds x 40 deliveries, E3 100 deliveries, E4 3 repetitions) and tests (Wilcoxon exact, Holm, Cliff's delta, Wilson, KS restricted to nominal delays of 240 s or more) fixed in the experiment scripts; E2 uses a 600 s cap so the cap is exercised; the unsigned baseline is definitional. |
| 2026-10-23 | v0.2 | Development smoke runs (shrunken sizes, `WAWP_QUICK=1`, written outside `results/`) found one harness bug (a duplicated user name in the E4 matrix) and were discarded. Development also found and fixed one plugin bug before code freeze (template JSON was corrupted by an unnecessary unslash, found by the template test), tightened tests (response-size test lowered from 100 KB to 20 KB after it timed out under PHP 7.4 on the loaded machine; cache-header assertion relaxed to accept WordPress 6.0's own `no-cache` header after a development run on WordPress 6.0.17 had 1 failing case out of 66). Those development test runs are not the recorded ones. |
| 2026-10-24 | v0.2 | **Interrupted run (kept):** the first attempt at the full experiment set (`node experiments/run.mjs all`, started 2026-10-24T00:02Z) was cut off while running E1 (48 of 72 units recorded) when the working session ended. No result from it is used. Raw data: `results/earlier-partial-run-1/`. |
| 2026-10-26 | v0.2 | **Complete but flawed run (kept):** a full invocation (2026-10-26T17:36Z to 19:16Z) ran all four experiments. After reading its E1 summary the author found that the E1 harness stopped driving retries after (budget + 2) rounds, so many deliveries (up to 137 of 320 in one condition) were still waiting and were counted as lost; the observed loss (for example 27.5% at p=0.5, B=3) disagreed with the model (4.3%). This is a post-hoc harness fix prompted by the data, disclosed here; it changes the E1 measurement procedure, not the plugin. E2 to E4 results of that run looked like the final ones but are not reported. Raw data and summaries: `results/earlier-run-2-e1-round-limit-flaw/`. |
| 2026-10-27 | v0.2 | **Final run (reported):** `npm run experiments`, one uninterrupted invocation of all four experiments (`results/run-metadata.json`, invocation inv-1, 2026-10-27T21:09Z to 22:21Z, 72 + 6 + 1 + 3 units, started its own Playground server and stopped exactly that PID), from a clean install in a short path. Plugin code was frozen before this run and unchanged since the automated test runs recorded in `results/test-run-*.json`; the only change between run 2 and this run is the E1 round limit (budget x events + 10). Known diagnostic gap: E1's receiver-side signature check used the real clock under accelerated time (see section 7, E1) and rejection reasons were not recorded; not fixed or rerun. |

---

[README](../README.md) · [Client case study](case-study.md) · [Research report](research-report.md)
