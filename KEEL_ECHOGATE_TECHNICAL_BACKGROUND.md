# KEEL & ECHOGATE — COMPREHENSIVE TECHNICAL BACKGROUND

**LuxCrypta Technologies LLC** · Thomas B. Coleman, Founder & CEO
**Compiled:** 29 October 2026
**Subject:** architecture of the Keel autonomous trading agent, architecture of the EchoGate admission-control layer, the exact mechanism by which the second governs the first, and a graded ledger of what has been demonstrated versus what has not.

---

## 0. EVIDENCE GRADING — READ THIS FIRST

Every substantive claim below carries a grade. Do not promote a claim across grades when drafting from this document.

| Grade | Meaning                                                                                                                    |
| ----- | -------------------------------------------------------------------------------------------------------------------------- |
| `[P]` | **Proven** — observed directly in a production log, a sealed receipt, a console, or an executed test whose output was read |
| `[V]` | **Verified** — read from source code, policy JSON, or project documentation at a named path and line                       |
| `[U]` | **Unproven** — designed, asserted, or pending measurement; never observed                                                  |
| `[X]` | **Discrepancy** — sources disagree, or a prior claim was found wrong; resolve before external publication                  |

The discipline this document records is itself the product. Where something failed, the failure is stated. Where a prior claim was wrong, the correction is stated. A governance company that conceals its own gaps has disproved its thesis in the act of publishing.

---

# PART I — KEEL

## 1. What Keel is

`[V]` Keel is an autonomous, multi-lane, multi-venue algorithmic trading agent written in Python, running continuously and unattended against a live funded broker account. It is the _governed subject_ in the LuxCrypta stack — the component that holds strategy, forms intent, and would place orders if permitted. It is explicitly **not** the governance layer.

`[V]` Keel's design premise, inherited from LuxCrypta doctrine: an autonomous system's failure modes in production are rarely failures of judgment. They are lost state after a restart, an unreconciled position, an action taken while blind to real exposure, or a decision that cannot be reconstructed afterward. Keel is therefore built as a _continuity_ artifact first and a trading artifact second. The validation account is intentionally small (NAV ≈ $99.74 at last read); this deployment proves plumbing, not returns.

`[V]` **No LLM exists anywhere in Keel's decision path.** Every term in every strategy is a pure function of trailing price bars. This is a deliberate constraint of the "lawful, inspectable, exact" doctrine and is what makes byte-for-byte replay possible.

## 2. Repository topology

`[V]` Four independent git repositories on the production host (`LuxCryptas-MacBook-Pro`, macOS, Python 3.10.18 under `.venv`):

| Path                    | Role                                                      | Console port |
| ----------------------- | ---------------------------------------------------------- | ------------ |
| `/Users/lct/keel`       | The agent + EchoGate binary + its own console             | 8765         |
| `/Users/lct/neptune`    | Strategy referee — sealed assays, deflated Sharpe         | 8766         |
| `/Users/lct/oyster`     | Strategy proposer (evolutionary search) + Reef map server | 8767 / 8769  |
| `/Users/lct/luxconsole` | Unified console federating all four engines               | 8768         |

`[V]` Inside `/Users/lct/keel`, the agent package `keelbot/` totals **5,091 lines**:

| Module                            | Lines   | Function                                                                            |
| --------------------------------- | ------- | ------------------------------------------------------------------------------------ |
| `engine.py`                       | 1798    | `Lane`, `KeelEngine` — tick loop, intent handling, reconciliation driver, lifecycle |
| `gate.py`                         | 440     | `EchoGateGate` — the Python↔binary invocation wrapper                               |
| `core.py`                         | 374     | `Config`, `Intent`, `Routing`, `Ledger`, `GridStrategy`, `RiskClassifier`, `Regime` |
| `coinbase_io.py`                  | 347     | Coinbase venue adapter                                                              |
| `io_adapters.py`                  | 298     | `OandaV20Execution`, `OandaV20MultiFeed`                                            |
| `reconcile.py`                    | 265     | Broker-truth divergence classification                                              |
| `kraken_io.py`                    | 253     | Kraken venue adapter                                                                |
| `kraken_sim_adapter.py`           | 180     | Kraken simulation execution/feed                                                    |
| `grid_exit.py`                    | 176     | Momentum exit gate                                                                  |
| `grid_sizing.py`                  | 169     | ATR + inventory dynamic sizer                                                       |
| `breakout.py`                     | 168     | `BreakoutStrategy` — Donchian channel trend-follower                                |
| `adaptive.py`                     | 128     | Mode selector (conservative/normal/aggressive)                                      |
| `meanrev.py`                      | 119     | `MeanRevStrategy` — vol-scaled fade with turn confirmation                          |
| `invgrid.py`                      | 99      | `InvertedGridStrategy` — momentum grid                                              |
| `brakedgrid.py`                   | 97      | `BrakedGridStrategy` — grid wrapper                                                 |
| `multi_venue_feed.py`             | 80      | Venue feed multiplexer                                                              |
| `sim_fx_io.py` / `sim_perp_io.py` | 51 / 49 | Simulation adapters                                                                 |

`[V]` Supporting paths: `run_oanda.py` (entry point), `policies/policy.json` (1,043 lines — the live cage), `config/keel_pairs.json`, `config/keel_modes.json`, `config/keel_v0.json`, `keel_watchdog.sh` (supervisor), `echogate/build/echogate` (the governance binary), `runtime/` (status, logs, receipts, audit chain, control channel), `tests/` (125 tests).

`[V]` **`run_oanda.py` is an entry point, not a library.** Importing it executes the module body, which builds lanes and starts the engine. Importing it once started a second live engine against the funded account. This is recorded as a standing operational rule.

## 3. The Lane abstraction

`[V]` A **lane** is the unit of governed activity: the triple of _(strategy, instrument, venue)_. Constructed at `engine.py:42`:

```python
def __init__(self, cfg, gate, execution, strategy=None,
             strategy_kind="grid", venue="oanda"):
```

Each lane owns its own `Config`, its own `EchoGateGate` instance, its own execution adapter, its own `Ledger`, and its own `Regime` classifier. Lanes do not share state; the portfolio-level view is assembled separately via `_portfolio_snapshot()` (`engine.py:1204`) and passed into each gate evaluation, which is how cross-lane ceilings become enforceable without cross-lane coupling.

`[V]` `engine.py:69` derives the governance identity string:

```python
self.source = f"{strategy_kind}_strategy"
```

This single line is load-bearing. `strategy_kind` determines (a) the `source` field submitted to EchoGate, (b) which policy rulebook applies, and — as discovered in the inverted-grid work — (c) several internal behaviors including FIFO sizing. The conflation of governance identity with execution identity in one field is the root of the most instructive bug in this record (§32).

`[V]` Live lane configuration (`config/keel_pairs.json`):

```json
[
  {
    "instrument": "EUR_USD",
    "strategy": "grid",
    "rung_spacing_pips": 20.0,
    "take_profit_pips": 18.0,
    "max_rungs": 3
  },
  {
    "instrument": "GBP_USD",
    "strategy": "grid",
    "rung_spacing_pips": 20.0,
    "take_profit_pips": 18.0,
    "max_rungs": 2
  },
  {
    "instrument": "USD_JPY",
    "strategy": "grid",
    "active": false,
    "pip_size": 0.01
  }
]
```

`[V]` The asymmetric `max_rungs` (EUR 3, GBP 2) is not arbitrary. One ceiling — 12% strategy-pair gross — meets two prices: GBP/USD trades roughly 16.5% higher than EUR/USD, so a single rung count could not fit both without one lane either wasting headroom or breaching. USD/JPY is stood down pre-live for wide spread, BoJ trend risk, and the highest quarantine rate observed in soak testing; its code and config are preserved intact.

## 4. The Config surface

`[V]` `Config` is a dataclass in `core.py`. The full governed surface:

