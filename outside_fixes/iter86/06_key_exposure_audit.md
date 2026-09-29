# iter86 / 2026-09-29 — KEY-EXPOSURE AUDIT (owner: "check if my OpenAI key was exposed — I want to rotate immediately")

Scope: GitHub account `Cesars-dev` (repo list pulled live), the private main repo
`Cesars-dev/clean-diallux-sdr` (full local history), the PUBLIC snapshot
`Cesars-dev/diallux-voice-engine-iter86`, and the legacy public repos.

## Verdict table

| key | where found | first git age | on GitHub | rotation? |
|---|---|---|---|---|
| OpenAI `sk-proj-yM4a-hJtl2O8sr708…` (7 files: retell/heating-uk perf tests + multiprompt) | PRIVATE clean-diallux-sdr, current origin/main tree + history | **2026-08-02** (commit 523dd51 "baseline: cal-slots + agent before hardening") | YES (private repo) | **ROTATE NOW** |
| `cal_live_73a8b37fb9…` (×87 refs — Booking.tools.json snapshots, LLM jsons, docs) | PRIVATE clean-diallux-sdr, tree + history (deep ages) | pre-Aug | YES (private) | **ROTATE** |
| `cal_live_e2d8752ebf…` (×9 — PROD llm.json, cal_slots docs/FINDINGS/SOP) | PRIVATE clean-diallux-sdr | since tech debt PT-09 (2026-08-19 noted) | YES (private) | **ROTATE** |
| `cal_live_be2771c023…` (Akrit, ×28) + `cal_live_a9704f08de…` (Akrit webhook, ×17) | PRIVATE clean-diallux-sdr (heating-uk configs + STATE.md) | PRE-Aug | YES (private) | **ROTATE** (+ ask Akrit) |
| `sk-lf-5ebb632e…` (Langfuse secret, engine/.env) | engine/.env only — **.env was NEVER git-tracked** (verified via `git check-ignore` + `git ls-files`); .env.example = placeholder | n/a | NO | optional |
| V7.9 PAT `ghp_5Q…` | never in THIS repo; burned in OLD `diallux-sdr-simulator` .git/config (MIGRATION_MAP) | n/a | repo is private; token was ALREADY flagged for rotation in PENDING_TASKS #1 | already flagged |

## What is NOT exposed
- **Public internet: NOT exposed.** GitHub code search across ALL public repos for
  `sk-proj-yM4a-hJtl2O8sr708` = **0 results** (rate limit hit before I could also
  double-check every cal-live prefix publicly — re-run these searches after the
  rate limit clears).
- Legacy PUBLIC repos cloned and grep-cleaned for the exact strings:
  `dialux-sdr-langgraph-v5` (v1.8-iter20 baseline) and `state_machine_voice` (iter21 fork)
  — both default branches: zero hits for the OpenAI key, zero `cal_live_…` full keys, zero
  high-entropy sk/ghp/glpat tokens.
- THE iter86 public snapshot (`diallux-voice-engine-iter86`) was scrubbed BEFORE push:
  all four cal_live full keys + the OpenAI key present in the private tree were replaced with
  `REDACTED` placeholders; `engine/.env` was never staged (gitignored, verified pre-commit).

## Caveat
- The private repo `clean-diallux-sdr` HISTORY still carries every key above (I redacted
  only the public snapshot's single commit). Exposure = repo collaborators + GitHub staff.
  Hard purge = BFG/history-rewrite, which per LAW 0 is an owner-ordered operation.

## Rotation checklist (owner)
1. OpenAI: platform.openai.com → API keys → revoke `sk-proj-…hxTNHd…`/the listed one → issue
   new key → update `engine/.env` (OPENAI_API_KEY) on Finland + OREGON worktrees.
2. Cal.com: API keys panel — rotate the 4 keys (personal 73a8b37f/inline e2d8752 + Akrit's
   two, 73a8b37f also the bookings tool default) → update `dialux_live` account in
   `services/cal_slots_endpoint/.env` (CAL_ACCOUNTS) + retell config build (regenerate from
   `build_llm.py`, NOT hand edits).
3. Optional: BFG purge of private repo history after rotation (owner order).
4. Re-run the rate-limited public code-searches post-rotation for peace of mind.
