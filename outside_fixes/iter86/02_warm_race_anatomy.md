# iter86 — the warm-race anatomy of a transition turn (data-scientist pass)

Data: b3 trace micbridge-6cc7e46b11e4 (Langfuse 8ac96fc4db19c5016bb3271b5632ba0d),
journal window 2026-09-29 02:21-02:27 (50 round-usage lines, 31 confirmed turns,
45 eager fires), SDK per-turn matrix in 00_sdk_b3_full_matrix.txt.
SDK commit: d158b4c (branch engine/iter86-warm-sync, no engine code touched).

## 1. The clock of ONE transition turn (turn 38 — ConfirmSlots to Closing, the b3 template)

All times UTC, same single call:

```
02:25:50.7  eager fire          turn 38 starts SPECULATIVELY (eager 0.35, mid-speech)
02:25:50.7  await_warm ConfirmSlots -> already_done 0ms      (warm prepared at t30, long dead)
02:25:51.7  round 1  ConfirmSlots 6784-cached, ttft=None    <- spawns the transition decision
02:25:52.0  round 1b ConfirmSlots 2688 (post-tool ctx), ttft=None  <- speaks + emits transition tools
02:25:53.5  tool chain runs     (validate -> verify -> create_booking... 5 tools, ~1.5s wall)
02:25:55.0  phaseb:chain_ok     transition_to_Closing ok
02:25:55-56 round 2 prep window   ~1.2s of remaining speak + drain -- the warm SHOULD fill this
02:25:56.4  round 2  Closing    input=6598  cache_read=0   ttft=773ms   <- COLD
02:25:58.6  eager fires AGAIN  await_warm key=Closing outcome=no_task  <- warm for Closing NEVER FIRED
02:25:59.9  round 41 Closing   cache_read=5760 (OpenAI's own accumulation now visible)
02:26:02.1  round 41b Closing  cache_read=0 AGAIN (post-patch dvs changed the tail anyway)
02:26:08..  turn 42: closing speech rides 4736 (warm finally exists via t41's own requests)
```

Same signature on the other two cold rounds: t30 -> ConfirmSlots (cache 0, ttft 579),
t41 -> Closing (cache 0, ttft 710).

## 2. Where the seconds go (user-perceived = S2L - HS + L2T + T2A, from the SDK matrix)

- Plain turns: user-perceived p50 ~1.0-1.2 s = real LLM TTFT (~695) + L2T (~300, TTS TTFB).
- Transition turns (t30/t38/t41 destination round): user-perceived 2.2-4.6 s. The cost:
  (a) two trips: round A speaks + fires tools (120-185 output tokens ~= 1.5-2.5 s of
  serialization) THEN tools run (~1.5 s in the phaseb chain) THEN round B pays its own TTFT
  on a COLD prompt (cache_read=0);
  (b) eager-anchor contamination inflates S2L (0.35 fires mid-speech; caller keeps talking;
  resumed rounds restart the round while the turn metric stretches).