**Grid geometry** — `instrument`, `direction`, `pip_size` (0.0001), `rung_spacing_pips` (10.0), `rung_size_units` (1000), `max_rungs` (10), `take_profit_pips` (10.0), `rung_fraction` (0.05)

**Exposure envelope** — `max_exposure_fraction` (0.50), `max_net_exposure_fraction` (0.50), `drawdown_floor_fraction` (−0.02), `max_gross_units` (10000), `drawdown_floor_usd` (−50.0), `leverage` (1)

**Regime detection** — `regime_trend_threshold_spacings` (4.0), `regime_sma_periods` (50), `trend_filter_enabled` (True), `trend_filter_er_threshold` (0.45), `trend_filter_er_periods` (20), `trend_filter_ma_periods` (50)

**Quarantine** — `quarantine_ttl_seconds` (10), `quarantine_sweep_seconds` (5.0)

**Crypto options** — `long_only` (spot cannot short → no sell-to-open rungs), `maker` (post-only limit rungs), `spacing_pct`, `take_profit_pct` (percent-of-price geometry instead of pip-based)

**Dynamic sizing** — `sizing_mode` ("fixed" | "atr_dynamic"), `atr_period` (14), `base_risk_pct`, `vol_scalar_min` (0.5), `vol_scalar_max` (3.0), `inv_scalar_min` (0.3), `min_units` (1000)

**FIFO** — `fifo_unique_rung_units` (True)

**Momentum exit gate** — `exit_gate_enabled`, `exit_fast_ema` (8), `exit_slow_ema` (21), `exit_confirm_bars` (2), `exit_cooldown_bars` (10), `exit_min_adverse_pips` (15.0)

`[V]` Two derived properties resolve pip-space to price-space (`core.py:63`, `core.py:67`):

```python
@property
def spacing(self)      -> float: return self.rung_spacing_pips * self.pip_size
@property
def take_profit(self)  -> float: return self.take_profit_pips  * self.pip_size
```

`[V]` Modes (`config/keel_modes.json`) are named presets applied to every lane — `conservative` / `normal` / `aggressive` / `live_small` / `sim_aggressive` / `adaptive_sim`. The live account runs `live_small`: spacing 20p, TP 18p, max_rungs 3, rung_fraction 0.0075, exposure fractions 0.25, `fifo_unique_rung_units: true`. The file's own comment states the governing principle verbatim: _"The EchoGate cage (gross/net/pair ceilings) bounds everything regardless — governance is not the lever, appetite is."_

## 5. The strategy layer

`[V]` Every strategy implements the same four-method interface — `start(price)`, `on_tick(price) -> [Intent]`, `mark_open(rung, price)`, `mark_closed(rung)` — and is deterministic. Strategies are interchangeable without touching the engine or the governor.

**`GridStrategy` (core.py:~185) — mean reversion.** Seats a center at first price; rung _k_ sits at `center − k·spacing`. Positive _k_ (below center) opens **long** as price _falls_ into the level; negative _k_ (above center) opens **short** as price _rises_ into it. Exits at a fixed take-profit in the reversion direction, carried as a resting **broker-side** take-profit order. Recomputes `_spacing` / `_tp` in `start()` (core.py:201–202) so percent-based geometry resolves against live price. **No stop** — reversion is the thesis; there is nothing to stop out of.

**`InvertedGridStrategy` (invgrid.py) — momentum.** The exact mirror. Same levels; opens **long** when price breaks _up_ through a level, **short** when it breaks _down_. Exits take-profit in the trend direction, and — necessarily — carries a **stop** at `stop_mult × TP` against the position, default `stop_mult = 1.5`. Its own docstring states the honest thesis: _"neither is an edge on its own — they are mirror images, each right exactly when the other is wrong. The real edge, if any, is REGIME-SWITCHING between them."_ Exits are **self-managed**: it emits `close_long`/`close_short` itself rather than resting an order at the broker.

**`BreakoutStrategy` (breakout.py) — Donchian trend.** Entry on channel break (`entry_period`, default 50), exit on reverse-channel break (`exit_period`, default 20), with an optional hard take-profit percentage and a post-trade cooldown. Runs on resampled bars (`bar_seconds`), decoupled from the feed poll rate. **Net-managed**: it drives to a target net position via a single delta order rather than per-rung opens and closes, which is the correct model on a netting account.

**`MeanRevStrategy` (meanrev.py).** Fades a stretched move only after it ticks back — `price < ref − entry_k·vol and turning_up` for a long — explicitly declining to catch a falling knife.

**`BrakedGridStrategy` (brakedgrid.py).** Wraps `GridStrategy` with additional braking logic.

`[V]` Strategy selection happens in `make_strategy(cfg, pc)` in `run_oanda.py:~126`, keyed off `pc.get("strategy", "grid")`. Returning `None` causes the `Lane` to construct a `GridStrategy` itself.

## 6. The tick loop and the intent pipeline

`[V]` Feed: `OandaV20MultiFeed` polls `/v3/accounts/{id}/pricing` for all instruments at `poll_seconds=2.0`. Each price tick drives `strategy.on_tick(price)`, which returns zero or more `Intent` objects.

`[V]` `Intent` is a frozen dataclass of four fields plus `tp_pips`: `kind`, `rung`, `price`, `units`. `kind` ∈ `{open_long, open_short, close_long, close_short}`. **An Intent is not an order.** It becomes an order only on an admitting verdict.

`[V]` Each intent flows through `_handle_intent` (`engine.py:1363`). The ordered gauntlet:

| #   | Stage                      | Line | Behavior                                                                |
| --- | --------------------------- | ---- | -------------------------------------------------------------------------- |
| 1   | `_normalize_intent`        | 1080 | Resolve units (fixed fraction or ATR sizer), attach `tp_pips`           |
| 2   | Trend filter               | 1369 | Grid opens only: refuse to stack inventory into a trend                 |
| 3   | Engine-state gate          | 1375 | No opens if halted, draining/killed, unarmed, drifting, or equity stale |
| 4   | Broker-TP close swallow    | 1380 | A close is dropped if the lane rides broker take-profits                |
| 5   | Anti-windup backoff        | 1383 | An unchanged already-rejected grid open is not re-gated                 |
| 6   | `risk.classify`            | 1386 | Build the `Routing` — proposal type + context findings                  |
| 7   | `gate.evaluate`            | 1388 | **Hand to EchoGate**                                                    |
| 8   | `_execute_admitted_intent` | 1415 | Only on `ADMIT`                                                         |

`[V]` Stages 2–5 are _agent-side_ refusals — Keel declining to even propose. Stage 7 is the _governance_ refusal. These are architecturally distinct: the first set is strategy discipline the agent could in principle revise; the second is a constraint the agent cannot reach. The comment at `engine.py:1377` captures the asymmetry exactly: **"Closes always pass."** The engine never blocks an exit, because blocking an exit is the one failure that can trap capital.

`[V]` `_normalize_intent` carries the sizing branch (`engine.py:1086–1120`). For opens it computes units either by `rung_fraction × equity` or by the ATR+inventory sizer (`compute_rung_units(equity_usd, tp_pips, open_long_rungs, open_short_rungs)`), then attaches the take-profit:

```python
tp = 0.0 if getattr(lane, "self_managed_exits", False) \
     else (cfg.take_profit_pips if lane.strategy_kind == "grid" else 0.0)
```

For closes it replaces units with the actual open position size and zeroes `tp_pips` (`engine.py:1122–1125`).

## 7. Execution and the venue abstraction

`[V]` `OandaV20Execution` (`io_adapters.py:133`) carries the class attribute `broker_take_profit = True`. Orders are `MARKET` / `timeInForce: FOK` / `positionFill: DEFAULT`. When `tp_pips` is non-zero the order body gains `takeProfitOnFill` at `price ± tp_pips·pip_size` rounded to 5dp.

`[V]` Every order carries `clientExtensions`:

```python
{"tag": f"{strategy_kind}.{instrument}",
 "id":  f"keel-{instrument}-r{rung}-t{ticks}"}
```

