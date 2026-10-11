# Adaptive Multi-Mechanism Trading System — Design Specification, Version 9

*A funding-oriented opportunity-selection architecture, built on the WMR benchmark-fix research programme.*

As of 1 October 2026.

---

## What changed in v9, and why

v9 keeps the v8 architecture intact. It adds two Phase 1 requirements that decide whether the first decisive test can be trusted, and three smaller changes that make the plan more honest about where profit has to come from.

| Change | Where | Status |
| --- | --- | --- |
| Measure real execution cost in the fix window before calibration | §15a, Phase 1 | Required before Phase 1 |
| Decide the fix-window data source and licensing before calibration | §15b, Phase 1 | Required before Phase 1 |
| State how deployability depends on the size of the WMR edge (IC sensitivity) | §14, L1 | Documentation fix |
| Treat externally sourced strategies as hypotheses only (new Step 0) | §9 | In force before any external idea is used |
| Re-rank candidates: trend + carry as the paired next tenants; session-boundary deferred | Part VI | Ordering only; no build before Mechanism 1 passes |
| v9.1 design decisions of 2026-10-05: standards, θ at the 50th percentile, cost percentiles, `required_ic`, false-acceptance cap, profit factor, worst-month flag, UNCONFIRMED sizing, constant classification | Appendix B.12–B.14; `docs/v9_1_design_decisions.md` | Decided 2026-10-05 |

Nothing in the gate arithmetic, the standards, the sizing rules or the governance changes from v8. (v9.1, 2026-10-05, does change several of these; see the last row of this table and Appendix B.14.)

### Context carried from v8

A research review rated v6 at 84/100 on producing a correct, defensible answer, and estimated roughly a 25% chance that answer would be profitable. Six gate-arithmetic faults it found were fixed in v8 and remain fixed. A separate review scored the spec 78/100 for funding capability: breadth, not rigour, is the binding constraint. v8 rebuilt the architecture around opportunity breadth. The WMR benchmark-fix mechanism is Mechanism 1, one tenant in a pool, not the business.

### The honest framing

"Unreasonable profit" is not a property any single mechanism can have. Information coefficients of 2–5%, Sharpe ratios of 0.5–1.0 and a few hundred independent bets a year describe what a genuine systematic edge looks like. A spec promising more is describing a bug, a fluke or a fraud. What is achievable is several real, modest, largely uncorrelated edges, sized properly and compounded for years. No version of this spec can guarantee a profit; its job is to maximise the odds that what is deployed is real, and to find out cheaply when it is not.

---

## Part I — Governing philosophy

### 1. The objective function

The objective is not this year's return, and not any single strategy's Sharpe. It is:

```
maximise   long-run geometric growth rate of capital
subject to P(ruin) <= stated bound
```

This is the Kelly framework, the precise translation of "maximise profit, with risk managed at my discretion." It has a real optimum: leverage beyond it reduces long-run growth, because volatility drag compounds against you. Part VII makes this concrete. The spec aims at the aggressive side of that curve without crossing into the reckless side.

### 2. Ten principles

1. **Economic value first.** A component is adopted because it demonstrably improves net, out-of-sample, cost-adjusted performance — never because it is sophisticated, fashionable or used elsewhere.
2. **No permanent edge.** A relationship is a current conditional estimate, not a law. The system estimates P(R_{t+h} | X_t, S_t) and re-estimates it continuously.
3. **Engines calculate. Models predict. Agents investigate.** Deterministic engines compute measurable quantities; a statistical or ML layer turns them into a return distribution; an agentic layer researches and monitors and never places a trade.
4. **Breadth matters as much as edge.** A real relationship with too few independent occurrences cannot fund anything. The system needs a pool of validated opportunities.
5. **No trade is the default, expected, frequent output.** Manufacturing trades to hit an activity target is the road to the failure mode this project has already lived through.
6. **Complexity earns its place by ablation, every time.** Every engine, feature and model is tested against the simplest working baseline.
7. **A mechanism must name its loser.** Every opportunity source states who is on the other side and why they cannot simply stop. "The model finds patterns" is not a mechanism.
8. **Thresholds are formulas in measured constants, never assertions.** This is the single most effective anti-fraud device available.
9. **Every gate carries a false-rejection and a false-acceptance rate, computed before it is frozen.** A gate a true edge cannot pass is as useless as one a false edge cannot fail.
10. **The final test is live economics.** Does it produce robust net returns under real conditions, and keep doing so once deployed?

---

## Part II — System architecture

### 3. The full pipeline

Market data feeds two parallel branches: state estimation (regime, volatility, liquidity, correlation) and event detection (fix windows, macro releases, calendar). These drive the trend, macro/carry and correlation engines and one engine per admitted mechanism. Their outputs form the state vector X_t, S_t, which the forecasting layer turns into a return distribution. A cost model converts that into net expected value per opportunity. Every live candidate enters the opportunity pool, which portfolio construction sizes and the execution engine trades. Monitoring and the agentic research layer feed back into the engines and gates.

```
                              MARKET DATA
                                   |
                 +-----------------+-----------------+
                 |                                   |
          STATE ESTIMATION                    EVENT DETECTION
                 |                                   |
     +-----------+-----------+                       |
     |           |           |                       |
   TREND    MACRO/CARRY  CORRELATION         MECHANISM ENGINES
   ENGINE     ENGINE       ENGINE            (Mechanism 1: WMR)
     |           |           |                       |
     +-----------+-----+-----+-----------------------+
                       |
              STATE VECTOR  X_t, S_t
                       |
                FORECASTING LAYER
                       |
            COST / EXECUTION MODEL  <----  EXECUTION-COST ENGINE
                 -> net EV                 (new in v9, §15a)
                       |
                OPPORTUNITY POOL
                       |
         PORTFOLIO CONSTRUCTION AND SIZING
                       |
        EXECUTION ENGINE -> LIVE POSITIONS
                       |
           +-----------+-----------+
           |                       |
   MONITORING / DRIFT      AGENTIC RESEARCH
           |                       |
           +---- feed back into engines and gates
```

The execution-cost engine is the only new component in v9; it feeds measured broker costs into the net-EV calculation.

Two things distinguish this from a conventional multi-strategy system. Every mechanism entering the pool passes the same falsification protocol (Part III); there is no informal admission path. And the forecasting layer is conditional on state S_t, so an engine's contribution can be zero in some regimes and positive in others.

### 4. The deterministic engines

Each computes a measurable quantity. None makes a trading decision.

| Engine | Computes | Feeds |
| --- | --- | --- |
| Market data / availability | Clean, timestamped, point-in-time-correct data; two-feed reconciliation where available (§15b) | Everything |
| Regime / state | Trend / mean-reversion / high-vol / low-vol probabilities via HMM or equivalent (§7) | S_t |
| Volatility | Realised and forward-looking volatility per instrument and window | Sizing, normalisation |
| Trend / momentum | Return over multiple horizons, trend strength | X_t; also the basis of Candidate 2 (Part VI) |
| Macro / carry | Rate differentials, risk-sentiment proxies | X_t; also the basis of Candidate 3 (Part VI) |
| Correlation / factor | Rolling cross-instrument correlation, dominant factor exposure (e.g. broad dollar) | Portfolio construction |
| Event / mechanism | One engine per admitted mechanism (Mechanism 1: WMR fix window) | X_t, opportunity generation |
| Execution cost (new in v9) | Quoted and filled spread, slippage and fill rate per instrument and time of day, from the live broker (§15a) | Cost model, capacity monitoring (§20) |

A single mechanism engine can only report "opportunity" or "no opportunity" for its own condition. Layering several engines into one state vector, and letting the forecasting layer learn which matter in which regime, turns one thin edge into a system that finds several. This is the architectural answer to the breadth problem.

### 5. The forecasting layer

The central estimate is conditional, not fixed:

```
P(R_{t+h} > 0 | X_t, S_t),   E[R_{t+h} | X_t, S_t],   Var(R_{t+h} | X_t, S_t)
```

Model choice is itself subject to ablation. Each rung must beat the one before it out of sample before it is retained:

1. The mechanism's own deterministic rule (Mechanism 1's rule has zero learned parameters and is the baseline).
2. Logistic / linear regression on engine outputs.
3. Gradient-boosted trees, once more than a handful of engines feed the state vector.
4. Regime-conditional variants of either, fit or blended separately per S_t.
5. Neural or sequence models, only if 1–4 are exhausted and a specific, named reason exists to expect them to help.

Every rung carries the Part III discipline: purged cross-validation, effective sample size, a measured floor a true edge must clear, and an audit trigger for results that look too good.

