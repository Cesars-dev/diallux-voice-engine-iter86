# plan_v5_iter86_warm_sync_call_fidelity — the fix plan (v2, restructured per owner 2026-09-29)

## Meta
- Date: 2026-09-29. Session status: SDK built + commit-history forensics DONE
  (`research/surgeon/iter86-warm-sync/01..04`). The racing-analysis phase is COMPLETE.
  This file = THE PROPOSED FIX, code-level, awaiting owner GO before any engine code changes.
- Working findings (proven, not theory):
  - **K1 (warm never scheduled):** state transitions executed by the ENGINE (phaseb chain in
    `_deterministic_node`, builder.py ~2666) never fire the prewarm — the two existing fire
    points (`tool_call_start` ~2377, post-execute `outcome.new_state` ~2544) both live in the
    model-speech path. Proof: 0 `warm:Closing` spans in b3; 6/6 `await_warm key=Closing =
    no_task`; the same hole exists in the PRE-fix1 baseline (`no_task Closing x7`).
  - **K2 (commit race):** the tool-detect warm can lose the destination round by ~0–200 ms
    (t30: warm fired 02:25:07.854, landed 09.136 after a 1282 ms prefill, destination round's
    request left 09.136–09.208, paid `cache_read=0`).
  - **K3 (await starved):** `await_warm` = shielded, correct, event-driven — found an
    in-flight warm ZERO times in 57 spans (47 already_done, 10 no_task). Awaiting alone
    cannot fix anything; the warm must be SCHEDULED.
  - **Step A (owner question "did fix1 compromise E2E?"):** NO. Best happy E2E in the
    ladder's fake-call history = the last commit (p50 1120 / p90 1395, user-perceived p50
    867). Barge p90 3076 beats both pre-fix1 baselines (3320 / 3661). The 2–4 s transition
    tails and multi-round turns pre-date fix1 (baseline had 12 multi-round turns, one with 4,
    uperm 3319–4571). Full table: `04_stepA_commit_history.md`.
  - **Fix1 eager-0.35 churn:** 45 fires → 32 turns → 17 billed-then-killed rounds (+5.4%
    input vs 0.70 baseline). Refires are REFUSED-then-cancelled (single-flight), not awaited.

## WORKING MODE (two-box law, unchanged)
- Session on FINLAND (MainVps). Repo ops typed on Finland, executed on OREGON via
  `ssh -o BatchMode=yes -i /home/julio/.ssh/id_ed25519_hetznor root@5.78.83.174 "<cmd>"`.
  Exception: hostname `ubuntu-4gb-hil-1` = you ARE on OREGON, run bare.
- Project root (OREGON): /home/julio/projects/clean_diallux_SDR. Finland tree = read-only
  plan mirror. Scratch: /tmp/opencode/.
- Env lane ports: fresh (8047/8048); never prod :8026/:8000-8003/:8090. systemd-run --user
  --collect; killed + port-verified at session end.

## THE FIX — T1: the warm-sync branch (build after GO)

Branch `engine/iter86-warm-sync` (already holds SDK d158b4c). FOUR changes, each ≤10 lines
of real logic:

### T1.1 — chain-path warm trigger (kills K1)
WHERE: `builder.py`, `_deterministic_node`, phaseb plan block (~2648–2705).
The plan is static and ends in `transition_to_Closing` — the destination is knowable the
moment the chain is entered. Two fires:
1. At chain start (right after the `patch["chain_done"] = True` latch):
   `warm_prompt_cache(<final state of plan>, force=True, dvs=current dvs)` — banks the whole
   webhook dead time (1.5–4.5 s) as head start; warm needs ~1.0–1.6 s.
2. After `record_booking_uid` / final dvs patch: re-fire `force=True` — supersedes via
   `_latest_warm`, byte-exact booking bytes so the destination round's prefix matches.
Implementation: `final_state = plan[-1][?]`… concretely the chain returns
`state_name="Closing"`; hard-wire `warm_prompt_cache("Closing", list(history)+list(new_history),
dvs=dict(dvs), force=True)` — the same payload shape the speech-path post-execute fire uses
(history+new_history = exact next-round history).
TESTS: (a) chain run emits 2 warm tasks for Closing (fire-count assert via `_warm_tasks`);
(b) fire #1 happens BEFORE the first webhook tool returns (fake executor with delays);
(c) `_latest_warm["Closing"].done()` true before the destination round's LLM call in a
fake-clock test.

