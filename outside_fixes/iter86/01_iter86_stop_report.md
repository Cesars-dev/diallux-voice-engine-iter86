# iter86 — STOP report: SDK built, fidelity quantified, race found. Owner decisions now.

Commit: engine/iter86-warm-sync @ d158b4c (SDK only — scripts/call_forensics.py; ZERO
engine/diallux changes; nothing merged, nothing pushed). Research artifacts:
research/surgeon/iter86-warm-sync/00_* (SDK outputs), 01_warm_race_anatomy.md,
02_call_fidelity.md.

## What you asked for, and what it found

**1. Forensics SDK — BUILT and VERIFIED** (`scripts/call_forensics.py`, stdlib-only,
committed on the iter86 branch, dry-run only):
- `--trace <id-prefix>` — full per-turn matrix: round TTFT (journal join), tokens
  in/cache/out, eager-anchor S2L, user-perceived (confirmed-EOT → audio-out),
  barge/eager-match flags, eager/warm/await span counts, cache_read=0 cold rounds, warm/await
  event dumps.
- `--compare <traceA> <traceB>` — the fidelity table (turns/min, gaps, barge rate,
  user-perceived).
- `--run <ledger run_id>`, `--costs [--hours N]` round out the modes.
Verified: reproduces the iter85 manual forensics EXACTLY (REAL TTFT p50 695 / p90 829-880,
17 dead rounds ttft=None, transition user-perceived 2.2-4.6 s) — so the SDK is already the
single command replacing that whole manual pass.

**2. Does the fake caller answer like a human? — NO, quantified:**
- harness b3: 6.03 turns/min, inter-turn gap p50 ~8 s, barge on 94% of turns.
- real owner mic call (Sep-24 15:14, Langfuse diallux-call lane): 1.97 turns/min, gap
  p50 ~17 s / p90 ~40 s, barge 67%.
- REAL per-turn agent latency is IDENTICAL in both worlds (user-perceived p50 1234 ms in
  both traces) — agent-side speed is fine. The difference is caller PACE, 3x.
- Consequence: the eager-0.35 churn pathologies are worst-case harness artifacts (real
  calls fire eager 3x less often per minute). But the transition-warm cold round happens on
  EVERY real call that books — caller-independent.

**3. Where it actually happens (the riddle, solved):**

The transition destination round pays cache_read=0 NOT because the warm is slow — because:
- (K1) **Closing is entered through the phaseb engine chain** (deterministic_node →
  executor.execute → return state_name=Closing). That path never streams a tool_call_start
  and never returns through the post-execute warm site → warm:Closing fired ZERO times in
  the whole trace; all 6 await_warm key=Closing = no_task. The chain idled ~4 s in webhooks
  (create_booking ~1.5 s) — 4 s a Closing warm could have used, never scheduled.
- (K2) The tool-detect warm (fired at transition tool name) races the post-patch dvs
  patch: the post-tool context arrives at the same second, changes the tail bytes, and the
  destination round can still miss the cached prefix (t41's round B: cache 0 even with a
  5760-cached round 100 ms earlier).
- (K3) The existing bounded await could NEVER help in b3: 57 await spans, zero found an
  in-flight warm (47 already_done + 10 no_task). The await waits only for something SCHEDULED
  — Closing's was never scheduled.

So your instinct was exactly right and the numbers prove it: the race is not lost by
milliseconds — for the worst transitions it's lost by NOT ENTERING.

## Evaluation of your three ideas

1. "Delete eager EOT on the destination turn / give it plain 0.70 EOT" — the data says the
   eager-0.35 churn (3 billed rounds on t38, 17 dead rounds) is mostly HARNESS-paced
   (94% barge). On a real call it fires 3x less. It would NOT fix the cold cache (that's
   K1/K2, not EOT). It trades the eager head-start bank (p50 280 ms banked in round-B checks)
   for relief of a harness-only symptom. Not the lever for THIS bug. Cheap to A/B later on
   the lane if you want the ear check.
2. "A deterministic wait moment before the destination LLM call, ~5 ms poll, release when
   warm lands" — the await infrastructure EXISTS (await_warm, shielded, capped, spanned);
   b3 shows it never returns a wait today because there is always either a landed warm
   or none at all. So a wait there is FREE today and pays exactly when the warm is scheduled
   correctly but still in flight (the one-ahead case). The right shape: bounded await (the
   WARM_AWAIT_MS knob, 300 ms lane) at the destination entry, releasing the moment the warm
   lands, 0-wait fast path preserved. It is NOT a blind sleep — it's a task-completion
   trigger with a cap.
3. "Kill the wait entirely; just fire the warm at a point that guarantees it wins" — THIS
   is the load-bearing fix (K1): schedule the destination warm ON THE ENGINE-CHAIN PATH
   (when the chain's first tool executes / at phaseb:chain_ok, with the post-patch dvs),
   plus ONE-AHEAD warming of each state's FSM successors at normal entry. Evidence: the
   chain gives 1.5-4.5 s of guaranteed idle while the caller hears fillers; a full warm is
   ~1.0-1.6 s (from warm done ms in the journal). That wins with seconds to spare, no
   await needed in the happy case — and the bounded await becomes the safety net instead
   of the primary.

## My proposed fix (for your GO, not built)

**(a) Fix the scheduling hole in the chain path** — in `_deterministic_node`, where the
phaseb plan executes, fire `warm_prompt_cache(new_state, ..., force=True)` right after the
chain's dvs patches are applied (the same post-execute semantics the speech rounds already
get). This single change makes Closing (and any future chain-typed transition) warm exactly
like the other states — K1 deleted at the source.
**(b) One-ahead warming at state entry** — entering X warms X's static successors
(FSM edge map already exists in the routing tables); deduped, latched per call, additive
warm calls only.
**(c) Bounded await knob** — WARM_AWAIT_MS (default 0 env-knob, 300 on the lane) at the
destination round for the `eot` boundary; 0-wait fast path when the warm already landed.
This is your 5-ms-poll idea in its correct deterministic form.
**(d) ttft-on-tool-rounds logging** — stamp ttft on the first tool-call delta. Observability
honesty for the SDK.
Tests: one-ahead fires on entry + chain-fire on chain transitions + await honors cap/0-wait
+ ttft on tool rounds + knob-0 byte-exactness.

Then a battery (happy + barge-loop) on the lane with the SAME round-B params, gates:
ZERO cache_read=0 destination rounds, user-perceived transition turns p50 < 1.4 s, §11.1
holds (booking created → cancelled).

NOT touched: engine code, driver, prod, services. Worktree /tmp/opencode/wt-iter86 on OREGON
(native session on ubuntu-4gb-hil-1) holds ONLY the SDK commit. NO lanes booted, NO bookings.

— Stop point: owner verdict on (a)+(b)+(c)+(d) → next session builds.