The tag maps broker trades back to lanes during reconciliation; the client id provides restart-race deduplication.

`[V]` Fills are **verified, never assumed**. `market_order` returns `{"filled": False, "reason": "no_fill", ...}` when `orderFillTransaction` is absent, and `{"filled": False, "error": str(e)}` on exception. Nothing is written to the ledger on an unconfirmed fill.

`[V]` Read paths: `account_summary()` returns `nav`, `balance`, `realized_pl`, `unrealized_pl`, `margin_used`, `open_trade_count` in one account-level call covering every lane. `open_trades()` returns signed units per instrument with `trade_id` and `take_profit_order_id` preserved. `closed_trades(trade_ids)` resolves each trade to a close price and realized P&L, tolerating `NO_SUCH_TRADE` as `"reason": "missing"` rather than raising.

`[V]` Two execution models coexist. Per-rung lanes (grid, invgrid) open and close individual rungs. Net-managed lanes (breakout) compute `delta = target_net − broker_net` and place one order, documented at `engine.py:~1234`: _"because every order = target − broker_net, a phantom can never accumulate, and a long→short reversal becomes one (−2N) order instead of a close-then-open that a netting account would just flatten."_ If `_broker_net_now()` returns `None`, the lane refuses to drive — _"broker unreadable → don't drive blind."_

## 8. Reconciliation — broker as source of truth

`[V]` `reconcile.py` encodes the stance that the **broker is the system of record**, not the agent's ledger. Its module docstring specifies that each `BrokerSnapshot` is recorded into the signed audit chain _before_ the gate decides from it — so the evidence precedes the decision in the chain, not the reverse.

`[V]` Divergence classes:

| Class             | Definition                                       |
| ----------------- | ------------------------------------------------- |
| **Orphan**        | Broker holds a trade the ledger does not         |
| **Phantom**       | Ledger holds a position the broker no longer has |
| **Unit mismatch** | Both hold the trade, quantities disagree         |

`[V]` Position divergence — not P&L divergence — drives the drift verdict. The docstring is explicit: position divergence _"is what the orphaned-longs incident was,"_ while value gaps decompose into spread, financing and direction and belong to a separate attribution bundle.

`[V]` `_apply_reconcile` (`engine.py:374`) adopts orphans into the ledger so the gate can see them (the orphaned-longs fix), and market-closes phantoms with a `RECON_CLOSE` audit event keyed to the trade id.

## 9. The drift state machine

`[V]` When divergence stands, the engine sets `_reconcile_drift` and **refuses all new opens** while allowing existing rungs to wind down at their own exits. Logged verbatim:

```
[DRIFT] 1 divergence(s) -> refusing new opens; resolve via console
        'adopt broker truth' or a clean reconcile
```

`[V]` Escalation (`_check_drift_escalation`, `engine.py:490`) is identity-keyed: a standing drift keeps one signature so age reflects the true duration rather than resetting each cycle. Stage 1 raises a console banner at ~300s. Stage 2 attempts in-session auto-resolve after a longer window — **but only for net-managed lanes**. `engine.py:533` gates grid auto-adoption behind `drift_auto_adopt_grids`, default **false**, because adoption flattens a grid's rung structure, which is a consequential change to the strategy that currently works. Grid drift therefore waits for a human.

`[P]` Observed escalation ages cluster tightly at 300–303 seconds and reset — transient, self-clearing. Contrast with the pre-E5 behavior described in §31, which never cleared in 76 hours.

## 10. Lifecycle and the control channel

`[V]` `runtime/control.json` is a plain inspectable JSON file — no IPC, fully replayable. `_poll_control` (`engine.py:701`) accepts four commands:

| Command   | Effect                                                                           |
| --------- | ---------------------------------------------------------------------------------- |
| `run`     | lifecycle → `running`; audit `RESUME`                                            |
| `drain`   | lifecycle → `draining`; stop opening, honor strategy exits                       |
| `kill`    | lifecycle → `killed`; `_force_close_all()` market-closes every position, audited |
| `resolve` | Adopt broker truth for a named instrument, or all currently-drifted lanes        |

`[V]` Commands are deduplicated on the **full** signature `f"{cmd}:{instrument}:{nonce}"`, so a repeated `resolve` against a second drift still fires while a steady `run` does not re-audit every tick.

`[V]` Breaker state derives from a high-water-mark equity reference persisted in `runtime/equity_hwm.json`. The watchdog launches the engine with `KEEL_CLOSE_ONLY_DD=-0.15` and `KEEL_HALT_DRAWDOWN=-0.25`.

`[V]` The console's **Start button was removed entirely** (commit `c7cf906`) because it launched the funded account outside the watchdog in sim mode. It was replaced with a **Resume** button that does nothing but write `control.json`. The agent can only be started through its supervisor.

## 11. The supervisor

`[V]` `keel_watchdog.sh` (v2, rewritten after the 2026-07-14 kill test) implements _"restart unless explicitly told not to."_ Version 1 restarted only on exit code 1 and treated SIGTERM (143), SIGKILL (137) and OOM as deliberate — a signal-killed engine would stay down. The current policy:

**Stays down only when:** `runtime/WATCHDOG_STOP` exists, `runtime/HALT` exists, or the engine exits `0` (clean in-process shutdown).

**Restarts on:** any other exit — crash, signal death, OOM, nonzero, aborted.

`[V]` **Exit code 75 is special-cased.** It is the feed-auth gate's signal that the engine correctly refused to arm blind. It is not a crash and not deliberate, so it retries at 30s and is explicitly **excluded from the crash-loop guard** — a twenty-minute broker outage must not trip the loop-stop.

`[V]` Crash-loop guard: 5 restarts in a 300s window → stop with _"Likely a persistent bug."_ Engine stdout is teed to `runtime/engine.log` with 50MB rotation; the engine's real exit code is read from `PIPESTATUS[0]`, not tee's.

## 12. The feed gate

`[V]` Before arming, the engine must authorize against the broker and confirm a live feed. Five attempts with 2/4/6/8/10s backoff, then:

```
[FEED-GATE ] FAILED after 5 attempts. Refusing to arm blind.
  Exiting code 75 (feed-not-ready) for the watchdog to retry.
```

`[V]` In-session, pricing errors retry 8 times with exponential backoff before `[FEED-FATAL] too many consecutive OANDA errors; raising for supervisor restart` → exit 1. The OANDA read timeout is a finite **15.0s** (commit `9104eb5`); it was previously `timeout=None`, which could hang the engine indefinitely against an unresponsive socket.

## 13. FIFO regulatory compliance

`[V]` OANDA Corporation US accounts fall under NFA Compliance Rule 2-43(b), which forces first-in-first-out closure ordering. A grid holding multiple rungs at the same unit count, each with its own resting take-profit, will have closes rejected as FIFO violations.

`[V]` `fifo_unique_rung_units` forces each rung to a unique unit count so the broker can disambiguate. The gate is at `engine.py:1114`:

```python
if getattr(cfg, "fifo_unique_rung_units", True) and lane.strategy_kind == "grid":
```

This keys on `strategy_kind`, which is why `strategy_kind` could not simply be changed to introduce a new governance label (§32.6).

---

# PART II — ECHOGATE

## 14. What EchoGate is

`[V]` EchoGate is a compiled binary — approximately **365 KB** — at `echogate/build/echogate`. It is a deterministic admission-control layer. It receives a _proposal_ and a _policy_ and returns a _verdict_ plus a sealed receipt.

`[V]` Four properties define it:

1. **Offline.** No network call exists in the decision path. No remote model, no API, no external dependency between a proposal and its verdict.
2. **Deterministic.** Identical inputs produce an identical verdict, every time, which is what makes third-party replay verification meaningful.
3. **Fail-closed.** `strict_mode: true`. A proposal type absent from the allow-list is refused before any numeric rule is evaluated. There is no default-permit path.
4. **Strategy-blind.** It is never given the information required to form an opinion about whether a trade is _good_. It evaluates only whether a trade is _permitted_.