### 6. The opportunity pool and portfolio construction

At any timestamp, the pool holds every instrument/mechanism pair clearing its own net-EV floor. Portfolio construction then decides allocation given how correlated those opportunities are:

```
w_i  ∝  fractional_Kelly( E[R_i], Var(R_i) )
```

subject to a single-position cap, a single-factor cap (e.g. aggregate broad-dollar exposure), a portfolio volatility target and the drawdown ladder (§18). Correlated opportunities are one bet, not several: three pairs' WMR signals firing on the same fix, in the same direction, from the same month-end flow, are a single factor exposure. §10 gives the adjustment.

### 7. Regime estimation and continuous re-validation

A Hidden Markov Model or comparable estimator is a candidate engine, admitted only if it improves out-of-sample net performance. If admitted, its regime probabilities become part of S_t.

Every mechanism in the pool is re-validated on a rolling basis against its own admission criteria. One that stops clearing its floor is demoted to reduced size, then removed. "No permanent edge" is enforced structurally.

### 8. The agentic layer

Scope is deliberately bounded, and the boundary is load-bearing.

| Permitted | Never permitted |
| --- | --- |
| Generating hypothesis candidates for Part III | Placing a trade or sizing a position |
| Designing and running ablation experiments | Overriding a gate result |
| Investigating data-quality anomalies | Acting on an unlogged, unregistered experiment |
| Comparing candidate models within the §5 ladder | Deciding, in natural language, "buy EUR/USD" |
| Flagging deterioration in an admitted mechanism | Bypassing the falsification budget |
| Recommending retraining or retirement | Importing an external strategy past Step 0 (§9) |

An opaque "look at this pair and decide" layer reintroduces the unauditable discretion this project exists to remove. Everything the agentic layer produces is logged to the trial registry and ablated like any other component.

---

## Part III — The mechanism admission protocol

The protocol is defined once and applied to every candidate, WMR included. No mechanism enters the opportunity pool without clearing it.

### 9. The protocol

**Step 0 — Provenance (new in v9).** An externally sourced strategy is a hypothesis, not evidence. Sources include open-source repositories, curated lists, books, blogs, vendors and agentic-layer suggestions. It enters at Step 1 like anything else. Its published backtest, win rate or accuracy figure is given zero weight. Every variant pulled from an external source is logged as a trial and counts toward the pool-level Deflated Sharpe (§26).

**Step 1 — Hypothesis and mechanism.** State the claim, the market condition it fires on, and — non-negotiably — who is on the other side and why they cannot simply stop. Generic technical-indicator rules (RSI, MACD, Bollinger Bands and similar) fail here unless a loser is named.

**Step 2 — Calibration, not assertion.** Measure volatility, sample size and the cost distribution before any threshold is written. Cost is measured from live broker quotes where possible (§15a), not assumed.

**Step 3 — Gate instantiation.** Every gate is a formula in measured constants and the closed list of chosen numbers (Appendix B.14, decisions document section 2; the standards themselves are in §13). Only those numbers are human-chosen; every other threshold is a standard convention or is calculated, and no other numeric threshold is asserted directly.

**Step 4 — Power and false-acceptance check.** Each gate's ability to pass a true edge and fail a false one is computed before freezing: at least 75% power, capped false-acceptance.

**Step 5 — The single decisive test.** One estimation produces the economic result, the mechanism test, and a placebo test against the most plausible generic explanation. All three are terminal if failed.

**Step 6 — Holdout confirmation, not re-discovery.** A reserved sample checks the effect persists. It is not asked to re-establish significance, because it is usually too small to.

**Step 7 — Incremental value against the existing pool.** The candidate must add something beyond what the pool already holds. Three engines that each rediscover the same broad-dollar signal are not three sources of edge.

**Step 8 — Falsification budget.** Three revisions per mechanism. A revision after a decisive negative is free; after an ambiguous result it costs one. At zero, the mechanism is retired.

**Note 9a — Cross-validation embargo (amended 2026-10-01, Stage 1 / S1.1).** Purged cross-validation (`validation/purged_kfold.py`) applies a **two-sided embargo by default**: besides purging overlapping label windows and embargoing training samples that start within h after the test fold, it drops training samples whose label window ends within h before the test fold. The forward-only embargo remains available (`two_sided=False`) but is not the default. **Width:** h = max(1% of the time axis, measured dependence horizon, label horizon, longest feature lookback). The 1% figure is a floor only. The label horizon and longest feature lookback are required splitter inputs and form a **structural floor** that the purge check enforces with no opt-out. The **measured dependence horizon** (`measure_dependence_horizon`) is the maximum, over returns, |returns|, the label series and every feature series (training rows only), of `ar1_dependence_horizon`: the larger of the lag implied by a one-sided 99% upper confidence bound on a bias-corrected AR(1) coefficient and the first lag after which five consecutive autocorrelations sit inside the ±2/√n band. The purge check also enforces h ≥ this measured horizon. A point estimate (first lag inside the band) was rejected: the sample autocorrelation of a persistent series is biased toward zero, and it fell below the theoretical horizon on 87.5% of seeds (AR(1) φ = 0.95, n = 301). **Limitation — this is not a general bound on dependence.** `ar1_dependence_horizon` is conservative only for AR(1)-like, short-gap dependence. It undersizes dependence that returns after a gap longer than five lags and dependence carried by volatility rather than by the level: in the S1.1 review's simulations a 24-bar seasonal series (true horizon ≈ 96 bars) gave a median of 2, and volatility clustering gave a median of 6. Measuring |returns| catches part of the volatility case, not all of it. Both limitations are pinned by tests. A full-lag-range scan with a multiple-testing allowance across returns, |returns|, label and feature series, calibrated on real FX data, is a named Stage 2 task and a hard prerequisite before any model-selection result is trusted (`project_state.md`, "Before trusting model selection"). **Evidence** (`reports/s1_1_leak_injection.json`, experiment 2: AR(1) returns φ = 0.95, label window 2 bars, 1000 samples, 5 folds, 1-nearest-neighbour on a time-identifying feature, 200 seeds, chance = 0.5, theoretical dependence horizon 54 bars): structural floor only (h = 2, forward) 0.555 (+10.7 SE above chance); forward-only h = 16: 0.544 (+8.3 SE); two-sided h = 16: 0.523 (+3.9 SE); 1% floor (h = 11, two-sided): 0.530 (+4.9 SE); forward-only h = 54: 0.533 (+5.8 SE); two-sided h = 54: 0.504 (+0.7 SE); two-sided with h from `measure_dependence_horizon` (mean 113, range 52–217 bars; 1 of 200 seeds below the theoretical horizon): 0.495 (−0.9 SE). **What this shows:** at equal width, adding the backward half lowers accuracy by 0.021 ± 0.005 at h = 16 and 0.029 ± 0.005 at h = 54 (paired); at the theoretical width, forward-only still leaks (+5.8 SE) while two-sided is at chance; widths below the dependence horizon leak in both modes. **What it does not show:** the process is AR(1), exactly the case the estimator is built for, and the model is deliberately leak-exploiting. The results establish the mechanism, not the embargo needed for real FX data, and they say nothing about seasonal or volatility-driven dependence. The measured h is conservative on this process (mean about twice the theoretical horizon), which costs training data.

### 10. Portfolio-level correlation treatment

When several admitted mechanisms or instruments fire at once, effective breadth is not the simple count. For n similarly sized, equicorrelated opportunities with average pairwise correlation ρ:

```
n_eff = n / (1 + (n − 1)·ρ)
```

ρ is measured from realised co-occurrence, never assumed. This is why three pairs running one mechanism do not triple the diversification benefit (§14).

---

## Part IV — Mechanism 1: WMR benchmark-fix reversion

This is the prior programme, corrected against the faults the research review found and reduced to what the mechanism requires. The formulas and definitions are in Appendix B, copied verbatim from the v6 specification; open conflicts with this Part are listed in B.13.

### 11. The claim and mechanism

Price displaced during the WMR fix window (15:57:30–16:02:30 London) by mandatory, price-insensitive hedging flow partially reverts in the following hour. The participants generating that flow — pension and index funds hedging at a contractually specified benchmark rate — cannot defer, split or optimise execution without breaching their mandate. They are complying, not erring, which is why the effect can persist.

Two tiers of flow exist: month-end rebalancing (~18 events a year, strong, signed by equity returns) and continuous benchmark-referenced flow (~130 events a year, weaker). The hypothesis stands or falls on the weaker tier, since the stronger tier alone lacks the breadth to test. This is a terminal condition.

