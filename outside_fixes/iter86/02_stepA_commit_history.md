# iter86 Step A — commit-to-commit E2E forensics on the FAKE harness calls (the owner's ask)

Sources: 9 FAKE (llm_mic_caller / b52 driver) battery traces across 4 commits, all metrics
from Langfuse turn reports via the SDK (cache columns from the b3 journal only; Langfuse
cache details are missing on pre-fix1 traces, so the baseline "caches" array there is NOT
ground truth).

| run (commit) | call | eager params | e2e p50 | e2e p90 | e2e max | uperm p50/p90 |
|---|---|---|---|---|---|---|
| fix7-sanity @4459f9f | happy-ish ref | eager off | 1178 | 1436 | 2621 | 1177/1401 |
| fix7-b1 @fa62162 | happy | off | 1200 | 1456 | 1613 | 1177/1456 |
| fix7-b2 @fa62162 | happy | off | 1200 | 1345 | 2086 | 1160/1324 |
| fix7-a2 @fa62162 | bargeloop | off | 1290 | 3661 | 4316 | 1246/3658 |
| fix5-happy @0693a9d | happy | 0.55/0.70 | 1412 | 1842 | 2405 | 1400/1831 |
| fix5-bargeloop @0693a9d | bargeloop | 0.55/0.70 | 1307 | 3320 | 4572 | 1293/3319 |
| fix1-happyA b97957f | happy | 0.55/600 | 1203 | 2144 | 3174 | 1166/2057 |
| **fix1-happyB b97957f** | happy | **0.35/0.65/3000+hold** | **1120** | **1395** | 1657 | **867/1065** |
| fix1-barge B3 b97957f | bargeloop | **0.35/0.65/3000+hold** | **1272** | **3076** | 4974 | 1192/2914 |

## Verdict 1 — "fix1 compromised E2E" — DISPROVEN in both directions

- Best happy-call E2E in the whole ladder = the LAST commit (fix1 0.35: p50 1120, p90 1395;
  user-perceived p50 867 — the "400/1100" numbers you remember are the plain-turn
  user-perceived on this run, the banked-head-start win).
- Best bargeloop E2E on this scenario since fix7 = also the LAST commit (p50 1272, p90 3076 —
  better than fix7-a2 1290/3661 and fix5 1307/3320 on identical driver/scenario).

## Verdict 2 — "every transition hangs / makes 3-4 calls" — TRUE, but pre-dates fix1

fix5-bargeloop (BEFORE fix1, eager 0.55, NO hold), multi-round turns and their
user-perceived: t11 3319, t15 **4571**, t18 2422, t24 3935, t26 4168, t33 4255 — eight 2-4s
turns, on NON-barged turns, one turn with FOUR rounds, and the identical `await no_task
Closing x7` hole. The 2-trip transition structure and the missing Closing warm are
OLD wiring, not an eager-0.35 regression. b3 (with eager-0.35) has 14 multi-round turns
vs baseline's 12 — round churn net-zero; the eager press just makes the S2L window honest.

## Verdict 3 — what eager-0.35 actually did at the margins

- t30 (contact→ConfirmSlots): destination prewarm FIRED at 02:25:07.854, completed 02:25:09.136
  (1282 ms prefill); the destination round's request left 02:25:09.136–09.208 — lost by ~0-200 ms.
  The knife-edge the owner predicted: a (bounded) await of ≤300 ms would have banked the cache hit.
- t38/t41 (→Closing): warm never scheduled (engine-chain path, hole predates fix1);
  chain idles 1.5-4.5 s in webhooks — the fix target.

## Step B (build list after owner GO, unchanged from 03 report)

(a) warm ON the engine-chain path (post-patch dvs, force=True) — kills the no_task hole;
(b) one-ahead successor warming at state entry; (c) WARM_AWAIT_MS bounded await (knob,
lane 300) — masters the t30 knife-edge; (d) ttft stamp on first tool-call delta.