`[V]` Its documentation states the scope constraint plainly (`echogate/docs/policy_rule_types.md`): _"EchoGate is not a general-purpose policy engine. Its policy layer exists to support behavior-change admission only."_

## 15. The invocation contract

`[V]` `keelbot/gate.py` wraps the binary. `EchoGateGate.__init__` (`gate.py:35`) takes `echogate_dir`, `policy_path`, `config_path`, `runtime_dir`, `instrument`, `direction`, and raises `FileNotFoundError` if the binary is absent.

`[V]` `evaluate(routing, intent, snapshot)` (`gate.py:223`) writes the proposal, a per-proposal-type memory pool (`_memory_for`, `gate.py:123`) and an ancestry file into the runtime directory, then:

```python
cmd = [..., "--ancestry", a_path, "--policy", self.policy_path, ...]
out = subprocess.run(cmd, cwd=self.echogate_dir, capture_output=True, text=True)
```

A non-zero exit raises `RuntimeError(f"echogate failed: {out.stderr or out.stdout}")` — the gate does not swallow a governor failure and proceed.

`[V]` Context assembly (`gate.py:175–183`) sets defaults then submits:

```python
context.setdefault("rung",     int(intent.rung))
context.setdefault("kind",     intent.kind)
context.setdefault("pair",     self.instrument)
context.setdefault("strategy", routing.source.replace("_strategy", ""))
...
"proposal_type": routing.proposal_type,
```

`[V]` A scaling note at `gate.py:156` records a real defect class: crypto notionals scaled by 1e6 would otherwise cause every crypto open to be _wrongly_ rejected on portfolio caps. The fix is in the wrapper, not the policy.

`[V]` A separate path, `evaluate_mode_change(proposed_mode, equity_usd, ...)` (`gate.py:291`), governs posture changes rather than trades. Its docstring: _"De-risking admits freely; escalation can be rejected when the book is in drawdown or over-loaded (per policy). The mode is applied by the engine only on ADMIT."_ Governance applies to the agent's self-modification, not only its trades.

## 16. The proposal — the only thing that crosses

`[V]` `RiskClassifier.classify()` (`core.py:250`) constructs what EchoGate sees:

```python
def classify(self, intent, ledger, regime, price, equity_usd,
             source="grid_strategy", strategy="grid"):
    is_open  = intent.kind in ("open_long", "open_short")
    is_short = intent.kind in ("open_short", "close_short")
    side     = "short" if is_short else "long"
    proposal_type = f"{strategy}_{'open' if is_open else 'close'}_{side}"
```

`[V]` The `findings` dict — the complete context EchoGate receives:

```
rung, kind, side, units, open_rungs, open_units, gross_units,
equity_usd, rung_notional, prospective_gross_notional,
prospective_net_notional, total_pnl_usd, drawdown_ratio, leverage
```

Plus portfolio-level ratios from the snapshot: `portfolio_gross_ratio`, `portfolio_net_ratio`, `pair_gross_ratio`, `strategy_pair_gross_ratio`. Plus the payload field `tp_pips`.

`[V]` Prospective quantities are computed _before_ the trade, not after. For an open: `open_rungs + 1`, `gross_units + |units|`, `net_units + signed_intent`. The gate therefore rules on the state the trade _would_ create.

`[V]` Routing targets encode the breach class: `trade_execution` (clean, or trend-filter conflict), `leverage_engine` (`leverage_above_1x`, `core.py:281`), `risk_envelope` (any envelope breach, `core.py:303`). `risk_envelope` is a **protected target** in policy — a proposal routed there cannot be admitted to act on the envelope itself.

`[V]` **What never crosses the boundary:** the strategy's signal, its parameters, its reasoning, its expected edge, its backtest, or any justification for why this trade. EchoGate cannot form an opinion on merit because it is never given the inputs to one. This is the architectural claim, and it is enforced by the shape of `classify()`, not by convention.

## 17. The policy bundle

`[V]` `policies/policy.json` — 1,043 lines. Top-level structure:

```json
{
  "policy_version": 1,
  "strict_mode": true,
  "authorized_sources":      [ "grid_strategy", "adaptive_selector",
                               "breakout_strategy", "meanrev_strategy",
                               "cognition_strategy", "invgrid_strategy" ],
  "authorized_targets":      [ "trade_execution", "posture_controller" ],
  "protected_targets":       [ "risk_envelope" ],
  "allowed_proposal_types":  [ ... 23 entries ... ],
  "rules": [ ... ]
}
```

`[V]` The six effective rule classes (`echogate/docs/policy_rule_types.md`): `AuthorizedSource`, `AuthorizedTarget`, `AllowedProposalType`, `ProtectedTarget`, `ExplicitDenyRule`, `ExplicitAllowRule`.

`[V]` The policy model (`echogate/docs/policy-model.md`) specifies: _"Rules currently support simple glob matching using `_`and`?`. Strict failures cause rejection."_

`[V]` Each numeric rule has the form:

```json
{
  "rule_id": "max_strategy_pair_gross_pct",
  "rule_type": "ratio_bound",
  "target_pattern": "*",
  "proposal_type_pattern": "*grid_open_long",
  "allow": true,
  "strict": true,
  "field": "context.strategy_pair_gross_ratio",
  "op": "<=",
  "value": 0.12
}
```

Two rule types exist in the live policy: `numeric_bound` (field vs. constant) and `ratio_bound` (field vs. `value × field2`, where `field2` is almost always `context.equity_usd`).

## 18. The complete live cage

`[V]` Eleven rules per side govern grid opens, plus one global:

| #   | `rule_id`                     | Field                                 | Op   | Bound          |
| --- | ------------------------------ | -------------------------------------- | ---- | -------------- |
| 1   | `max_rungs`                   | `context.open_rungs`                  | `<=` | 10             |
| 2   | `max_drawdown_pct`            | `context.total_pnl_usd`               | `>=` | −0.02 × equity |
| 3   | `max_rung_pct`                | `context.rung_notional`               | `<=` | 0.05 × equity  |
| 4   | `max_exposure_pct`            | `context.prospective_gross_notional`  | `<=` | 0.5 × equity   |
| 5   | `max_net_pct`                 | `context.prospective_net_notional`    | `<=` | 0.5 × equity   |
| 6   | `exit_required_min`           | `payload.tp_pips`                     | `>=` | 5              |
| 7   | `exit_bounded_max`            | `payload.tp_pips`                     | `<=` | 20             |
| 8   | `max_portfolio_gross_pct`     | `context.portfolio_gross_ratio`       | `<=` | 0.75           |
| 9   | `max_portfolio_net_pct`       | `context.portfolio_net_ratio`         | `<=` | 0.35           |
| 10  | `max_pair_gross_pct`          | `context.pair_gross_ratio`            | `<=` | 0.25           |
| 11  | `max_strategy_pair_gross_pct` | `context.strategy_pair_gross_ratio`   | `<=` | 0.12           |
| —   | `leverage_cap` (pattern `*`)  | `context.prospective_gross_notional`  | `<=` | 1.0 × equity   |

Each has a `_short` twin on `grid_open_short`. Parallel blocks exist for `breakout_open_*` (suffix `_bk`), `meanrev_open_*` (`_mr`), and `cognition_open_*` (`_cl`/`_cs`). Mode changes are governed by `set_mode_aggressive_drawdown` (≥ −0.005 × equity), `set_mode_aggressive_gross` (≤ 0.6), and `set_mode_normal_drawdown` (≥ −0.015 × equity).

`[V]` **Exposure is bounded at five simultaneous scopes:** per rung (3), per lane (4, 5), per instrument pair (10), per strategy-and-pair (11), and across the whole portfolio (8, 9). Concurrency across venues therefore cannot be used as a mechanism to exceed total risk.

