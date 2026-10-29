# EchoGate / Keel — Public Evidence Record

**LuxCrypta Technologies LLC** · [luxcrypta.ai](https://luxcrypta.ai) · [github.com/LuxCrypta](https://github.com/LuxCrypta)

This repository is a standalone, public record of **EchoGate**, LuxCrypta's deterministic admission-control (governance) layer for autonomous AI/algorithmic systems, and **Keel**, the live autonomous trading agent used to test it.

It exists to let outside engineers, researchers, and standards bodies read the architecture, the policy, and the production evidence directly — including the failures — rather than take a vendor's word for it.

**This repo contains no compiled binaries and no EchoGate source code.** It is documentation and evidence only. See [What this repo is / is not](#what-this-repo-is--is-not) below.

---

## Why this exists

Modern AI systems are increasingly autonomous, long-running, and multi-agent. The industry has strong reasoning engines and weak operational runtimes: it is comparatively easy to get an agent to *decide* something and comparatively hard to prove, after the fact, exactly what it was permitted to do, why, and whether anyone could reconstruct that decision later.

EchoGate is LuxCrypta's answer to the second half of that problem for one concrete, narrow case: **should this specific proposed action be admitted, right now, under this policy** — evaluated without any opinion on whether the action is a *good idea*. Keel is the live subject used to pressure-test that layer on a funded, real-money account, because a governance claim that has only ever been simulated is not yet a governance claim.

## What's in this repository

| File | Contents |
|---|---|
| [`KEEL_ECHOGATE_TECHNICAL_BACKGROUND.md`](./KEEL_ECHOGATE_TECHNICAL_BACKGROUND.md) | The full technical record: Keel's architecture, EchoGate's architecture and policy, the exact mechanism by which one governs the other, a graded ledger of what has been proven versus what has not, real production incidents (including defects and how they failed), and two appendices (pre-vetted claim language; key identifiers). |
| [`SAFE_RFC_SUBMISSION.md`](./SAFE_RFC_SUBMISSION.md) | A submission mapping two of the real incidents below onto the Open Secure AI Alliance's SAFE "From Lessons to Controls" template, for the public OSAA RFC process. |
| [`evidence/console-captures/`](./evidence/console-captures/) | Screenshots of the live Keel console and runtime logs: the dashboard, the proposal-vs-admission boundary, a sealed receipt chain, and a real sustained-refusal event in production. |

## Evidence grading

Every substantive claim in the technical background document carries a grade. This is load-bearing — read it before quoting anything from this repo:

| Grade | Meaning |
|---|---|
| `[P]` Proven | Observed directly in a production log, a sealed receipt, a console, or an executed test whose output was read |
| `[V]` Verified | Read from source code, policy JSON, or project documentation at a named path and line |
| `[U]` Unproven | Designed or asserted; never observed |
| `[X]` Discrepancy | Sources disagreed, or a prior claim was found wrong — the correction is recorded in place, not hidden |

The technical background document includes its own internal review that found and corrected an error in earlier published material (a fourth verdict, `DEFER`, that could not initially be substantiated against Keel's production logs). That correction is published here, in place, rather than quietly fixed and reposted. A governance claim that hides its own mistakes has disproved itself in the act of publishing.

## What this repo is / is not

**Is:**
- Architecture and design documentation for EchoGate (the admission-control layer) and Keel (the governed agent)
- The live policy cage's actual numeric rules (ceiling percentages, rule structure) — not hidden, because the claim is that the cage holds even when you can see it
- Real production evidence: real trade IDs, real account NAV figures, real log excerpts, real commit hashes, real test results — both the successes and the defects, including three defects that briefly affected live operation
- A graded, corrected, falsifiable record — claims that could not be substantiated are marked and explained, not deleted

**Is not:**
- EchoGate's source code or compiled binary. EchoGate itself is not open-sourced; this repo describes its contract, its policy format, and its observed behavior, never its implementation.
- A performance or returns claim. The live account is a small validation account; it proves plumbing, not profitability. See Appendix A of the technical background doc for the explicit "do not state" list this repo holds itself to.
- Audited or peer-reviewed in any formal sense. It is a vendor's own evidence record, graded and self-corrected, offered for independent scrutiny.

## Independent verification

Nothing here asks to be taken on faith. The technical background document names exact file paths, line numbers, commit hashes, trade IDs, and log excerpts so that specific claims can be checked against specific evidence rather than against the document's own summary of itself. Where a claim could not be verified during our own review, it is marked `[U]` or `[X]` rather than presented as settled.

## Contact

LuxCrypta Technologies LLC — [luxcrypta.ai/contact](https://luxcrypta.ai/contact/)
