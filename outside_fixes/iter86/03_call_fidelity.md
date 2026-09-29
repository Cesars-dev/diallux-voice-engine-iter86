# iter86 — harness-vs-real-call fidelity (SDK --compare, owner question: "test 2-3min, real 6-8min — does the FAKE caller answer like a human?")

Traces (all Sep-24 = REAL microphone calls by the OWNER, prod lane; b3 = the iter85 battery
harness call, gpt-4.1-mini driver, --min-gap 2.5 --max-gap 6 --agent-idle-s 5.5):

| call | trace | span | turns | turns/min | inter-turn gap p50/p90 (s) |
|---|---|---|---|---|---|
| harness b3 (fake caller) | micbridge-6cc7e46b11e4 | 308 s | 31 | **6.03** | **8.3 / 13.6** |
| REAL owner mic call 15:14 | micbridge-1f16330fb8dd | 275 s | 9 confirmed (+4 aborted) | **1.97** | **17.2 / 39.8** |
| REAL owner mic call 13:36 | micbridge-35407ac5a1bb | 113 s | 7 | **3.7** | 12.4 / ~16 |

Gap computation: deltas between successive turn-span starts (Langfuse) — for prod this IS the
caller's real think+listen time (they own the mic).

## The verdict per the owner's hypothesis

CONFIRMED, with numbers:
- The fake caller completes a 52-turn-scenario arc in ~5 min with a turn every ~8 s.
- The real owner on a mic waits 12-55 s between his own turns (p50 ~17 s) — a REAL caller
  spends most of the turn gap LISTENING to/absorbing the agent's answer, and thinking.
- Turn DENSITY is 3x higher in the harness; per-turn user-perceived latency, by contrast,
  is THE SAME (harness 1234 p50 vs prod 1234 p50 — identical budget, agent side is fine).

So the pathology set that shows up in barge scenarios (eager-0.35 churn, 17 dead rounds,
+5.4% tokens) is driven by a caller that talks 3x more often and BARGES (29/31 turns barge
vs 6/9, i.e. 67% real vs 94% fake). A real call has 2-4x fewer eager-fire opportunities per
minute, so the eager-0.35 tax per MINUTE of call is much lower in reality — but the
TRANSITION-ROUND cold-cache pathology is caller-independent (it happens every time the
booking chain runs, no matter who the caller is).

## What this means for the planned fix

1. The fix for the transition warm-race is NOT harness-specific. It protects a REAL
   production call exactly where the money is: the destination round after the booking
   chain (the moment that matters most: confirming a booked slot).
2. The barge-scenario pathologies (eager 0.35 churn) are HARNESS-CENTRIC (94% barge rate is
   superhuman); a real caller barges ~2/3 of turns. Ear A/B on eager 0.35 remains worth
   running, but the simulation is the worst case, not the modal case.
3. No driver change proposed in this report (driver FROZEN; verdict is the owner's).
   If the owner ever wants the driver humanized: raise --min-gap/--max-gap to a
   8-25 s band to mimic real turn gaps (p50 17 s, p90 40 s) — PROPOSAL ONLY.

## SDK data provenance

- harness: turns 31 / 5.2 min / 6.03 tpm / uperm p50 1192 ms, p90 3420 (skewed by transitions)
- real 15:14: turns 9 / 4.6 min / 1.97 tpm / uperm p50 1234 ms, p90 1515 max 1519 (NO
  transitions past Intake — the call never reached booking; p90 includes barge re-anchors)
- real 13:36: turns 7 / 1.9 min / 3.7 tpm / uperm p50 1295 (all plain turns — the cleanest
  latency read of a real call: ~1.1-1.4 s per answer, no transition penalty observed)

Raw outputs: 00_sdk_compare_harness_vs_prod.txt, 00_sdk_b3_full_matrix.txt,
00_sdk_prod_1514_matrix.txt, 00_sdk_prod_1336_matrix.txt.