`[V]` **Closes are unconstrained by design.** `grid_close_long` and `grid_close_short` appear in `allowed_proposal_types` but as a `proposal_type_pattern` in **zero** rules. An exit need only be a permitted type; no numeric bound can refuse it. This is deliberate — a governance layer that can block an exit can trap capital.

## 19. Verdicts and scores

`[P]` Every evaluation returns a verdict plus four scores: **L** (legality), **E** (echo), **A** (ancestry), **S** (stability). The observed refusal signature in production:

```
[REJECT] EUR_USD open_long rung -1 @ 1.16972  L=0.10 E=1.00 A=0.60 S=1.00
```

A legality score pinned at 0.10 with the other three nominal is the signature of a policy-rule violation rather than a provenance or stability problem.

`[P]` Verdicts observed in Keel production logs: **ADMIT**, **REJECT**, **QUARANTINE**. `QUARANTINE` holds a proposal rather than passing or discarding it; the engine maintains a `_quarantine_log` (capped at 200 entries) with instrument, strategy, kind, rung, reason, price, receipt prefix, timestamp, state, and a plain-language description, and sweeps it on a TTL (`_maybe_sweep_quarantine`, `engine.py:1185`).

`[X]` **A correction on the record.** The EchoGate/Keel white paper produced in this program describes a fourth verdict, `DEFER`. That claim could not be substantiated from source during the verification pass; it appears to have been imported from **Neptune's** distinct four-way ruling (`ADMIT` / `REJECT` / `CONFLICTED` / `INDETERMINATE`), which is a different engine judging different objects. **Verify `DEFER` against the binary or remove it before external publication.** Keel production logs show three verdicts only. *(Post-review note: `DEFER` remains a supported EchoGate capability generally; it is simply not wired into Keel's current policy, because Keel's specific cage has not needed it. The three-verdict framing above is correct for every claim scoped specifically to Keel.)*

## 20. Receipts and the audit chain

`[V]` Every evaluation emits a canonical receipt. Receipts are **hash-chained**: each entry commits to the prior ledger head, so altering a historical decision breaks the chain from that point forward. `gate.py:145` records `"policy_hash": "keel_v0"` in the receipt envelope, binding each verdict to the policy version that produced it.

`[V]` `gate.audit_event(key, kind, detail)` writes lifecycle and reconciliation events into the same signed chain — `RECOVERED`, `RECON_CLOSE`, `RECON_ADOPT`, `CLOSED_TP`, `KILL_CLOSE`, `COMPOSITE_ABORT`, `DRAIN`, `RESUME`, `KILL`. The chain records not only decisions but the broker evidence the decisions were made from, and in that order.

`[P]` The adjacent Neptune engine demonstrates the same receipt discipline at scale with externally verifiable integrity: **1,394 records, chain intact, 1,394/1,394 receipts sealed and chain-linked, 1,397 evidence files matching sealed digests exactly.** A representative sealed envelope (`OY_S20260808_0126`, schema `neptune.receipt.v1`) carries `config_digest`, `data_digest`, `params_digest`, `strategy_digest`, `instrument_digests` and `ledger_head_prior` — all SHA-256 — alongside `declared_trials: 225` sealed _before_ evaluation, so the raw Sharpe of 1.8887 is deflated against declared search breadth to 1.3488. The verdict carries its own margin: `T = 0.9541` against `t_min = 0.95`.

`[P]` **A refusal is receipted exactly as an approval is.** In the Neptune corpus, 845 REJECTs are sealed and chain-linked identically to the single ADMIT. The trustworthy "no" is counted, not discarded.

## 21. Contradiction preservation

`[V]` EchoGate does not collapse conflicting evidence into a false resolution. On contradiction it writes a record (`gate.py:209–218`) with `contradiction_id`, `proposal_id`, `conflicting_gates` (observed: `["arbiter", "legality"]`), `resolution_score`, `status: "open"`, and `default_resolution` (default `"drop"`). The unresolved state is _recorded as unresolved_ rather than silently resolved in either direction.

`[V]` This is the implementation of the LuxCrypta doctrinal line _"contradiction should be examined, not erased,"_ and it is the reason the verdict vocabulary is larger than allow/deny: a binary gate forces every ambiguous case into one of two wrong answers.

---

# PART III — HOW ECHOGATE GOVERNS KEEL

## 22. The seam

`[V]` The entire governance relationship reduces to two adjacent statements at `engine.py:1386–1392`:

```python
routing = lane.risk.classify(intent, lane.ledger, lane.regime, price,
                             self.last_equity_usd,
                             source=lane.source, strategy=lane.strategy_kind)
v = lane.gate.evaluate(routing, intent, snapshot=self._portfolio_snapshot())
self.verdicts[v.verdict] += 1
...
if v.verdict == "ADMIT":
    self._execute_admitted_intent(lane, intent, price)
```

`[V]` There is exactly one path from intent to order, and it runs through `gate.evaluate`. There is no bypass, no fast path, no cached-verdict shortcut, and no exception handler that proceeds on governor failure. A second, identical seam exists for atomic multi-leg groups at `engine.py:1465`, which aborts the whole group if the engine is not in a state to open (`COMPOSITE_ABORT`).

## 23. The label is the rulebook selector

`[V]` The mechanism by which policy attaches to a proposal is a **string**. `classify()` composes `proposal_type` from the `strategy` argument; EchoGate glob-matches that string against `proposal_type_pattern` in every rule. The string therefore selects the rulebook.

`[V]` This has a profound and non-obvious consequence, discovered empirically (§32): **changing a strategy's governance label changes which laws apply to it.** That is correct and desirable — a momentum strategy and a reversion strategy should not be judged by identical exit rules. But it means the label must be chosen deliberately, and that governance identity must be separable from execution identity. In the current build they are deliberately decoupled: `strategy_kind` stays `"grid"` for FIFO and sizing, while the classify-time label is `"invgrid"` for governance.

## 24. Defence in depth — five independent refusal points

`[V]` A proposal must survive all five to become an order:

| Layer                          | Mechanism                                      | Refuses when                                            |
| ------------------------------- | ------------------------------------------------- | ---------------------------------------------------------- |
| **1. Strategy**                | `on_tick` emits nothing                        | No level crossed, rung already open                     |
| **2. Engine state**            | `engine.py:1375`                               | Halted, draining, unarmed, drifting, equity stale       |
| **3. Strategy discipline**     | Trend filter `1369`, backoff `1383`            | Stacking into a trend; re-proposing an unchanged reject |
| **4. Source & type whitelist** | `authorized_sources`, `allowed_proposal_types` | Unknown source or proposal type — **fail-closed**       |
| **5. Numeric policy**          | 11 rules per side + `leverage_cap`             | Any strict bound breached                               |

`[V]` Layers 1–3 are agent-side and in principle revisable by changing Keel. Layers 4–5 are governor-side and are not reachable from the agent. The watchdog, the feed gate and the broker's own FIFO rules form further independent constraints outside this chain entirely.

## 25. What the agent cannot do

`[V]` By construction, Keel cannot: raise its own ceilings (`risk_envelope` is a protected target); place an order without an admitting verdict (single seam, no bypass); act without writing to the signed chain (snapshot recorded _before_ the decision); start itself outside its supervisor (console Start button removed, `sh/` installers guarded, direct `run_oanda.py` invocation prohibited by standing rule); or escalate its own posture without governance (`evaluate_mode_change` applies the mode only on ADMIT).

---

# PART IV — WHAT HAS BEEN PROVEN

## 26. Summary ledger