### 12. What was fixed (v8, unchanged)

| Prior fault | Fix in force |
| --- | --- |
| Holdout definition and instantiation disagreed, leaving a deployment gate nearly vacuous | Holdout fixed at 30% of the window; every formula uses the same count |
| Three deployment gates implied three different Sharpe standards | One research standard and one operational standard (§13); every gate checked against both before freezing |
| Two low-power remedies could trigger each other indefinitely | One remedy, applied once, before any result is seen; if power is still short, the programme says so and stops |
| Ten pre-validation modules for a zero-parameter rule | Pre-decision build is two components: data layer and signal/estimation layer |
| Calibration constants measured on data including the holdout | All calibration restricted to training and validation |

### 13. The two standards

- **Research standard:** 0.60 annualised Sharpe, single instrument — the bar for "this mechanism is real and worth extending."
- **Operational standard:** 0.75+ annualised Sharpe, portfolio level — the bar for committing meaningful capital, evaluated only after breadth is added.

A once-daily event on one instrument, even at the top of published systematic-FX skill, tops out near an information ratio of 0.6. A higher research standard would be unreachable by construction.

### 14. Breadth: the corrected arithmetic, with IC sensitivity

Mechanism 1 runs across GBP/USD, EUR/USD and USD/JPY from day one: the same benchmark fix applies to all three. Because the same fix drives all three on the same day, the §10 adjustment applies. At pairwise correlation 0.2–0.5, effective annual breadth is about 227–324.

v8 showed the resulting information ratio only at IC = 0.05, the top of the plausible range. v9 shows the full sensitivity, using IR = IC × √breadth:

| IC | ρ = 0.5 (breadth ~227) | ρ = 0.3 (breadth ~284) | ρ = 0.2 (breadth ~324) |
| --- | --- | --- | --- |
| 0.03 | 0.45 | 0.51 | 0.54 |
| 0.04 | 0.60 | 0.67 | 0.72 |
| 0.05 | 0.75 | 0.84 | 0.90 |

> **v9.1 note (2026-10-05):** the breadth figures above assume about 148 qualifying events per pair per year (θ at the 40th percentile). With θ at the 50th percentile (v9.1, 2026-10-05) it is about 126, so the required IC is higher. S3.0 recomputes from measured values.

**What this means.** Three-pair WMR clears the 0.75 operational standard only if IC is about 0.042–0.050, depending on ρ. The single-pair research standard (0.60 at ~148 events a year) already needs IC of about 0.049. Mechanism 1 can pass only if its edge sits at the upper end of the published range. That is acceptable, but it is the main reason a second, less-correlated mechanism (Part VI) is a requirement for the business, not an extra.

When Phase 2 reports a measured IC and ρ, this table is the first thing consulted: it says immediately whether the result is deployable alone or only as one tenant.

### 15. Gates, compressed

The formula set carries over in substance (verbatim text in Appendix B; unresolved differences in B.13):

- Mean net return in volatility-normalised units, tested against a floor derived from the research standard.
- A mechanism test requiring the effect to scale with displacement size and concentrate at month-end.
- A placebo test requiring the effect at the fix to be stronger than at three control times with no benchmark.
- A Deflated Sharpe requirement adjusted for every configuration ever tried.

Every threshold is a formula in measured volatility, kurtosis, cost and sample size. The decisive result arrives about three weeks after data work begins, across all three pairs at once.

### 15a. Execution cost measurement (new in v9 — required before calibration)

The expected edge per trade is a few basis points. Spreads widen around 16:00 London, and retail or prime-of-prime spreads in that window can be the same size as the edge. A historical cost model cannot settle this; the actual broker's quotes can.

1. From the first day of Phase 1, record the intended broker's streaming quotes for all three pairs from 15:50 to 17:10 London, every trading day.
2. Capture at least one month-end fix (the strong-tier event, where spreads are widest). If Phase 1 ends before month-end, cost capture continues until it is covered.
3. Where the account allows, place minimum-size test orders at the intended entry and exit times to measure fill price against quote.
4. Express round-trip cost in the same volatility-normalised units as the gates. This becomes a measured constant in Step 2.
5. **Early feasibility check.** From the measured cost, compute the IC needed for the net floor to be clearable. If that IC exceeds 0.05 (the top of the published range), Mechanism 1 is flagged as likely infeasible at this broker before Phase 2 is run. The options are then a different execution venue, or moving effort to the Part VI candidates.

### 15b. Fix-window data (new in v9 — decision required before calibration)

Every gate is calibrated on the fix window. Timestamps that are wrong by a minute or two calibrate the gates on the wrong window, and nothing downstream can detect it.

- **Resolution:** tick or one-second quotes for 15:50–17:10 London, timestamps in UTC to at least the second.
- **Clock handling:** the fix is at 16:00 London local time. Every series is converted with explicit UK daylight-saving rules; a fixed UTC offset is a known silent failure.
- **Two feeds:** reconcile two independent sources in the window and log disagreements above a stated tolerance (the §27 rule applies: the check must first be shown to fail on a deliberately shifted series).
- **Benchmark prints:** the official WMR fix rates are licensed from LSEG. With them, displacement is measured against the actual benchmark. Without them, a feed-derived 16:00 mid is used as a proxy, and the proxy error is measured and stated in L6.
- **Free sources** are acceptable for exploration but not for frozen calibration unless they pass the two-feed check.

**Decision needed:** license the WMR prints, or accept the proxy with its measured error.

---

## Part V — Capital deployment and compounding

This is where aggressive growth is actually decided.

### 16. Position sizing

Fractional Kelly, not fixed-fraction volatility targeting. Full Kelly is optimal only if the true edge is known exactly, and it never is: every Sharpe here is a sample estimate with a real confidence interval. Full Kelly on an estimate overshoots the true optimum and is a fast route to a blow-up-shaped drawdown. Sizing runs at one-quarter to one-half Kelly, scaled down further while a mechanism's live record is short. The Part III effective-sample-size logic sets how much of the estimated edge is trusted with real size.

### 17. Staged deployment

1. Gate pass → small live pilot, capped at 5–10% of eventual intended size.
2. Size increases as live sample accumulates and the live-vs-backtest gap is confirmed within tolerance. The §15a cost record is the baseline for that gap.
3. Full fractional-Kelly sizing once live evidence, not just backtest evidence, supports it.

Live economics is the final test (Principle 10). A long paper-trading phase delays the only evidence that matters while adding little a well-specified backtest did not already give. The capped pilot bounds the downside.

### 18. The drawdown ladder

| Drawdown from peak | Action |
| --- | --- |
| 0–12% | Full sizing |
| 12–20% | Half of target sizing |
| 20–30% | Quarter sizing; flag for manual review |
| > 30% | Flatten; explicit re-authorisation required before resuming |

### 19. Reinvestment policy

Profits compound back into the sizing base by default, which is what makes the Part VII figures real. A stated capital floor — a sum never put at risk however well the system performs — is withdrawn from the base up front and held aside. This trades a little long-run growth for a bound on the absolute worst case.

### 20. Capacity

A benchmark-fix reversion trades a five-to-ten-minute liquidity window. As capital grows, the position starts moving the price it trades, and historical costs stop describing reality. Every mechanism has a ceiling on the capital it can absorb before its own edge degrades. That is why breadth — more mechanisms, not more size on one — is the right lever once a strategy is capacity-constrained. Realised versus modelled slippage is tracked per mechanism by the execution-cost engine, and size is capped the moment they diverge.

---

## Part VI — Candidate mechanisms (re-ranked in v9)

Trend and carry are now the next two tenants, developed as a pair; session-boundary liquidity provision is deferred to fourth. None is built before Mechanism 1 clears its gates. The re-ranking reflects §14: Mechanism 1 alone clears the operational standard only at the top of the plausible IC range, so the next mechanism must be the best-evidenced, highest-capacity and least-correlated source available.

| Rank | Candidate | Timescale | Driver | Evidence base |
| --- | --- | --- | --- | --- |
| 2 | Time-series trend | Weeks to months | Slow-moving information and flow | Moskowitz, Ooi & Pedersen (2012) |
| 3 | Rate-differential carry | Weeks to months | Policy-rate differentials; crash-risk premium | Lustig & Verdelhan; Menkhoff, Sarno, Schmeling & Schrimpf (2012) |
| 4 (deferred) | Session-boundary liquidity provision | Intraday | Participant rotation across sessions | Widely known pattern; weakest evidence |

