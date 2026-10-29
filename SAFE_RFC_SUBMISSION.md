# SAFE RFC Submission — Two Production Lessons from EchoGate / Keel

**Submitted by:** Thomas Coleman, LuxCrypta Technologies LLC ([luxcrypta.ai](https://luxcrypta.ai))
**For:** Open Secure AI Alliance (OSAA) — SAFE (Shared AI Findings Exchange), "From Lessons to Controls" RFC
**Full supporting record:** [`KEEL_ECHOGATE_TECHNICAL_BACKGROUND.md`](./KEEL_ECHOGATE_TECHNICAL_BACKGROUND.md) in this repository

These two entries are drawn directly from a real deployment: **Keel**, an autonomous multi-lane FX/crypto trading agent running unattended on a live funded broker account, governed at every decision point by **EchoGate**, a deterministic, offline, fail-closed admission-control binary. Every identifier below — commit hashes, trade IDs, NAV figures, log excerpts — is real and independently checkable against the full technical record linked above, which also carries an evidence grade (`[P]`/`[V]`/`[U]`/`[X]`) on every claim. Nothing here is simulated or reconstructed after the fact; both entries are graded `[P]` (Proven — observed directly in a production log, console, or sealed receipt) in that record.

---

## Entry 1 — A policy-violating action admitted by label, refused by rule

**Failure and affected systems.** An autonomous trading agent (Keel) was extended with a new strategy (`InvertedGridStrategy`, a momentum strategy — the mathematical inverse of the existing mean-reversion grid). The new strategy was wired into the live engine and submitted proposals to the governance layer (EchoGate) under the *existing* strategy's governance label (`grid_open_long` / `grid_open_short`), because the engine's governance-identity field and its execution-identity field were the same string (`engine.py:69`, `self.source = f"{strategy_kind}_strategy"`).

**Required defensive outcome.** A governance layer must refuse an action whose risk profile does not match the policy it is being evaluated against — specifically, it must not silently inherit permissions designed for a strategy with fundamentally different exit mechanics (broker-managed take-profit vs. self-managed stop-loss).

**Minimum control and alternatives considered.** The live policy's exit rule (`exit_required_min`: `payload.tp_pips >= 5`, `strict: true`) was a blunt instrument not designed for this purpose — it existed to bound reversion-strategy take-profits, not to police governance-label mismatches. An alternative considered and rejected was changing the agent's internal `strategy_kind` field to disambiguate the new strategy; this was rejected because `strategy_kind` is also load-bearing for unrelated FIFO broker-compliance sizing (`engine.py:1114`) and changing it would have introduced a second defect. The control actually in force was simpler and already present: **fail-closed, strict-mode numeric policy evaluation with no default-permit path.**

**Reproducible verification method.** With the new strategy armed on a flat book, every single proposal — both directions, both live instruments — was rejected by the existing exit rule, because the new strategy intentionally submits `tp_pips=0` (it manages its own exits) and `0 >= 5` is false under `strict: true`. Observed production log signature:
```
[REJECT] EUR_USD open_long  rung -1 @ 1.16972  L=0.10 E=1.00 A=0.60 S=1.00
[REJECT] GBP_USD open_short rung -1 @ 1.36491  L=0.10 E=1.00 A=0.60 S=1.00
```
Approximately 48 sustained REJECTs were logged before the mismatch was diagnosed. **Zero orders were transmitted to the broker. Zero capital moved.** The governance layer never received any information about momentum, reversion, or market direction — it evaluated permission against a strict numeric rule and failed closed, which is the entire point: the control caught a defect class (exit-architecture mismatch under label inheritance) that no amount of strategy-level code review had caught.

**Evidence to retain, owner, and deadline.** Retained: the rejection log excerpt above; the eventual fix (policy broadened by glob-pattern declaration only — `*grid_open_long` / `*grid_open_short` — with the four exit rules deliberately left literal so they continue to bind only the original strategy; committed as `ec2dcc2`); and the executed test suite confirming the fix (9 policy-level assertions, including a typo'd proposal type correctly rejected, and 125/125 full suite green). Owner: LuxCrypta Technologies engineering. Retention: indefinite, as part of the permanent governance audit chain (hash-chained receipts, each entry committing to the prior ledger head).

**Adoption, effectiveness, and review metrics.** Adoption: the fix was a one-line policy-pattern change plus two call-site arguments — the governance binary itself was never modified. Effectiveness: post-fix executed assertions confirmed the cage still bites under the new label (exposure and rung-count breaches are still refused) and that a fumbled label (typo in the proposal type) still fails closed rather than falling through to an ungoverned path. Review trigger: any future strategy addition that introduces a new exit architecture must be checked against this same class of label-inheritance mismatch before going live.

---

## Entry 2 — A recovery mechanism that regenerated the condition it was meant to clear

**Failure and affected systems.** The same agent (Keel) reconciles its internal position ledger against the broker's authoritative account state on every cycle, because the broker — not the agent's own ledger — is treated as the system of record. The reconciliation routine (`_adopt_broker_truth`) collapsed a grid lane's multiple distinct broker trades into a single netted ledger entry with an empty trade identifier (`trade_id=""`). That collapsed entry was invisible to the next cycle's *per-trade* reconciliation check (nothing to match against) but was fully counted in net exposure — so the real broker trades reappeared as unrecognized ("orphan") positions on the very next cycle and were re-adopted, indefinitely.

**Required defensive outcome.** A self-healing mechanism must not itself regenerate the fault condition it exists to clear. When a system enters a corrective loop, that loop must either converge or be detectable and bounded — it must not run undetected and unbounded against a live funded account.

**Minimum control and alternatives considered.** The minimum sufficient control was to preserve per-trade identity through the adoption path — i.e., adopt each broker trade individually with its real trade ID, rather than netting multiple trades into one unidentifiable ledger rung. A narrower alternative (increasing the reconciliation interval, or simply alerting on repeated adoption) was not sufficient on its own, because it would not have stopped new exposure from opening while the underlying position state remained wrong; the control actually required was architectural (preserve identity), not procedural (watch more closely).

**Reproducible verification method.** The defect was confirmed in production: one instrument pair (GBP/USD) was blocked from new opens for **76 continuous hours** — the agent's own drift-detection correctly refused new exposure throughout (`[DRIFT] 1 divergence(s) -> refusing new opens`), so no bad trade resulted, but the underlying loop ran roughly 1,167 adopt/reconcile iterations before the fix. After the fix (preserving `trade_id` through adoption for grid lanes), the same class of divergence was observed to resolve in 300–303 seconds and clear automatically, confirmed across four subsequent real events, including two full broker/network outages. Representative post-fix recovery, with real trade IDs and account NAV from the broker:
```
[FEED-GATE ] OANDA authorized; account NAV $99.80. Feed live. Arming.
[RECONCILE ] EUR_USD: adopted short 1u @ 1.15783 (trade 7667) -> ledger rung 1000
[RECONCILE ] GBP_USD: adopted short 1u @ 1.35119 (trade 7664) -> ledger rung 1000
[RECONCILE ] GBP_USD: adopted short 2u @ 1.34897 (trade 7631) -> ledger rung 1001
[ARMED     ] broker sync complete: realized baseline $-86.67, NAV $99.80, 3 open trade(s).
```

**Evidence to retain, owner, and deadline.** Retained: the fix commit (`272e0f5`); the triggering defect's root-cause comment, preserved verbatim in source (`engine.py:~667`: "sat blocked ~49h while the resolver regenerated the drift it was clearing"); the four independent post-fix recovery events (including the one excerpted above, tied to a separate earlier infrastructure-timeout fix at commit `9104eb5`); and the full hash-chained audit record of every drift episode, before and after. Owner: LuxCrypta Technologies engineering. Retention: indefinite, as part of the permanent audit chain.

**Adoption, effectiveness, and review metrics.** Adoption: immediate, applied to all grid-type lanes. Effectiveness metric actually used: drift-episode duration. Before the fix, the defect manifested as unbounded (76 hours, not self-resolving). After the fix, the same divergence class consistently resolves in roughly five minutes across all observed recurrences. Review trigger: any future change to the reconciliation/adoption path must preserve per-trade identity end-to-end, and should be checked against this specific regression before merge.

---

*Both entries above are reproduced in full detail, with additional architecture and policy context, in [`KEEL_ECHOGATE_TECHNICAL_BACKGROUND.md`](./KEEL_ECHOGATE_TECHNICAL_BACKGROUND.md) §27–28 and §32 of this repository, under the evidence-grading discipline described there.*

— Thomas Coleman, LuxCrypta Technologies LLC