| Claim                                                   | Grade | Evidence                                             |
| --------------------------------------------------------- | ----- | ------------------------------------------------------- |
| Fails closed rather than acting blind                   | `[P]` | Three production events, Aug 9 / 15 / 20             |
| Recovers unattended from infrastructure failure         | `[P]` | Two full outage cycles, no human in loop             |
| Reconciles to broker truth per-trade with IDs preserved | `[P]` | Aug 7 cutover, Aug 15, Aug 20                        |
| Divergence blocks new exposure                          | `[P]` | `[DRIFT] refusing new opens` in production           |
| Governs a strategy it cannot understand                 | `[P]` | ~48 sustained REJECTs, zero orders sent              |
| Cage bites on exposure breach after relabel             | `[P]` | Executed assertions, §33                             |
| Fail-closed on malformed/unknown input                  | `[P]` | Binary rejected hand-built proposal; typo test       |
| Extension by declaration, not modification              | `[P]` | 18 rules globbed, binary untouched, commit `ec2dcc2` |
| Audit volume is tractable                               | `[P]` | 196,000+ → ~40 entries, 0.07s on 87MB                |
| Suite integrity across governance change                | `[P]` | 125 tests green                                      |
| Receipts sealed, chained, replayable                    | `[P]` | 1,394/1,394 + 1,397 evidence files (Neptune)         |
| invgrid stop-out reaches the broker                     | `[U]` | **Never executed against a real broker**             |
| EchoGate glob semantics on `*grid_open_*`               | `[U]` | Proven against reconstructed model only              |
| Boot recovery after reboot (§4d)                        | `[U]` | Never observed; see §37                              |
| HFT latency                                              | `[U]` | Unbenchmarked                                        |

## 27. E1 — the finite timeout

`[V]` Commit `9104eb5`. The OANDA read timeout was `timeout=None` — an unresponsive socket could hang the engine indefinitely with positions open and no supervision possible, because a hung process is not a crashed process and the watchdog cannot see the difference.

`[P]` Proven in production during a real power-cut/DNS outage:

```
[FEED-RETRY] pricing error (1/8): ReadTimeout: ... (read timeout=15.0); retrying in 2.0s
[FEED-RETRY] pricing error (2/8): ConnectionError: ... NameResolutionError ...
[FEED-FATAL] too many consecutive OANDA errors; raising for supervisor restart.
Exiting code 1 for supervisor restart.
```

The engine **failed closed rather than hanging**. The watchdog relaunched it. It then hit the feed gate, exhausted five attempts, and refused to arm:

```
[FEED-GATE ] FAILED after 5 attempts. Refusing to arm blind.
  Exiting code 75 (feed-not-ready) for the watchdog to retry.
```

Twice. Then, on feed restoration:

```
[FEED-GATE ] OANDA authorized; account NAV $99.80. Feed live. Arming.
[RECONCILE ] EUR_USD: adopted short 1u @ 1.15783 (trade 7667) -> ledger rung 1000
[RECONCILE ] GBP_USD: adopted short 1u @ 1.35119 (trade 7664) -> ledger rung 1000
[RECONCILE ] GBP_USD: adopted short 2u @ 1.34897 (trade 7631) -> ledger rung 1001
[ARMED     ] broker sync complete: realized baseline $-86.67, NAV $99.80,
             3 open trade(s). Trading enabled.
```

`[P]` Process evidence from the same event: watchdog PID 69110 at `etime 08-03:54:26` (8 days — never died), engine PID 43517 at `03:17:05` (3 hours — restarted). The supervisor survived; the engine cycled under it. **Unattended throughout.**

## 28. E5 — the self-generating drift

`[V]` Commit `272e0f5`. The highest-value defect in the record, and the one that most clearly demonstrates the class of failure Continuity Engineering exists to address.

`[V]` **Root cause.** `_adopt_broker_truth` collapsed a grid lane's multiple broker trades into a single netted ledger rung with `trade_id=""`. That rung was invisible to per-trade reconciliation (no ID to match) but fully counted in net exposure. The real broker trades therefore reappeared as orphans on the next cycle and were re-adopted — indefinitely. The engine's own source comment at `engine.py:~667` records it: _"sat blocked ~49h while the resolver regenerated the drift it was clearing."_

`[V]` GBP/USD was blocked for **76 hours**. The resolution mechanism was the cause.

`[V]` **Fix.** Adopt each broker trade individually, preserving `trade_id`, for grid lanes.

`[P]` **Proven three times in production.** At the Aug 7 cutover, GBP adopted as two discrete rungs (trades 7600, 7581) rather than one collapsed rung, and stayed in sync through a 15-minute watch. Again through the Aug 9 outage. Again Aug 15 and Aug 20 (trades 7664, 7631 shown above).

`[P]` **Behavioral proof of the fix:** post-E5 drift escalations clock at 300, 301, 302, 303 seconds and **reset**. The Aug 13 episode self-cleared at 05:22. E5 converted _permanent_ drift into _transient_ drift — precisely its design intent. The 1,167-iteration adopt/reconcile loop that characterized the pre-fix behavior is now named by the Captain's Log as a single episode rather than 1,167 events.

## 29. E3 / E3b — the inert ceiling

`[V]` Commits `7487626`, `0cc8cc9`. `max_rungs` was configured in `keel_modes.json`, where `make_config` never read it — the value was **shadowed and inert**. Moved to `keel_pairs.json` and set per-pair (EUR 3, GBP 2) to fit one 12% strategy-pair ceiling across two instruments at materially different prices.

`[V]` The lesson is general: a configured constraint that is never read is indistinguishable from no constraint at all, and nothing in the system surfaced the difference until it was looked for.

## 30. E4 — the console that lied

`[V]` Commit `c7cf906`. The Keel console defaulted `KEEL_POLICY_FILE` to the **simulation** policy and displayed ceilings of 300/200/250/250% while EchoGate was in fact enforcing **75/35/25/12%**. The operator's instrument was reporting a cage four to twenty times looser than the one actually in force.

`[V]` Fixed by correcting the default. The same commit removed the console's Start button (it launched the funded account outside the watchdog in sim mode — a live hazard) and replaced it with a Resume button that only writes `control.json`.

`[V]` Related: C2 (`0c49649`) rewrote the status pill to read lifecycle and freshness _together_ — `WEDGED? claims RUNNING · no status written for Nh` — because a wedge had previously presented as two separately-plausible badges for **46 hours** before anyone noticed.

## 31. The test-isolation defect

`[P]` Commit `244f63c`. `tests/test_engine_netaware.py` wrote into the **real** `runtime/` directory rather than a scratch path. Running `pytest` therefore injected a **$99,700 fixture NAV** into `runtime/equity_hwm.json` — the live drawdown reference — and overwrote `runtime/control.json`, the channel the live engine reads commands from.

`[P]` **This halted the live cutover twice** before the cause was found. Fixed to `tempfile.mkdtemp()`.

`[V]` The downstream consequence is that `Keel_Cutover_Runbook.md` §1.2 still listed "run pytest" as a cutover _precondition_ — the written procedure instructed the operator to perform the exact action that caused the halt. Correcting that line remains an open documentation item.

## 32. The inverted-grid case study — governance demonstrated adversarially

This is the most complete demonstration in the record, because it was not designed as a demonstration. It was an attempt to ship a feature, and the governance layer refused it correctly at a seam nobody had anticipated.

**32.1 The request.** `[P]` Replace the live reversion grid with its exact inverse — momentum — on the funded EUR/USD and GBP/USD lanes, live, with explicit acceptance of strategy risk and of running previously-unexecuted code.

**32.2 The strategy already existed.** `[P]` `git log --oneline -- keelbot/invgrid.py` returned exactly one commit: `a8bcfa7 Inverted grid with momentum`. No test file referenced it. Nothing in `runtime/` mentioned it. A single commit with no follow-up is the signature of code written in one sitting and **never executed** — not live, not in simulation, not once.

**32.3 The dead stop branch.** `[V]` `engine.py:1380`:

```python
if intent.kind in ("close_long","close_short") \
        and getattr(lane.exec, "broker_take_profit", False):
    return
```

Grid lanes set `broker_take_profit = True`. `InvertedGridStrategy` emits its **stop-out** as a plain `close_long`/`close_short` — the same kind as a take-profit close. Therefore every stop-out would have hit this branch and returned silently. The position the stop was meant to protect would have stayed open, and the strategy would have continued adding rungs into an adverse run **with no brake**. The one feature that makes momentum survivable was wired into a dead path.