### T1.2 — destination-round await at a NEW state (kills K2)
WHERE: `builder.py` state_node await block (~1838–1857) + `config.py`.
Add knob `warm_await_ms: int = 0` (config.py near `prewarm_entry_wait_eot_ms`, ~line 219).
Extend the await gate with one branch:
```python
elif runtime._prev_round_state != state_name and runtime.settings.warm_await_ms > 0:
    await runtime._await_warm(state_name, wait_ms=runtime.settings.warm_await_ms)
```
Semantics: the destination round (first round on a NEW state in this turn) releases the
await the INSTANT the in-flight warm completes (event-driven — this is the deterministic
moment; NOT a guessed time). Cap = safety only (a warm that died must not hang the call).
Outcomes: warm landed → 0 ms, cache hit. In flight → hold the round until it commits
(closes the t30 0–200 ms knife-edge). Never fired → ≤300 ms pay-cold-and-go (T1.1 removes
this case).
`_await_warm` already logs `await_warm key=X outcome=warm_waited waited_ms=N cap_ms=…` —
no new observability needed.
TESTS: (a) in-flight warm honored (fake clock) ≤ knob; (b) landed warm → 0 wait; (c)
knob 0 → byte-exact today's behavior; (d) `outcome=timeout` still proceeds (shield keeps
warm alive for the next round — existing contract).

### T1.3 — one-ahead successor warming (insurance, K1-generic)
WHERE: `builder.py` — at state ENTRY (or wherever the transition warm fires), also warm the
static FSM successors of the entered state. The edges already exist
(`transition_to_*` names in states; route tables). Latched per call (the `_prewarmed` set
already prevents duplicates; force=False).
Cost: ~1 extra warm per state entry (~1.0–1.6 s prefill each, 16-token cap, cheap).
Value: any transition that slips through the two explicit triggers still finds a standing
warm. Cover the map explicitly (ConfirmSlots→Closing, Booking→Closing, Closer→Offer,
contact_details→ConfirmSlots…).
TESTS: entering X fires warm(X) + warm(succ(X)) exactly once per call; no warm for
terminal Closing's (empty) successor set.

### T1.4 — ttft honesty on tool-bearing rounds (observability gap)
WHERE: `builder.py` ~2244 (`ttft_s` stamped only on `ev["type"]=="token"`).
Change: stamp the FIRST delta of ANY kind (first SPOKEN token for text rounds, first
tool-call delta for tool rounds). This closes the `ttft=None` gap that made ~17 rounds
invisible in the journal and lets the SDK distinguish killed vs cold without assumptions.
TESTS: fake stream with a tool_call event first stamps ttft; token-first stream unchanged.

### T1.5 — lane battery + gates (after T1.1–T1.4 green, same as iter85 battery law)
Lane 8047/8048, boot variant with the round-B knobs (EAGER_HOLD_AUDIO_UNTIL_CONFIRM=true,
0.35/0.65/3000, PAUSE_REASK_MS=8000 …) + `WARM_AWAIT_MS=300`.
GATES:
- ZERO destination rounds with `cache_read=0` on ANY transition (SDK matrix = ground truth).
- `await_warm` shows `outcome=warm_waited waited_ms<=300` ONLY where a warm was in flight.
- §11.1 holds: turn-1 e2e <1000 ms, eager_final_match 100%, zero held-path cancels,
  booking created → cancelled 200 (Cal.com event 3801235 REAL; cancel POST requires a
  cancellationReason body).
- Transition-turn user-perceived p50 improves vs b3's 2.2–4.6 s class.
- Suite: zero new reds (existing 7 stale reds acceptable, unchanged).
SDK runs on both new traces → `research/surgeon/iter86-warm-sync/05_iter86_battery.md`.

## THE FIX — T2: eager async release-by-ID (DESIGN DOC FIRST — needs owner GO as its own
iteration; turn-law: parallel speculative rounds change turn bookkeeping)

Current truth: single-flight (`_on_eager_eot` refuses while a turn runs; resume cancels;
refire). Owner's design: fire freely, keep them, release ONLY the one whose transcript
matches the confirmed EOT.