### 21. Candidate 2 — Time-series trend

**Claim:** an instrument's own past return over weeks to months predicts its future return over a similar horizon.

**Mechanism and loser:** to be stated precisely at Step 1. Candidate explanations are under-reaction to information and slow-moving institutional flows (hedging programmes, rebalancing, central-bank paths) that cannot be executed at once.

**Why it suits this pool:** it runs on a different timescale and driver from the WMR fix. It is high-capacity, which addresses §20. Historically it has tended to profit in sustained risk-off moves, which is where carry suffers — this complementarity is a hypothesis to be measured at Step 7, not assumed.

**Universe:** trend draws its breadth from many instruments. It is specified on a wider universe (a G10 FX cross-section, or liquid futures) rather than the three Mechanism 1 pairs; the exact universe is chosen at Step 2 from cost and data availability.

**Known risk:** whipsaw in range-bound regimes; crowding in a widely traded style.

### 22. Candidate 3 — Rate-differential carry, risk-managed

**Claim:** higher-yielding currencies have historically drifted upward against uncovered interest parity over multi-week to multi-month horizons.

**Mechanism:** the most-studied anomaly in FX, with a plausible risk-based explanation (compensation for crash risk). It may be a risk premium rather than an inefficiency. That changes how it is sized — never leveraged as if it were free — but does not disqualify it.

**Known risk:** negative skew: it grinds up, then crashes in risk-off episodes. The carry sleeve is sized for that shape, not averaged into ordinary volatility targeting.

**Pairing with trend:** carry and trend are admitted separately, each through the full protocol. They are designed and evaluated together because their skews may offset. Their joint value is tested at Step 7 against the realised co-occurrence of their drawdowns.

### 22a. Candidate 4 — Session-boundary liquidity provision (deferred)

**Claim:** volatility and directional persistence differ across the Tokyo / London / New York rotation, exploitable by conditioning size and holding period on the active session.

**Mechanism:** different participant populations dominate different hours with different objectives. It is uncorrelated with the WMR fix by construction.

**Why deferred:** it is the most widely known intraday FX pattern, and the Step 5 placebo test is likely to find a generic effect. It stays on the list as a later candidate.

### Sequencing

No candidate is built while Mechanism 1 is unproven. Adding breadth to rescue a weak result is a failure this project has already named and rejected. If Mechanism 1 fails decisively, trend and carry move to the front and enter the protocol directly.

---

## Part VII — The honest growth model

### 23. What a real portfolio Sharpe compounds into

Using the simplified relationship between Sharpe S, volatility target σ and long-run geometric growth g, maximised at full Kelly when σ = S:

```
g ≈ S·σ − σ²/2
```

| Portfolio Sharpe | Quarter-Kelly: vol target → growth/yr | Half-Kelly: vol target → growth/yr |
| --- | --- | --- |
| 0.75 (three-pair WMR at the top of the IC range, §14) | 18.8% → 12.3% | 37.5% → 21.1% |
| 1.00 (WMR + trend/carry sleeve) | 25.0% → 21.9% | 50.0% → 37.5% |
| 1.30 (a genuinely well-diversified pool) | 32.5% → 37.0% | 65.0% → 63.4% |

Full Kelly is not shown as a target. At Sharpe 1.0 it implies a 100% volatility target, which assumes the Sharpe estimate is exact. Given how uncertain these estimates are (Part III), quarter-Kelly is the realistic operating range and half-Kelly the upper edge of aggressive-but-deliberate.

### 24. What compounding produces

| Annual growth | 5 years | 10 years |
| --- | --- | --- |
| 12% | 1.76× | 3.11× |
| 15% | 2.01× | 4.05× |
| 20% | 2.49× | 6.19× |
| 25% | 3.05× | 9.31× |

This is where "unreasonable" lives: not in one year or one backtest, but in roughly nine times capital over a decade from a realistic portfolio Sharpe. It requires, in order: the edge must be real (Part III), the sizing must survive the volatility (§16–18), and it must run for years.

### 25. What these tables do not include

Real returns are fat-tailed: excess kurtosis of 9–11 has been measured on this kind of signal, so realised drawdowns will be worse than this model predicts at any volatility target. Costs, capacity limits (§20) and periods of mechanism demotion (§7) all reduce the figures. The first row also assumes the WMR edge sits at the top of its plausible range (§14). These are properties of a real, validated, properly sized return stream — not a forecast, and not a promise.

---

## Part VIII — Governance

### 26. Trial registry and falsification budget, at pool level

Every configuration ever evaluated, across every mechanism and every model in the §5 ladder, is logged — including abandoned ones, which are exactly the draws that make a survivor look better than it is. From v9 this explicitly includes every externally sourced strategy and every variant of it (Step 0), and every parameter sweep run in fast vectorised research tools. The falsification budget is tracked per mechanism. A portfolio-level Deflated Sharpe is computed against the total trial count across the pool: testing five mechanisms and reporting the best carries the same multiple-comparisons risk as testing five parameter sets.

### 27. Test integrity

Every test guarding an invariant — a risk cap, a data-quality assertion, a leakage check, the §15b two-feed and daylight-saving checks — is shown to fail against the violation it exists to catch before it is trusted to pass. A test that cannot fail is worse than no test, because it retires the concern it appears to address.

---

## Part IX — Standing limitations

**L1 — Cross-pair correlation and edge size are estimated, not known.** The 0.2–0.5 correlation band in §14 is plausible, not measured; the real figure is computed at G0 from realised co-occurrence. Deployability is also highly sensitive to IC: three-pair WMR clears the operational standard only at IC ≈ 0.042–0.050, and the single-pair research standard needs IC ≈ 0.049. Both are recomputed from measured values before any capital decision.

**L2 — Candidate mechanisms are unproven.** Trend, carry and session-boundary are named because a pool needs more than one tenant, not because they have cleared Part III. Until they do, the portfolio Sharpe this document can defend is three-pair WMR alone.

**L3 — Capacity is a real ceiling.** A mechanism that works at modest size can degrade once capital scales. This is the most common reason a good backtest and a disappointing live result coexist, and no upstream rigour fixes it.

**L4 — The agentic layer is a new failure surface.** Bounding it to research and monitoring (§8) removes the largest risk. Its outputs are logged and ablated, and it has no path to act unilaterally.

**L5 — The growth model is a property of a validated process, not a guarantee.** It answers "what does compounding a real edge look like," not "what will happen."

**L6 — Benchmark reference (new in v9).** If the licensed WMR prints are not used, displacement is measured against a feed-derived proxy. The proxy's measured error against the official fix is stated here once known.

**L7 — Execution venue dependence (new in v9).** Mechanism 1's net edge is specific to the broker and account it is measured on. A change of venue requires re-running §15a before sizing is carried over.

---

## Part X — Roadmap

The first capital decision arrives around week nine: a small live pilot on whichever pairs pass, sized per §17. v9 adds one earlier decision point — the cost feasibility check at the end of Phase 1 — so an unworkable execution venue is found in week 3, not week 9.

Decision points: **cost feasible?** (end of Phase 1, §15a) → **decisive test** (end of Phase 2, Step 5 and the §14 table) → **first capital** (end of Phase 3, about week 9).

1. **Phase 1 — Data, cost and calibration (3 weeks; cost capture runs to the first month-end).** Settle the §15b data decision; build the data layer with two-feed and daylight-saving checks; start §15a cost capture on day one; instantiate Mechanism 1 gates across three pairs. **Exit:** the §15a feasibility check.
2. **Phase 2 — Decisive result (3 weeks).** Run the Step 5 test across all three pairs; read the result against the §14 sensitivity table. If it passes, build signal, risk and execution components.
3. **Phase 3 — Holdout and pilot (3 weeks).** Holdout confirmation; staged live pilot on passing pairs.
4. **Phase 4 — Portfolio construction (2 weeks).** Kelly sizing and portfolio layer built around the live pilot.
5. **Phase 5 — Trend + carry admission (6–8 weeks).** Both enter the protocol in parallel with live Mechanism 1. If Mechanism 1 failed in Phase 2, this phase starts immediately instead.
6. **Phase 6 — Regime and agentic layers (ongoing).** Admitted only by ablation against the running pool.

---

## Appendix — External resources and how they are used