`[V]` This is the clearest instance of the recurring pattern: two independently correct mechanisms — a close-swallow for broker-managed exits, and a self-managed stop — colliding at their boundary.

**32.4 The coupling cascade.** `[V]` Setting `broker_take_profit = False` fixes the routing (the close falls through to the per-rung market-close at `engine.py:1438`), but `engine.py:1120` would then still attach a _broker_ take-profit to invgrid opens, fighting the strategy's own exit management. Resolved with a `self_managed_exits` lane flag gating three behaviors: no broker TP on opens (`1120`), skip the reversion trend filter (`1369` — opening into trends is the entire point of momentum), and skip the adaptive-grid step (`1033`, defensive).

**32.5 The governance refusal — the centerpiece.** `[P]` With the strategy wired and the lane armed on a flat book at a one-rung cap, production logged a sustained stream:

```
[REJECT] EUR_USD open_long  rung -1 @ 1.16972  L=0.10 E=1.00 A=0.60 S=1.00
[REJECT] GBP_USD open_short rung -1 @ 1.36491  L=0.10 E=1.00 A=0.60 S=1.00
[REJECT] GBP_USD open_long  rung -2 @ 1.36702  L=0.10 E=1.00 A=0.60 S=1.00
... every proposal, both directions, both instruments, sustained
```

`[V]` **Diagnosis.** invgrid opens were labeled `grid_open_*`, so EchoGate applied the grid rulebook — including `exit_required_min`: `payload.tp_pips >= 5`. invgrid opens carry `tp_pips = 0` **by design**, because the strategy manages its own exits. `0 >= 5` is false; `strict: true`; REJECT. Every open, both directions. `L = 0.10`.

`[P]` **What this proves.** The governance layer detected that a new strategy's _exit architecture_ was structurally incompatible with the policy it had inherited — and refused every proposal, before any order reached the venue — **without being told anything about momentum, reversion, or market direction.** It did not evaluate the strategy. It evaluated permission, found a strict rule violated, and failed closed. Zero orders transmitted. Zero capital moved.

`[V]` An initial diagnosis attributed the refusals to a trend-filter rule in policy. Reading the policy verbatim disproved that: **no trend predicate exists in `policy.json` at all.** The `trend_filter_conflict` tag on some rejects originates in the _engine_ (`core.py:300`), not the governor. The universal killer was the exit rule. This correction is recorded because reasoning from a grep rather than the file is exactly the error class this program documents.

**32.6 The resolution — extension by declaration.** `[V]` The fix adds governance vocabulary without modifying the governor:

1. `authorized_sources += "invgrid_strategy"`
2. `allowed_proposal_types += invgrid_open_long/close_long/open_short/close_short`
3. **18 exposure rules** (nine per side) broadened from literal `grid_open_long` / `grid_open_short` to glob `*grid_open_long` / `*grid_open_short` — which matches both `grid_` and `invgrid_`, because "invgrid" ends in "grid"
4. **The four exit rules left literal** to `grid_open_*`, so they continue to bind the reversion grid and simply do not match invgrid
5. At `classify`, self-managed lanes pass `strategy="invgrid"` and `source="invgrid_strategy"` — **`strategy_kind` stays `"grid"`**, preserving FIFO sizing at `engine.py:1114` and all internal grid behavior

`[V]` The binary was never modified. One policy file and two call sites changed. The momentum strategy inherits the **identical capital cage** — byte-for-byte the same rules, not copies — while shedding only the broker-TP requirement written for a different exit architecture. The reversion grid's rules remain in force, unchanged, for the strategy they were written for.

`[P]` Applied cleanly: `globbed 18 exposure rules to *grid_open_* (exit rules left literal)` followed by an in-patch assertion `verified: exit rules remain literal grid_open_*`. Committed as `ec2dcc2`.

## 33. Executed test evidence

`[P]` **Strategy-level (run against `keelbot.core.Config`, 4 passed):**

| Test                                                         | Result |
| -------------------------------------------------------------- | ------ |
| Long opens on up-break; take-profit close fires              | PASS   |
| **Long stop-out emits a market close**                       | PASS   |
| **Short stop-out emits a market close**                      | PASS   |
| Percent-based `spacing`/`take_profit` recompute in `start()` | PASS   |

The fourth test closed a latent trap: the original `invgrid.py` read `cfg.spacing` and `cfg.take_profit` directly and never recomputed them in `start()` the way `GridStrategy` does (`core.py:201–202`). On a percent-geometry pair this would have collapsed **every rung onto the center price**. Behavior-neutral on the current pip-based FX pairs; latent and severe on any future crypto lane.

`[P]` **Policy-level (run against a reconstructed model of the documented glob-whitelist semantics):**

| Assertion                                    | Expected                           | Result |
| ----------------------------------------------- | ------------------------------------- | ------ |
| Reproduce the bug: grid-labeled open, `tp=0` | REJECT `exit_required_min`         | ✓      |
| `invgrid_open_long`, `tp=0`, within caps     | ADMIT                              | ✓      |
| `invgrid_open_short`, `tp=0`, within caps    | ADMIT                              | ✓      |
| `invgrid_open_long`, gross 0.9 > 0.5         | REJECT `max_exposure_pct`          | ✓      |
| `invgrid_open_short`, rungs 99 > 10          | REJECT `max_rungs_short`           | ✓      |
| Real `grid_open_long`, `tp=0`                | REJECT (unchanged)                 | ✓      |
| Real `grid_open_long`, `tp=18`               | ADMIT                              | ✓      |
| Typo'd type `invgrd_open_long`               | REJECT `proposal_type_not_allowed` | ✓      |
| `invgrid_close_long` (the stop-out)          | ADMIT                              | ✓      |

`[P]` The two load-bearing results: **the cage still bites** (exposure and rung breaches are refused under the new label), and **a fumbled label fails closed** (a typo yields rejection, never an ungoverned order).

`[U]` **Scope limit, stated plainly.** These policy assertions ran against a reconstruction of EchoGate's documented semantics using Python `fnmatch`, not against the binary. Three attempts to drive the real binary from a test harness failed on harness construction errors — a doubled path (`echogate/build/build/echogate`), a malformed payload (`error: Missing required key: equity_usd`), and a wrong class name (`RiskEnvelope`). The binary's glob matcher is **presumed** equivalent to `fnmatch` for these patterns; it is not confirmed.

`[P]` One incidental proof from that failure: when handed a hand-constructed proposal missing a required context field, the binary **refused outright** — `error: Missing required key: equity_usd` — rather than defaulting, inferring, or proceeding on a partial payload.

`[P]` **Full suite: 125 tests green** after every change in this cycle, including after the governance relabel. The diff touched five files: `config/keel_pairs.json`, `keelbot/engine.py`, `keelbot/invgrid.py`, `policies/policy.json`, `run_oanda.py`.

## 34. Audit tractability

`[P]` The Captain's Log (commits `4986355`, `6988cd7`, `41687cd`, `80693cc`) reduces **196,000+ raw audit records to approximately 40 operator-readable entries** through heartbeat suppression, repeat-collapsing, and cycle detection — naming a 1,167-iteration adopt/reconcile loop as _one_ episode rather than 1,167 events. Bounded reads with a 12MB tail cap complete in ~0.07s against an 87MB log. 34 dedicated tests.

`[V]` The headline reports **absence** as a first-class signal: _"Last trade decision Nh ago."_ A system that has silently stopped deciding looks identical to a quiet market unless absence is measured.

## 35. Multi-venue governance

`[V]` One gate, one policy bundle, one receipt chain govern concurrent lanes across venues. FX runs through `OandaV20Execution`; a crypto lane runs through Kraken behind `KEEL_CRYPTO=1`. Lane configuration, sizing and ledger are per-lane; aggregate ceilings (rules 8–11) are enforced across the whole book.