The 2-trip structure is BY DESIGN (round A ack+tools / round B next question on the new
state's prompt). The ONLY abnormal part is cache_read=0 on round B -- our own warm should
have made that ~0.

## 3. The smoking gun: Closing is entered via the ENGINE chain, and the engine chain path never fires the warm

Warm fires exist in two places in builder.py:

1. `tool_call_start` in the STREAM (iter46 T4a, builder.py ~2377): fires the moment the
   transition tool NAME completes in the model's stream. This covered Discovery, Closer,
   Offer, contact_details, ConfirmSlots -- every state whose transition was decided BY THE
   MODEL inside a normal LLM speech round.
2. post-execute (iter43 T3 / iter44 T1, builder.py ~2544): `outcome.new_state` fires the
   post-patch force=True warm -- same speech-round path above.

Closing is NOT reached by either: its transition_to_Closing runs inside the **phaseb engine
chain** (builder.py ~2666): deterministic_node -> executor.execute(...) -> return
{state_name: Closing}, all inside `_deterministic_node`, which NEVER streams a
tool_call_start event and never returns through the stream handler that fires the post-execute
warm. Result: zero warm:Closing spans in the entire trace; every
`await_warm key=Closing` (6 of them) = no_task.

The chain ITSELF idled ~4 s (validate at 02:25:52.9 -> transition_to_Closing at 02:25:55.0,
create_livecall_booking webhook ~1.5 s) while the caller heard fillers -- 4 s that a Closing
warm could have used. Nobody scheduled it.

And the t30 ConfirmSlots cold round: the contact_details -> ConfirmSlots transition ALSO went
through a phaseb-style path where the warm fired at tool_call_start of
transition_to_ConfirmSlots... but post-patch dvs (selected_time etc.) changed the tail bytes
so the warm (fired pre-patch) cached a PREFIX SHAPE that did not match the round B request --
cache_read=0级的 destination cache hit requires the post-PATCH bytes (iter46 T4a fire got
superseded by the post-execute fire for model-driven transitions; the chain path has no
equivalent).

## 4. The eager-0.35 tax on ONE transition turn

The t38 window shows fire -> adopt/discarded -> fire -> adopt -> adopt -> THREE billed rounds
in 10 s (8558 + 8674 + 6598 input tokens; two had ttft=None -- killed mid-flight but billed).
Eager fires ~80 ms into speech growth; the caller of the barge scenario talks 5-12 s per
turn, so each turn restarts 1-2 rounds that still bill OpenAI input tokens (+5.4% vs the
0.70 baseline bargeloop: 312k vs 296k).

## 5. Verdict + the fix levers (owner-ordered: NO build this session)

The warm does not lose the race because it is slow -- it loses because for the worst
transitions (Closing via the phaseb chain) it is NEVER SCHEDULED, and where it IS scheduled
(tool-detect path), the post-tool dvs patch can change the tail bytes so the entry round
misses the cache anyway. The existing bounded await (await_warm, cap 500/100 ms) NEVER once
found an in-flight warm in the whole b3 trace: 57 await_warm spans = 47 already_done +
10 no_task, 0 landings, 0 timeouts.

Fix levers (evaluated; nothing built):
- (a) Deterministic ONE-AHEAD warm (warm X and X's FSM successors on entering X). Solves the
  NEXT transition's cold round in theory -- but the hook must live ON THE CHAIN PATH (after
  transition_to_X.exe returns inside deterministic_node / at phaseb:chain_ok), not only in
  the stream handler; otherwise Closing's cold round happens again.
  AND one-ahead alone does NOT fix t38-style predictability: the phaseb chain run's own
  webhook latency IS the head start (usually >1 s), so a warm scheduled at chain start
  (first chain tool execute) with the post-patch dvs of that pass would land well before
  the destination round.
- (b) Bounded await at destination entry (WARM_AWAIT_MS, default 0): the await point EXISTS
  (await_warm at state entry; prewarm_entry_wait_eot_ms=500 already active in lane). The
  b3 data says: this await would WAIT 4/57 times today (only the no_task cases would go
  from 0 wait to ... 0 effective -- no_task means the warm was never fired, so waiting CANNOT
  help). Await only helps where the warm was fired but is still in flight -- which NEVER
  happened in b3. So (b) alone changes nothing today; it only pays once (a) guarantees the
  warm exists. Sharp conclusion: (a) is load-bearing; (b) is insurance.
- (c) ttft-on-tool-rounds logging: observability honesty, cheap, keep in plan.

Also CONFIRMED by the second cold case (t41b cache 0 even though t41 had a 5760-cached
round 100 ms earlier): the post-record_booking_outcome dvs patch CHANGED the tail bytes ->
partial cache lost. Any real fix must ALSO schedule the warm AFTER the post-patch bytes are
final (the current post-execute fire does this for speech rounds; the chain needs its own).

## Appendix: SDK usage

```
python scripts/call_forensics.py --trace <id|prefix> [--journal <journalctl.txt>]
python scripts/call_forensics.py --trace A --trace-join ...       # per-turn matrix
python scripts/call_forensics.py --compare <traceA> <traceB>     # fidelity table
python scripts/call_forensics.py --run <ledger_run_id>
python scripts/call_forensics.py --costs [--hours 24] [--name micbridge]
```
Modes verified against the manual forensics: b3 REAL round TTFT p50 695 / p90 829-880,
17 dead rounds TTFT=None, user-perceived transition turns 2.2-4.6 s, warm race never lost
by a hair -- never even entered for Closing.