External resources supply tools and ideas, never evidence (Step 0). Curated lists such as [awesome-systematic-trading](https://github.com/wangzhe3224/awesome-systematic-trading) are useful mainly as a catalogue of infrastructure. Their strategy collections are mostly generic technical-indicator rules, which fail Step 1, and advertised win rates or accuracy figures carry no weight.

| Resource | Use in this system | Caution |
| --- | --- | --- |
| [pysystemtrade](https://github.com/robcarver17/pysystemtrade) | Reference design for Candidates 2 and 3: vol targeting, forecast scaling and combination across many instruments | Read for method; its parameters are not admitted without the protocol |
| [nautilus_trader](https://github.com/nautechsystems/nautilus_trader) | Event-driven backtest and live engine with near-identical code paths; narrows the §17 live-vs-backtest gap | Its fill model is not a substitute for §15a measured costs |
| [vectorbt](https://github.com/polakowo/vectorbt) | Fast research sweeps | Every sweep is logged as trials (§26); speed makes multiple-comparisons risk worse, not better |
| [finmarketpy](https://github.com/cuemacro/finmarketpy) | FX-focused analysis utilities for the data layer | Verify any timestamp conventions against §15b |

**Literature behind the mechanisms:** Melvin & Prins (2015) on equity-hedging flows at the London 4pm fix, and Krohn, Mueller & Whelan on intraday dollar patterns around fixings (Mechanism 1); Moskowitz, Ooi & Pedersen (2012) on time-series momentum (Candidate 2); Lustig & Verdelhan, and Menkhoff, Sarno, Schmeling & Schrimpf (2012) on carry (Candidate 3). Mechanism 1's papers are read before Phase 1 calibration, so gates are not calibrated against an effect size the literature already contradicts.

### Sources

- [wangzhe3224/awesome-systematic-trading](https://github.com/wangzhe3224/awesome-systematic-trading) — reviewed 1 October 2026

---

## Appendix B — Gate formulas and definitions

**Status (2026-10-01): copied; partly reconciled.** Human decisions of 2026-10-01 are recorded in B.13. Items marked **pending v9.1 design** are unspecified and block S1.2. This appendix is the authority for the WMR gate formulas and definitions. The text in each B.1–B.11 block is copied verbatim from the v6 specification (`docs/spec_v6.md`, line ranges given). Neither v8 (`docs/spec_v8_detailed.md`, a summary despite its name) nor the earlier body of this spec contains these formulas. v5 (`docs/spec_v5.md`) is not a source: v6 never defers to it. Precedence is v9 > v8 > v6 > v5, applied only where a later version explicitly changes something; each such change is noted under the excerpt it affects. Points where v8/v9 differ from v6 are listed in B.13 with the human's decision or their open status. S1.2 does not start until every item it needs is decided.

Reading rules: blocks between `verbatim:begin` and `verbatim:end` markers are byte-identical to the cited v6 lines, except that markdown headings are demoted two levels. Section references inside a block (§II.2, §18.3, §22, §17.5 and so on) are **v6 numbers**, not this document's. Notes after a block (marked **v9 change**, **v9 addition** or **Open**) are not v6 text.


### B.1 Signal, outcome, volatility estimators and selection

*Source: `docs/spec_v6.md` v6 §H1.3 (subsections Signal → Selection), lines 114–167.*

<!-- verbatim:begin B.1 -->
##### Signal

```
d      = (mid at post-window start − mid at pre-window start) / mid at pre-window start
d̃      = d / σ_pre(window)
```

##### Outcome

```
entry   = ask (if buying) or bid (if selling), first tick ≥ post-window start
exit    = bid (if long) or ask (if short), at post-window end
r_exec  = signed executable return
u       = r_exec / σ_post(window)          ← THE PRIMARY OUTCOME
```

##### Why the outcome is volatility-normalised

Position size is volatility-targeted: one σ_post move equals a fixed fraction of equity. Position size is therefore proportional to `1/σ_post`, and **realised equity return per trade is proportional to `r_exec / σ_post`, not to `r_exec`.**

Measuring returns in basis points while sizing on volatility measures a quantity that does not correspond to profit and loss unless σ_post is constant, which it is not. All gates, all regression coefficients and all reported economics are in σ-units. Basis-point figures are reported alongside for interpretability and are never gated.

##### Volatility estimators

```
σ_pre(w)   trailing 60-observation realised volatility of window w's PRE period
σ_post(w)  trailing 60-observation realised volatility of window w's POST period
```

**A 60-observation lookback is specified rather than a shorter one.** Each window yields one observation per trading day, so the lookback length *is* the observation count. The sampling coefficient of variation of a volatility estimate from *n* observations is `1/√(2n)`:

| Observations | Sampling CV |
|---|---|
| 20 | 15.8% |
| **60** | **9.1%** |
| 120 | 6.5% |

Noise in this estimator sits in the denominator of both `d̃` and `u`, inflating `std(u)` and therefore widening every downstream confidence interval. Sixty observations balances that against the lag a longer window introduces. **The lookback is fixed a priori and is never swept.**

**Method and lookback are frozen on the training partition. Values are computed on rolling trailing data at every date, including within the holdout.** A trailing estimate uses only prior prices and is available at any date.

##### Selection

```
QUALIFYING EVENT: |d̃| ≥ θ
    θ = 60th percentile of |d̃| on the FIX window, training partition only,
    applied AS THE SAME NUMERIC VALUE to all four windows
```

**θ applies identically across all four windows.** If the fix window were selected to its top fraction by displacement while control windows used all observations, the placebo comparison would contrast selected extremes against unselected data. Mean reversion is stronger after large moves, so the control offsets would be biased and F9 would appear to pass regardless of whether the effect is fix-specific.

**Qualifying counts per window are reported at G0.** A fix window with materially more qualifying events is itself weak evidence of additional flow. If any window falls below the F2 minimum, F9 is designated underpowered per F2 rather than treated as passing.

**θ drift.** The realised qualifying rate per calendar year is reported at G0. If the rate varies across years by more than 15 percentage points, **σ_pre's lookback is extended to 120 observations and θ recomputed once, before any outcome is examined.** This is a single pre-specified remedy applied before measurement, not a discretionary adjustment.
<!-- verbatim:end B.1 -->

> **Open:** θ percentile and the θ-drift remedy — see B.13 C7. Measurement-window timestamps (v6 §H1.3, lines 101–112 of `docs/spec_v6.md`) are not copied here; they differ from the v9 fix window — see B.13 C14.

> **Superseded where B.14 differs** (θ 50th percentile; single θ-drift remedy; audit trigger target + Φ⁻¹(0.95)·SE replaces 3 × target). See `docs/v9_1_design_decisions.md`.


### B.2 The unified estimation

*Source: `docs/spec_v6.md` v6 §H1.5b, lines 235–264.*

<!-- verbatim:begin B.2 -->
#### H1.5b — The Unified Estimation

All mechanism conditions are coefficients of one pooled regression, on volatility-normalised outcomes, across all four windows, on qualifying events only.

```
u = α
  + Σ_w α_w·I_w                    window main effects
  + λ·month_end                    month-end main effect
  + μ·(month_end × fix)            month-end level effect at the fix
  + β·d̃                            reference slope: fix window, non-month-end
  + Σ_w δ_w·(d̃ × I_w)              control-window slope offsets
  + γ·(d̃ × month_end × fix)        month-end slope increment at the fix
  + ε
```

**All lower-order terms of every interaction are included.** `γ` is a three-way interaction, so `month_end` and `month_end × fix` appear as main effects; without them `γ` would absorb level differences between month-end and other days and would be biased.

**Standard errors are clustered by date**, since the four windows on a given date are not independent observations. For the fix-window-only economic test, where each date contributes one observation, **standard errors are additionally computed with a Newey–West correction** at a lag length of `⌈N_G1^(1/4)⌉`, and the larger of the two is reported.

| Quantity | Interpretation | Prediction | Resolves |
|---|---|---|---|
| **β** | Fix window, non-month-end slope = **Tier 2** | < 0, CI excludes zero | **F6** |
| **β + γ** | Fix window, month-end slope = **Tier 1** | more negative than β | — |
| **γ** | Month-end slope increment, fix window only | < 0, CI excludes zero | **F5** |
| **δ̄ = mean(δ_w)** | Average control-window slope offset | > 0, CI excludes zero | **F9** |
| mean(`−sign(d)·u`) − costs | Economic edge in σ-units | Per §II.2 | **F1** |

**F9 tests the mean offset, not each component.** Requiring all three control offsets to be individually positive costs between 14% and 49% of the test's power depending on the effect size, for no additional protection: the mean offset already captures the claim that the fix differs from ordinary times of day. **Individual signs are reported as a diagnostic**, and any negative component is flagged in the G1 report without being terminal.

AUC, hit rate and calibration are **diagnostics**, never gated.
<!-- verbatim:end B.2 -->


### B.3 Falsification conditions

*Source: `docs/spec_v6.md` v6 §H1.6, lines 266–298.*

<!-- verbatim:begin B.3 -->
#### H1.6 — Falsification Conditions

| # | Condition | Threshold | Consequence |
|---|---|---|---|
| **F1** | No economic edge | One-sided 95% CI on mean net `u` includes zero, **or** point estimate below the viability floor (§II.2) | **Terminal** |
| **F2** | Insufficient sample | Any gating test has effective n below the minimum in §II.2 and was not designated underpowered in advance with a point-estimate criterion | Terminal or designation |
| **F3** | Effect not graded | Spearman rank correlation between quintile index of `d̃` and quintile mean net `u` has a 95% CI including zero, **or** the top-minus-bottom quintile difference has a CI including zero | **Terminal** |
| **F4** | Not distinguishable from luck | Deflated Sharpe Ratio below the threshold in §II.2, out-of-sample | **Terminal** |
| **F5** | Mechanism — month-end | γ CI includes zero, **or** γ point estimate is positive | Unconfirmed |
| **F6** | **Core claim — Tier 2 reversion** | β CI includes zero, or β is not negative | **Terminal** |
| **F7** | Mechanism — equity link | Correlation between equity month-to-date return and **signed `d̃`** on month-end events has point estimate below the floor in §II.2 | Unconfirmed |
| **F8** | Measurement artefact | β under the shared-endpoint construction is more negative than β under the separated construction by more than its own 95% CI width, **and** the separated β fails F6 | **Terminal** |
| **F9** | Not specific to the fix | δ̄ CI includes zero, or δ̄ is not positive | **Terminal** |

##### The hierarchy is explicit

**F6 is the claim itself** — no Tier 2 reversion means nothing tradeable, since Tier 1 alone fails F2 on breadth. **F9 rules out the alternative explanation** — if reversion at control times is indistinguishable from the fix, the effect is generic short-horizon mean reversion and fix flow explains none of it. **F8 rules out the construction artefact.** Each is terminal.

**F5 and F7 are mechanism details.** Failing them means the effect may be real but unexplained, which warrants caution rather than termination.

##### F3 tests gradedness, not strict ordering

The concern F3 addresses is an effect driven by a handful of outliers rather than a graded relationship. **Requiring strictly increasing means across ten deciles does not test that** — with realistic noise, adjacent decile means are ordered essentially at random, and the probability that all nine comparisons fall in order is negligible even when the underlying effect is genuine and linear.

F3 therefore uses a rank correlation across five quintiles together with a top-minus-bottom difference, both with confidence intervals. This detects gradedness with adequate power while remaining sensitive to an outlier-driven result, which produces neither a rank correlation nor a spread between the extreme quintiles.

##### F7 uses signed displacement, not its sign

H1.2 states that the size of the hedge adjustment is proportional to the equity return. Correlating with signed `d̃` is faithful to that prediction and materially more powerful than a point-biserial correlation against a binary sign.

##### Underpowered designations

Any gating test whose effective sample falls below the F2 minimum is designated **underpowered in advance**, evaluated on a point estimate against a pre-specified floor, and **does not count toward any terminal condition.** An underpowered result is not evidence against the hypothesis; recording the distinction before measurement prevents an absent test being read as a negative one.
<!-- verbatim:end B.3 -->

> **Resolved (C8):** the count is F1–F9, as in v6. `§II.2` inside this excerpt means B.6.


### B.4 Event count and breadth (definitions of N_G1, N_yr)

*Source: `docs/spec_v6.md` v6 §H1.7 (table and N_G1 note), lines 300–312.*

<!-- verbatim:begin B.4 -->
#### H1.7 — Event Count and Breadth

| Quantity | Expected | Measured at |
|---|---|---|
| Test window | 2015-02 → present, ~11.5 years | — |
| Raw fix days | ~2,900 | G0 |
| After θ | ~1,160 | G0 |
| **`N_G1`** — qualifying events available to G1 (training + validation) | **~928** | **G0** |
| **`N_yr`** — qualifying events per year | **~101** | **G0** |
| Month-end subgroup (Tier 1) | ~138 | G0 |
| Control-window qualifying counts | Per window | G0 |

**`N_G1` excludes the holdout.** G1 runs before the architecture is frozen and therefore operates on training and validation only. Every power calculation in §II.3 uses `N_G1`, not the full sample.
<!-- verbatim:end B.4 -->

> **v9 change (final_spec §12, explicit):** the holdout is 30% of the window, not v6's 20% (B.11). `N_G1` (training + validation) is therefore measured on 70% of the window. The expected counts in this table are v6's and are not recomputed here.


### B.5 Measured constants

*Source: `docs/spec_v6.md` v6 §II.1, lines 394–408.*

<!-- verbatim:begin B.5 -->
#### §II.1 — Measured Constants

Every threshold in this specification is a function of the following, all measured at G0 before any gate is frozen.

| Symbol | Quantity |
|---|---|
| `s` | `std(u)` on qualifying fix-window events |
| `k` | `kurtosis(u)` |
| `N_G1` | Qualifying fix-window events in training + validation |
| `N_yr` | Qualifying fix-window events per year |
| `N_q` | Number of complete quarters in the holdout evaluation |
| `F_|u|(·)` | Empirical distribution of `|u|` |
| `F_spread(·)` | Empirical distribution of entry spread |

**`s` is measured, not assumed.** `u` is a return divided by a *trailing* volatility estimate, which is a forecast rather than the realised volatility of the observation it normalises. Estimator lag and estimator noise both inflate the ratio's dispersion, and standardised financial returns retain substantial excess kurtosis. Every interval, power figure and volatility-derived allowance in this specification scales with `s`.
<!-- verbatim:end B.5 -->

> **Resolved (C5):** `N_hold` is `N_q` counted over the holdout only — see B.12. **`S_research` is pending v9.1 design (C2/C3, reopened 2026-10-02).**


### B.6 Gate formulas

*Source: `docs/spec_v6.md` v6 §II.2, lines 410–445.*

<!-- verbatim:begin B.6 -->
#### §II.2 — Gate Formulas

```
SE            = s / √N_G1                       standard error of mean(u)
target        = 0.89 · s / √N_yr                mean(u) implying Sharpe 0.89
floor         = 0.60 · s / √N_yr                mean(u) implying Sharpe 0.60
annual_vol    = R · s · √N_yr                   R = per-trade equity risk, §II.4
```

| Gate | Formula | Rationale |
|---|---|---|
| **Economic floor (F1)** | mean net `u` ≥ `floor`, and one-sided 95% CI excludes zero | Separates *is it real* from *is it worth trading* |
| **Effective sample (F2)** | `n_eff ≥ 200` per gating test | Below this, intervals cannot separate edge from luck |
| **Deflated Sharpe (F4)** | > 0.5, or > 0.8 if the mechanism is unconfirmed | Adjusted for total trial count |
| **Equity-link floor (F7)** | Correlation ≥ +0.10 | Point estimate; CI reported |
| **Maximum spread** | `Q_spread = F_spread⁻¹(0.99)` | Skips pathological entries without discarding ordinary variation |
| **Disaster stop** | `Q_disaster = F_|u|⁻¹(0.999)` | Trigger rate known in advance; results reported with and without |
| **Profit factor** | ≥ PF corresponding to Sharpe 0.73, computed from the empirical distribution | Verified stable across both normal and fat-tailed distributions |
| **Maximum drawdown** | < `2.0 × annual_vol` | Scales with measured volatility rather than fixing a percentage |
| **Worst month** | Better than `−m × monthly_std`, where `m` is set so the expected number of breaches over the evaluation period is ≤ 0.20 under the measured kurtosis | A fixed multiple derived under normality binds roughly twice as often as intended when returns are fat-tailed |
| **Quarterly consistency** | Proportion profitable ≥ `p_floor − 1.04·√(p_floor(1−p_floor)/N_q)`, where `p_floor = Φ(0.60/2)` | Calibrated so false rejection at the *viability* Sharpe is ≤ 15% |

##### Worked instantiation

Illustrative only — the operative values are those produced at G0.

```
measured:  s = 1.30   k = 9.3   N_G1 = 928   N_yr = 101   N_q = 46

SE                = 1.30/√928        = 0.0427
target            = 0.89·1.30/√101   = 0.1151
floor             = 0.60·1.30/√101   = 0.0776
annual_vol        = 1.0%·1.30·√101   = 13.1%
max drawdown gate = 2.0 × 13.1%      = 26.2%
consistency gate  = 0.618 − 1.04(0.072) = 0.543 → 54%
```
<!-- verbatim:end B.6 -->

> **v9 change (final_spec §12, §13 and §9 Step 3, explicit):** gates are checked against one research standard (0.60) and one operational standard (0.75+), and each gate is "a formula in measured constants and one stated business input". v6's `target` (Sharpe 0.89) and profit-factor gate (Sharpe 0.73) are separate standards. **Pending v9.1 design (C2/C3, reopened 2026-10-02):** which standards `S_research` and `S_target` take is open, because v9 §13 states the 0.75 operational standard at portfolio level, while these gates are single-mechanism. The profit-factor standard is also pending v9.1 design (C9).

> **v9 change (final_spec §15a, explicit):** the primary cost constant is the broker-measured round-trip cost in volatility-normalised units, not only the per-event tick spread.

> **Resolved (C6, v9.1 decision D3):** `annual_vol` is evaluated at the fixed reference risk `R` = 1%. **Resolved (C5):** consistency is quarterly, over the holdout only. **Pending v9.1 design:** profit factor (C9), worst month (C10).


### B.7 Power requirement

*Source: `docs/spec_v6.md` v6 §II.3, lines 447–464.*

<!-- verbatim:begin B.7 -->
#### §II.3 — Power Requirement

**Every gate carries a false-rejection rate computed at the design target from measured constants, and no gate may exceed 25%.** A gate that a true hypothesis cannot pass is as useless as one a false hypothesis cannot fail.

Instantiated at the illustrative constants above:

| Gate | Power at design target | Status |
|---|---|---|
| Economic floor (F1) | **81.0%** | ✔ |
| CI excludes zero, one-sided | 85.4% | ✔ |
| **Joint (F1 as specified)** | **81.0%** | **✔** |
| Quarterly consistency | 85.1% at viability Sharpe | ✔ |
| Profit factor | Implied by Sharpe 0.73 < viability 0.60 threshold spacing | ✔ |
| Maximum drawdown | Scales with `s`; allowance fixed at 2.0× annual vol | ✔ |
| Worst month | `m` chosen so expected breaches ≤ 0.20 | ✔ |
| Deflated Sharpe | Dominated by F1; reported for completeness | ✔ |

**If any gate's power falls below 75% at instantiation, the θ decision in H1.7 is revisited before measurement** — the 40th percentile increases `N_yr` and `N_G1` and raises power across the table. This is a single pre-specified remedy applied before any outcome is examined.
<!-- verbatim:end B.7 -->

> **v9 addition (final_spec principle 9 and §9 Step 4):** every gate also carries a false-acceptance rate, which is capped. No source gives the cap or the false hypothesis to compute it under — see B.12 and B.13 C13.

> **Open:** the θ remedy in the last paragraph is one of v6's two low-power remedies; v8/v9 §12 keeps only one — C7. **Resolved (C1):** references to the gate formulas point at B.6 (v6 §II.2), and to the power requirement at B.7 (v6 §II.3).

> **Superseded where B.14 differs** (θ 50th percentile; single θ-drift remedy; audit trigger target + Φ⁻¹(0.95)·SE replaces 3 × target). See `docs/v9_1_design_decisions.md`.


### B.8 Position sizing

*Source: `docs/spec_v6.md` v6 §II.4, lines 466–470.*

<!-- verbatim:begin B.8 -->
#### §II.4 — Position Sizing

**Sizing is volatility-targeted:** one σ_post move equals `R` = 1.0% of equity, halved to 0.5% if the mechanism is unconfirmed. Kelly and adaptive schemes are prohibited through G3.

Percentage drawdown gates are meaningless without a stated sizing scheme, and `annual_vol` — which several gates scale against — is a direct function of `R` and `s`.
<!-- verbatim:end B.8 -->

> **v9 change (final_spec §16, explicit):** "Fractional Kelly, not fixed-fraction volatility targeting." **Resolved (C6, v9.1 decision D3):** gates are evaluated at the fixed reference risk `R` = 1% (v6), regardless of Kelly sizing.


### B.9 Interpretation bands (G1)

*Source: `docs/spec_v6.md` v6 G1 → Interpretation bands, lines 520–528.*

<!-- verbatim:begin B.9 -->
##### Interpretation bands

| Point estimate | Reading |
|---|---|
| CI includes zero | Dead. Stop |
| Below `floor` | Below viability. Stop |
| `floor` to `target` | Real but thin. Proceed with elevated scrutiny |
| `target` to `3 × target` | Proceed |
| **Above `3 × target`** | **Audit for artefact and leakage before proceeding** |
<!-- verbatim:end B.9 -->

> **Superseded where B.14 differs** (θ 50th percentile; single θ-drift remedy; audit trigger target + Φ⁻¹(0.95)·SE replaces 3 × target). See `docs/v9_1_design_decisions.md`.


### B.10 The two overrides (audit trigger)

*Source: `docs/spec_v6.md` v6 §II.5, lines 553–557.*

<!-- verbatim:begin B.10 -->
#### §II.5 — The Two Overrides

**Marginal means stop.** A point estimate between zero and the viability floor is below viability, not a finding.

**Surprisingly good means audit.** A point estimate above three times the design target at G1, or a Deflated Sharpe above 1.5 at G3, is treated as an artefact or leakage suspect first and a result second.
<!-- verbatim:end B.10 -->

> **Superseded where B.14 differs** (θ 50th percentile; single θ-drift remedy; audit trigger target + Φ⁻¹(0.95)·SE replaces 3 × target). See `docs/v9_1_design_decisions.md`.


### B.11 Temporal partitions

*Source: `docs/spec_v6.md` v6 §17.1, lines 783–791.*

<!-- verbatim:begin B.11 -->
##### 17.1 Temporal partitions

| Partition | Share | Use |
|---|---|---|
| **Training** | 60% | θ, the volatility method and lookback, and all H1.5a parameters are derived here and only here |
| **Validation** | 20% | Controlled architecture decisions |
| **Holdout** | 20% | Untouched until the architecture is frozen |

`N_G1` is the qualifying count across training and validation combined, since G1 runs before the architecture is frozen.
<!-- verbatim:end B.11 -->

> **v9 change (final_spec §12, explicit):** holdout fixed at 30% of the window, and every formula uses the same count. The training/validation split of the remaining 70% is not stated in v8 or v9.


### B.12 Not in any source

Names the S1.2 prompt or v9 uses that no spec (v5, v6, v8, v9) defines, and their status after the 2026-10-01 decisions (B.13):

| Name | Status |
|---|---|
| `S_research` | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** (C2/C3, reopened 2026-10-02) |
| `S_target` | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** (C2/C3, reopened 2026-10-02): v9 §13's 0.75+ is a portfolio-level standard, so it cannot be applied as is to a single-mechanism gate |
| `N_hold` | **Resolved (C5):** v6 `N_q`, complete quarters in the 30% holdout only |
| `block_consistency_threshold` | **Resolved (C5):** v6 quarterly consistency (B.6), using `N_q` as above |
| `required_ic` | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** (C12) |
| net floor | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** (C11) |
| false-acceptance cap | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** (C13) |
| `profit_factor_gate` | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** (C9) |
| `worst_month_gate` | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** (C10) |

### B.13 Conflicts and ambiguities — decisions of 2026-10-01

| # | Topic | v6 (copied) | v8 / v9 / project docs | Decision / status |
|---|---|---|---|---|
| C1 | Section numbering | Gate formulas are §II.2; §II.3 is the power requirement | Playbook S1.2 and old MASTER_CONTEXT cited "§II.3 (gate formulas)" | **Resolved:** repoint to B.6 (gate formulas) and B.7 (power) accepted |
| C2 | Number of Sharpe standards | `floor` 0.60, `target` 0.89, profit factor at 0.73 (B.6) | §12: "One research standard and one operational standard (§13); every gate checked against both before freezing"; §9 Step 3: one business input | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** (decided 2026-10-01 as "v9's standards win", `S_research` = 0.60, `S_target` = 0.75; **reopened 2026-10-02 by the user**). Reason: v9 §13 says "**Operational standard:** 0.75+ annualised Sharpe, portfolio level — the bar for committing meaningful capital, evaluated only after breadth is added", so it cannot be dropped in as `target` on a single-mechanism gate. The research standard reads "0.60 annualised Sharpe, single instrument". The profit-factor standard (0.73) is decided v9.1 (2026-10-05) (C9) |
| C3 | `S_target`, `S_research` | Literals 0.89 and 0.60 | Prompt names them as inputs | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** with C2 (reopened 2026-10-02) |
| C4 | Holdout share | 20% (B.11) | 30% (§12, explicit) | **Resolved:** 30% holdout, per v9 |
| C5 | Consistency gate | Quarterly, over `N_q`; worked example `N_q = 46` is the whole ~11.5-year window | Prompt: `block_consistency_threshold(N_hold)` | **Resolved:** count quarters over the holdout only. v6's worked example (`N_q = 46`) does not apply |
| C6 | Sizing behind `annual_vol` | `R` = 1.0% vol-targeted (0.5% if unconfirmed), Kelly prohibited through G3 (B.8) | §16: fractional Kelly, "not fixed-fraction volatility targeting" | **Resolved — v9.1 decision D3:** gates are evaluated at a fixed reference risk `R` = 1% (v6), regardless of Kelly sizing. Not addressed by the decision: v6's halving to 0.5% when unconfirmed |
| C7 | θ and low-power remedies | θ = 60th percentile; two remedies: θ-drift → σ_pre lookback 120 (B.1), power < 75% → θ to 40th (B.7) | §12: one remedy, applied once; MASTER_CONTEXT §6: θ = 40th percentile | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** |
| C8 | Falsification count | F1–F9: F8 artefact, F9 placebo (B.3) | MASTER_CONTEXT and playbook said "F1–F8" | **Resolved:** F1–F9. MASTER_CONTEXT, playbook and build plan corrected |
| C9 | Profit factor | "≥ PF corresponding to Sharpe 0.73, computed from the empirical distribution" | v5 gave PF = E[X⁺]/E[X⁻] and 1.20 under normality; v6 does not defer to v5 | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** |
| C10 | Worst month | `m` from expected breaches ≤ 0.20 under measured kurtosis; `monthly_std` undefined | v5 used −3 × monthly std | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** |
| C11 | Costs and the net floor | Cost = per-event spread ÷ σ_post; F1 tests mean net `u` against `floor` | §15a: broker round-trip cost is the measured constant (explicit); "net floor" undefined | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** |
| C12 | `required_ic` | — | §15a step 5 requires it; no formula | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** |
| C13 | False-acceptance | Only false-rejection specified (B.7, ≤ 25%) | Principle 9 / Step 4: capped false-acceptance | **Decided v9.1 (2026-10-05), see docs/v9_1_design_decisions.md** |
| C14 | Fix measurement windows | FIX pre 15:45:00–16:02:30, post 16:02:35–17:00:00 | v9 §11 and MASTER_CONTEXT: fix window 15:57:30–16:02:30 | **Deferred to S2.3** |
| C15 | Asserted constants | v6 formulas contain fixed numbers (0.89, 0.60, 0.73, 2.0, 1.04, 0.20, 0.5/0.8, 200, 3×, 1.5, 0.99, 0.999) | §9 Step 3: "No numeric threshold is asserted directly"; MASTER_CONTEXT rule 1 allows frozen values in `config/gates_frozen.yaml` | **Approved 2026-10-05** |

### B.14 v9.1 decisions (2026-10-05)

Source: `docs/v9_1_design_decisions.md`, which is the authority for every decision it records. The two sections below are copied from it verbatim (its sections 7 and 2), except that each section heading is demoted to fit this appendix. They are not v6 text and are not between `verbatim` markers. Where they differ from a v6 block above (for example θ at the 50th percentile, the cost percentiles, the 0.84 target, the profit-factor and drawdown minimums), these decisions apply. References inside the copied text to "section N" and "Decision N" are to sections and decisions of that document.

#### 7. Formula reference for S1.2

Notation as in Appendix B. All calibration uses training + validation only; "holdout" quantities count holdout periods only.

- `floor = S_research · s / √N_yr`; `target = S_target · s / √N_yr`; `SE` per B.2 (larger of date-clustered and Newey–West for the fix-only test).
- **Cost.** For each pair p and day type k ∈ {ordinary, month_end}: `c[p,k]` = Q_k of the adjusted per-event cost pool (Q = 0.75 ordinary, 0.60 month-end); `mean_c[p,k]` reported. Each event is charged `c[p, type(event)]`.
- **F1 (gate).** `mean(u_gross − c_event) ≥ floor` AND one-sided 95% lower bound of mean net u > 0.
- **Power (F1).** P(pass F1 | true net mean = target) ≥ 0.75, both conditions jointly. The freeze is refused if not met.
- **False acceptance (F1).** P(pass F1 | true net mean = 0) ≤ 0.05. Report P(pass | true net mean = floor) as the Kind 2 property.
- **required_ic.** `IC = mean(−sign(d)·u_gross)/s`. Per pair: `S_research/√N_yr + c̄/s`. Pooled: `S_operational/√B + c̄_pooled/s_pooled`, `B = n·N_yr/(1+(n−1)ρ)`, ρ measured. `c̄` = cost weighted by the share of qualifying events of each day type. "Likely infeasible" if pooled required IC > 0.05; per-pair verdicts reported.
- **Profit factor (gate).** `PF(x) = Σx⁺ / Σ|x⁻|`. Minimum = PF of empirical net u shifted additively to mean = floor.
- **Quarterly consistency (gate).** Proportion profitable ≥ `p_floor − z·√(p_floor(1−p_floor)/N_q)`, `p_floor = Φ(S_research/√4)`, `z = Φ⁻¹(0.75)`, `N_q` = complete holdout quarters.
- **Maximum drawdown (gate).** Limit = 75th percentile of maximum drawdown over the holdout length, block bootstrap of net u shifted to mean = floor, at R = 1%.
- **Worst month (flag).** `monthly_std` = std of monthly net sums. `m` = smallest value such that P(any holdout month < −m·monthly_std) ≤ 0.25 under block bootstrap of net u shifted to mean = floor. Reported, not gated.
- **F2 minimum effective sample.** Smallest n at which F1's false-acceptance rate (block bootstrap, mean shifted to 0) ≤ 0.05.
- **G1 audit trigger.** `target + Φ⁻¹(0.95)·SE` (mean-u units); replaces "3 × target" in B.9 and B.10.
- **Deflated Sharpe (D16, 2026-10-05).** Minimum 0.5 (0.8 if unconfirmed) on the **probability scale** returned by `TrialRegistry.deflated_sharpe` (`Φ(z)`, in [0, 1]). Audit alarm restated on the **Sharpe scale**: the level a genuine strategy at S_target would exceed only 5% of the time (the same 5% rule as the G1 trigger); replaces "Deflated Sharpe above 1.5". Neither value is in the S1.2 config; both are settled at S3.1, which must confirm combined power at S_target with F4 included before the freeze.
- **Fixed quantiles.** `Q_spread = F_spread⁻¹(0.99)`; `Q_disaster = F_|u|⁻¹(0.999)`.
- **Bootstrap settings.** Block length = measured dependence horizon (S1.1); draw count and seed recorded per run.

#### 2. The closed list of chosen numbers

During the session the list was stated as eight numbers. Applying decision 2 strictly, every number the user chose belongs on it, so the full list is longer:

| Number | Role | Where decided |
|---|---|---|
| 0.60 | Research standard (single-mechanism pass mark) | Decision 1 |
| 0.84 | Design target — an **assumption**, checked against literature | Decision 1 |
| 0.75 | Operational standard, portfolio level only | Decision 2 |
| 0.05 | Feasibility cutoff for `required_ic` | Decision 5 |
| 0.5 / 0.8 | Deflated Sharpe minimum (unconfirmed: 0.8), probability scale of `TrialRegistry.deflated_sharpe` | Decision 10; D16 (2026-10-05) |
| 99% | Spread skip quantile | Decision 10 |
| 99.9% | Disaster-stop quantile | Decision 10 |
| 50th percentile | θ | Decision 3 |
| 75th / 60th percentile | Cost percentile, ordinary / month-end | Decision 4 |
| ¾ | Live size multiplier for UNCONFIRMED (Stage 5) | Decision 9 |
| R = 1% | Reference risk for gates | D3 (2026-10-01) |
| 30% | Holdout share | C4 (2026-10-01) |
| 15 pp; 60 / 120 | θ-drift trigger; σ lookbacks | **Not yet confirmed** (section 6) |

**Conventions (not on the list):** 95% one-sided confidence, 75% minimum power, 5% false-acceptance cap.

**Calculated in the code, with these supporting settings:** bootstrap block length = the measured dependence horizon (S1.1); the number of bootstrap draws and the random seed are computational settings, recorded with each run.