`[X]` **Required precision for any external claim.** FX lanes have run **live funded**. Crypto lanes have run **in simulation** (`KrakenSimExecution`, `KrakenSimFeed`). The governance path is identical in both — same binary, same policy, same chain — but the execution endpoints differ. **Do not claim live crypto execution.** The defensible claim is: _one policy, one gate, two venues, two asset classes, concurrent, with FX live and crypto in simulation._

`[V]` Venue idiosyncrasy is handled in the **agent**, not the governor: FIFO ordering constraints live in Keel's sizing logic; EchoGate's ceilings are untouched by venue rules. This keeps the policy layer portable.

---

# PART V — WHAT HAS NOT BEEN PROVEN

`[U]` **36. The inverted grid's stop-out has never executed against a real broker.** Strategy-level emission is unit-proven (§33). The _routing_ — `broker_take_profit=False` → fall-through at `engine.py:1380` → per-rung market close at `engine.py:1438` → ledger close → reconcile — has never run against OANDA. This remains the single highest-value open item. Current mitigation: `max_rungs = 1` per lane, so an unbraked position is bounded to one rung.

`[U]` **37. §4d — unattended boot recovery.** No LuxCrypta agent, including Keel's own watchdog, has ever been observed starting itself after a full machine reboot. C5 established that launchd does not honor documented autostart behavior on this host: `load`/`bootstrap` register without starting, `RunAtLoad` does not fire, `KeepAlive` does not relaunch a killed job; only `launchctl kickstart -p gui/$(id -u)/<label>` works. Four diagnoses were tried and rejected. Manual restart is accepted policy for C5; §4d remains open.

Both August outages stayed benign **only because the MacBook's internal battery carried the machine through**. That is two live data points confirming the gap rather than closing it.

`[V]` **Scope of this limitation.** It is a property of the **agent's host deployment**, not of EchoGate. The governor is invoked per-decision as a local binary and has no resident process requiring startup. Any publication must keep that distinction, or a deployment gap will be read as a governance-layer defect.

`[U]` **38. EchoGate glob semantics against the binary** — see §33.

`[U]` **39. High-frequency latency is unbenchmarked.** The architecture is suited to low-latency deployment: a local, offline, in-process-adjacent subprocess call with no network in the decision path. No benchmark figures exist. State architectural suitability; do not claim HFT capability.

`[U]` **40. Governance does not create edge.** EchoGate bounds risk; it does not generate returns. A strategy operating entirely within policy can be unprofitable. Cost realism — spread, slippage, financing, carry — is validated outside the governance layer and must never be inferred from an admitting verdict. The Neptune corpus illustrates the point sharply: its single ADMIT in 1,394 assays was a USD/TRY donchian breakout assayed at `fee_bps: 1.0, slippage_bps: 1.0` — roughly 2bp per side on an illiquid exotic where realistic cost is 15–25bp — with pooled kurtosis 188.25 and skew +10.25, meaning a handful of tail events carried the entire result. The governance machinery worked perfectly and still surfaced a candidate that cost realism would likely destroy. **An ADMIT is a permission, never a prediction.**

`[X]` **41. The `DEFER` verdict** — see §19. Resolved: `DEFER` is a real, supported EchoGate capability; it was simply never wired into Keel's live policy, because Keel's specific cage had no need for it. Any claim scoped to Keel should say "three verdicts" (ADMIT/REJECT/QUARANTINE); any claim about EchoGate as a general-purpose governor may note that a deferral/hold-for-review outcome is a supported, optional capability not currently exercised in this deployment.

---

# PART VI — THE DOCTRINAL FINDING

`[P]` Across every defect in this record, one signature repeats: **two independently correct mechanisms colliding at their boundary.**

- A resolver that regenerated the drift it was clearing (E5)
- A close-swallow for broker-managed exits that silently disabled a self-managed stop (§32.3)
- An exit rule written for reversion that forbade the exit architecture of momentum (§32.5)
- FIFO sizing coupled to the same field as governance identity (§32.6)
- A console reading the simulation policy while the governor enforced the live one (E4)
- A test suite writing into live runtime state (§31)

In **every** observed case, the system failed closed at the collision. Not one produced a malformed order. Not one moved capital. The conservative outcome was the automatic one each time — and in the invgrid case, the governor was right about a strategy it had no capacity to understand.

`[V]` That is the whole claim, and it is the only one worth making:

> **Intelligence without governance is unsafe. Complexity without replay is untrustworthy. Contradiction should be examined, not erased. Autonomy must be lawful, inspectable, and exact if it is to be trusted in the real world.**

---

# APPENDIX A — CLAIM LANGUAGE

**Safe to state:**

- "EchoGate evaluates permission, never merit; it is never given the inputs required to form an opinion on whether a trade is good."
- "A strategy and its mathematical inverse are governed by the same layer, with the capital cage unchanged."
- "The governor detected an incompatible exit architecture and refused every proposal before any order reached the venue, without knowledge of the strategy."
- "Fail-closed: a proposal type absent from the allow-list is refused before any numeric rule is evaluated. There is no default-permit path."
- "Exposure is bounded at five simultaneous scopes; concurrency across venues cannot exceed total risk."
- "Refusals are sealed and chain-linked exactly as approvals are."
- "One policy, one gate, two venues, two asset classes, concurrent — FX live, crypto in simulation."
- "Approximately 365 KB, offline, deterministic, with no network in the decision path."

**Do not state:**

- Any performance, return, Sharpe, or profitability claim attributable to Keel or to governance
- Live crypto execution
- HFT capability or any latency figure
- That the inverted grid's stop-out has been proven end-to-end
- That boot recovery is solved, or that §4d is an EchoGate limitation
- That EchoGate's glob matcher has been verified against the binary

---

# APPENDIX B — KEY IDENTIFIERS

| Item                    | Value                                                                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Governance binary       | `echogate/build/echogate`, ~365 KB                                                                                                                               |
| Live policy             | `policies/policy.json`, 1,043 lines, `policy_version: 1`, `strict_mode: true`                                                                                    |
| Policy hash in receipts | `keel_v0`                                                                                                                                                         |
| Agent package           | `keelbot/`, 5,091 lines, 17 modules                                                                                                                              |
| Engine                  | `keelbot/engine.py`, 1,798 lines                                                                                                                                 |
| Gate wrapper            | `keelbot/gate.py`, 440 lines                                                                                                                                     |
| The seam                | `engine.py:1386–1392`                                                                                                                                            |
| Live cage               | 75% / 35% / 25% / 12% · rung 5% · drawdown −2% · leverage 1×                                                                                                     |
| Live lanes              | EUR_USD (3 rungs), GBP_USD (2 rungs), USD_JPY disabled                                                                                                           |
| Mode                    | `live_small` — 20p spacing, 18p TP, FIFO-unique units                                                                                                            |
| Supervisor              | `keel_watchdog.sh` v2 — 5 restarts / 300s; code 75 exempt                                                                                                        |
| Test suite              | 125 tests                                                                                                                                                        |
| Key commits             | `9104eb5` E1 · `a042832` E2 · `272e0f5` E5 · `7487626`/`0cc8cc9` E3 · `c7cf906` E4 · `244f63c` test isolation · `a8bcfa7` invgrid · `ec2dcc2` invgrid governance |
| Neptune corpus          | 1,394 records · 1 ADMIT / 845 REJECT / 0 CONFLICTED / 548 INDETERMINATE                                                                                          |
| Reference receipt       | `OY_S20260808_0126`, schema `neptune.receipt.v1`, `T=0.9541` vs `t_min=0.95`                                                                                     |

**— END —**

---

*Note on this copy: this is the unmodified source document, including its own internal evidence-grading review and self-corrections (notably §19/§41 on the `DEFER` verdict). It is published verbatim, including the correction, because the discipline of grading and correcting claims is itself part of what this record is demonstrating. A later, dated addendum appended to §19 and §41 above resolves the `DEFER` point for public reference: it is a real, supported EchoGate capability not currently wired into Keel's policy, not a factual error.*