### T2.1 — the shape (code-level, for the design doc)
```python
# session.py
self._speculative: dict[str, asyncio.Task]   # keyed by normalised eager transcript
N_MAX = 2, evict oldest on overflow (cost cap; losers billed anyway)

_on_eager_eot(tr):        register+launch; NO refusal, NO cancel of older fires
_on_turn_resumed():       do NOT cancel; mark older speculative transcripts stale
                          (they keep running in background = billed, but silent)
_on_eot(final_tr):        winner = _speculative.get(_norm(final_tr))
                          winner in flight → adopt it (release ITS held audio)
                          else → queue normal turn (eager_final_match stays the guard)
                          cancel every remaining speculative at adopt
```
### T2.2 — the three shared structures that must NOT stay single-instance
1. TurnClock/turn_index per speculative (today: one `_clock`, swapped on adopt).
2. Gate hold per speculative (FIX-1's single `gate.hold()`/`_eager_held`/held buffer —
   releasing the winner must not leak the loser's held audio).
3. Graph-thread exclusivity: losers must never reach `executor.execute`/history writes
   (double-append = the iter38 disaster class). Losers are cut off at the first tool event.
### T2.3 — honest cost/latency framing (from the data)
- Latency win ≈ 0: today's refire starts a fresh LLM call within 80–160 ms and FIX-1's hold
  already hides that TTFT; the adopt path is latency-safe.
- What the churn costs TODAY: billing (+5.4% input), 45 RAG laneB embeds vs 32 needed,
  bookkeeping noise. Parallel-keep-alive would RAISE billed input, not lower it.
- Cheaper churn relief (candidate, small): laneB embed dedup across rapid refires of a
  growing transcript; absorb micro-resumes (<250 ms apart, small delta) as one fire.
Both are phase-1 candidates inside the SAME iteration if the owner opts for T2-lite.
### T2.4 — deliverable at GO: decision doc 03 series → owner verdict → build as next
iteration (engine branch per LAW 0).

## Env & dependencies (unchanged)
- OREGON venv: /home/julio/projects/clean_diallux_SDR/engine/.venv/bin/python (never bin/pip).
- wt-iter86 on OREGON (branch engine/iter86-warm-sync @ SDK commit d158b4c), .env copied.
- MODEL LAW: agent gpt-5.4; caller gpt-4.1-mini temp 0.3.
-VOICE LAW: lane tests keep EL Sarah. Ledger DB via venv sqlite3 (no CLI); micbridge trace
import must REPLACE live_sql.fetch_traces (name-prefix trap).
- Evidence: driver json, Langfuse micbridge-<sid> traces, lane journal; SDK as the parser.

## Validation plan (end-to-end)
Unit tests (4 new files-worth, fake clock / fake gate / fake stream) → targeted suite green
(26+ new asserts, zero new reds) → lane battery (8047 happy / 8048 bargeloop) under the
round-B knobs → SDK matrix on both traces → gate table → report → cancel every test booking
→ teardown lanes (`ss -ltnp` port-verified) → docs push + Finland mirror re-sync.
Rollback: every new behavior sits behind `WARM_AWAIT_MS` (default 0 = today byte-exact) and
additive warm fires (never block a round beyond the bounded await).

## Deferred / Out
- Driver pacing changes (frozen; Step A showed caller-pace is the fidelity gap — owner
  decides). FIX 8 soft filler after this session. Fragments t10/t34 → FIX 3 wave 2.
- Verbatim dupe checks re-run after T1.5 with real (fixed) ttft telemetry.
- Public analysis mirror repo = session artifact, not repo law (single-commit snapshot).

## Decisions awaiting owner
| # | Decision | Recommendation |
|---|---|---|
| D1 | GO for T1.1–T1.5 (warm-sync build + battery) | GO — all four changes are small and additive |
| D2 | T2 eager async release-by-ID: build as next iteration | Design doc first; latency gain ≈ 0 (hold already hides TTFT); decide for cost/churn motives |
| D3 | T1.3 one-ahead: include now or after battery | Include — cheap insurance, kills the whole cold class generically |
| D4 | PAUSE_REASK_MS 8000 as lane default | Confirm after T1.5 numbers |
