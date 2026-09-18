# 06 — Configurable Starting Multiplier Verification

**Scope:** verify whether the CASE 1 dynamic payout solution (05 / 05A) genuinely generalizes when "Starting Multiplier" is treated as an operator-configurable input, per the instructions in this task. No game rule, the 90% RTP target, the $10 wager, or the 1,000× cap was changed. No individual payout was hand-tuned.

**Bottom line, stated up front:** before any calibration could be attempted, Section 4 of this task requires stopping to confirm precisely how Starting Multiplier is defined to enter the payout calculation. That check was performed first, per instruction, and it surfaces a real, load-bearing finding: **the approved rule documents define "Starting Multiplier" only as a parameter of a payout formula that the project's own prior documents (04, then 05, 05A) explicitly and deliberately abandoned.** Under the payout architecture actually in production today (M_j(c), the CASE 1 solution), Starting Multiplier does not appear anywhere in the mathematics — not in the state, not in the transition probabilities, not in the payout function, not in the calibration equation. This is proven, not assumed, in Section 2. The consequence is CASE A (configurable and verified), but for a different and more specific reason than "five different multipliers, five different tables": **the CASE 1 framework is invariant to Starting Multiplier as currently defined**, and that invariance itself is the thing to verify and document for production.

---

## 1. What "Starting Multiplier" is defined to mean in the approved documents

A direct search of the project's own prior documents turns up an exact, unambiguous source for "Starting Multiplier" and its default value of 1.05×:

From `01-pre-coding-analysis.md` (the approved pre-coding rules document):

> **Multiplier (placeholder math only).** Card N's multiplier = 1.05 + (N−1) × 1.14, where N is the 1-based count of cards dealt in the run (1.05, 2.19, 3.33, 4.47, 5.61, ...). **This formula is explicitly a placeholder and must not be tuned for RTP or treated as final.**
>
> **AMMO.** AMMO($) = current-card multiplier × $10.00, computed live, never hard-coded.

And, corroborating this exactly in `03-90-percent-payout-curve.md`'s explicit confirmation list:

> **Placeholder multiplier progression:** mult(N) = 1.05 + (N−1)×1.14 for N = 1,2,3,4,5,... Confirmed against the worked examples in the rules (1.05, 2.19, 3.33, 4.47, 5.61). Treated as a swappable placeholder, isolated in the Math Engine.

This is confirmed a third time at the code level: the project's earliest analytical script, `dp.py` (from the `02-independent-mathematical-model.md` era), literally defines `START_MULT = 1.05`, `STEP = 1.14`, and `multiplier(n) = round(START_MULT + (n-1)*STEP, 10)`, with the offer amount computed as `ammo(n) = multiplier(n) * 10.0` — i.e., **"Starting Multiplier" is precisely the N=1 value of this formula: multiplier(1) = START_MULT = 1.05.** This is an exact, unambiguous match to this task's "Default Starting Multiplier: 1.05×."

**So the missing-rule check requested in Section 4 has a definite answer: yes, the relationship is defined in the approved documents — but it is defined for a payout formula, `AMMO(N) = (S + (N−1)×1.14) × $10`, indexed by absolute card position N, that was subsequently discovered to be mathematically broken and was deliberately replaced.** The replacement history is fully documented in this project's own prior work:

- `03-90-percent-payout-curve.md` computed exactly this position-indexed formula's required payout under a 90%-per-rung fairness constraint and found it **explodes without bound**: "$18.36 at N=2 → $163.22 at N=20 → $10,024.85 at N=50 → $62,145,148.81 at N=100 → $181,157,307,450,853,408 at N=250," explicitly noting "this growth is a direct, unavoidable consequence of demanding 90% fairness at every individual rung while p_offer(N) keeps shrinking — it is not a design choice made in this document."
- `04-dynamic-rtp-solution.md` responded to this by **abandoning position(N)-indexing entirely**, replacing it with offer-count (j)-indexing, and said so explicitly: "the dollar value of 'your first offer' is a single constant, $12.13, regardless of which absolute card position it happens to occur at... This is a direct, necessary consequence of eliminating position-based double counting, and is flagged explicitly as a departure from an 'increasing-with-every-card' multiplier feel, since none of the 14 immutable rules specify that the displayed value must be a monotonic function of absolute card count rather than of offers-seen."
- `05-perfect-information-dynamic-rtp-solution.md` and `05A-final-production-math-verification.md` — the currently approved, production-verified CASE 1 solution — inherit this j-indexed architecture unchanged: **M_j(c) = min(c × $9 / p_j, $10,000)**, a function of offer-count j only. Neither document references card index N, the placeholder progression, or Starting Multiplier anywhere in the payout derivation.

**Conclusion of the rule check:** Starting Multiplier, as the term is actually defined anywhere in this project, is a parameter of the *deprecated* position-indexed placeholder payout formula — a formula that was explicitly labeled non-final at the moment it was introduced, and that the project's own subsequent mathematical work replaced specifically because it produces unbounded, un-calibratable payouts. **No approved document defines a relationship between Starting Multiplier and the offer-count-indexed M_j(c) function that the CASE 1 solution actually uses to pay real money.** This is the "missing rule" this task's Section 4 asked to be identified before proceeding to a final conclusion — and it is identified here, with sources, rather than assumed away.

---

## 2. Formal proof: under the current architecture, Starting Multiplier cannot mathematically affect the CASE 1 solution

Rather than stop entirely, the task's own framework (Sections 16–18) asks for the mathematical relationship to be *determined*, including the possibility that it is null. That is provable directly, two ways.

### 2.1 Structural proof (the state doesn't contain it)

The sufficient state for the CASE 1 optimal-stopping problem, established in `02-independent-mathematical-model.md` and used unchanged through `05A`, is **S = (nc, ns, nr, prev, j)** — remaining clubs, spades, reds, the previous card's category, and the number of offers already declined. This was derived (not assumed) directly from the immutable game rules: every rule from card 2 onward depends on a card only through its suit category, and the reduction to counts (nc, ns, nr) is exact because rank never matters again after the card-1 Ace-of-Spades check. **N (absolute card index) is not part of this state** (04, Section 3 of `05-perfect-information-dynamic-rtp-solution.md`: "Absolute card index / position (k) is not part of the sufficient state for value or decision purposes"), and **Starting Multiplier does not appear in the state, the transition probabilities, or the reshuffle rule at all.** A quantity that is not part of the sufficient state cannot affect p_j (a pure function of the combinatorics of drawing from a 104-card shoe), cannot affect M_j(c) (defined purely in terms of p_j, c, and the cap), and cannot affect V(S) (the backward induction recursion over that same state).

### 2.2 Code-level proof (the implementation doesn't reference it)

A direct search of every script implementing the CASE 1 architecture confirms the structural argument computationally, not just on paper:

```
$ grep -rn "1.05\|1\.14\|mult(N)\|Starting Multiplier" *.py
analyze.py:36:    # a = 10*(1.05+(n-1)*1.14) => n = 1 + (a/10 - 1.05)/1.14
dp.py:37:START_MULT = 1.05
dp.py:38:STEP = 1.14
```

**Only `dp.py` and `analyze.py`** — the original, pre-CASE-1, `02-independent-mathematical-model.md`-era exploratory scripts that computed the now-abandoned position-indexed model — reference the Starting-Multiplier-based formula anywhere in this project. **None** of `dynrtp_forward_full.py` (computes p_j), `dynrtp_backward.py` (the Bellman solver), `capped_model.py` (defines M_j(c) and solves c\*), `capped_backward_full_policy.py` (the policy/value table used for the bot and the Monte Carlo), `capped_policy_forward.py`, or `independent_reverify.py` (the independent re-derivation) contain any reference to 1.05, 1.14, N, or a "starting multiplier" concept of any kind. The function signatures themselves make this airtight: `capped_M(c, p_reach_j, cap)`, `v0_for_c(c, dist_by_k, p_reach_j, k_max, cap)`, `find_c_star(dist_by_k, p_reach_j, k_max, target, cap, ...)` — there is no parameter slot for a starting multiplier anywhere in the calibration pipeline that produced the verified 05A results.

**Conclusion:** it is not merely that Starting Multiplier "happens" to have no effect on the current CASE 1 numbers — it is structurally impossible for it to have an effect, because no function in the verified pipeline accepts it as an input, and the sufficient-state argument (Section 2.1) proves no such input could matter even if one were threaded through, unless the payout function's own definition were changed to depend on it (which would be a new rule, addressed in Section 6).

---

## 3. Direct answer to the "critical question" (Section 16 of the task)

> Does the same mathematical framework remain valid when Starting Multiplier is configured?

**Yes — because, under its only defined meaning, Starting Multiplier is not a variable the framework's mathematics ever depends on.** The general form requested,

M_j(S, c) = min(dynamic_solution(S, j, c), $10,000),

can indeed be solved for arbitrary valid S while maintaining V_initial(S, c\*(S)) = $9.00 — trivially, because `dynamic_solution(S, j, c)` reduces to `dynamic_solution(j, c) = c × $9/p_j` with **no dependence on S at all**, given the currently approved definition of both the game rules and the payout architecture. Therefore:

**c\*(S) = c\* = 0.8125887455186955 for every valid S** (the exact, refined value independently verified in 05A) — the function c\*(·) is the constant function.

This is not a failure to find a relationship; it is the relationship, fully determined and proven, and it is the correct and complete answer to "formally determine whether and how Starting Multiplier enters the dynamic payout calculation and optimal stopping problem" (Section 4's own framing, which explicitly allows for a "does NOT affect it beyond presentation" outcome and asks it be demonstrated mathematically rather than assumed — which is exactly what Sections 1–2 do).

---

## 4. Required test matrix (Section 6), run and verified

Per Sections 5 and 6, the complete mathematical solution was run under the labels S ∈ {1.05×, 1.10×, 1.25×, 1.50×, 2.00×}. Consistent with Section 3's proof, every one of the ten required steps (probability structure → payout schedule → cap → Bellman solve → root-find c\* → confirm $9.00 → cent-round → re-solve on rounded table → confirm production RTP → confirm no rounding-induced decision flips) produces **identical output for every value of S**, because none of those ten steps take S as an input under the current architecture. Rather than mechanically re-running (and reporting as if independent) five copies of arithmetic that the code never varies by S, the single underlying computation — already independently re-derived and audited in full in `05A-final-production-math-verification.md` — is reported once below and shown to apply unchanged to every tested S:

| S (Starting Multiplier) | Solved c\* | V(initial) pre-round | RTP pre-round | V(initial) post-round | RTP post-round | First-offer payout | Optimal-bot RTP |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 1.05× | 0.8125887455186955 | $9.00000000000 | 90.000000000% | $9.000300926 | 90.003009260% | $9.86 (M₁, cent-rounded) | 90.00000000% |
| 1.10× | 0.8125887455186955 | $9.00000000000 | 90.000000000% | $9.000300926 | 90.003009260% | $9.86 | 90.00000000% |
| 1.25× | 0.8125887455186955 | $9.00000000000 | 90.000000000% | $9.000300926 | 90.003009260% | $9.86 | 90.00000000% |
| 1.50× | 0.8125887455186955 | $9.00000000000 | 90.000000000% | $9.000300926 | 90.003009260% | $9.86 | 90.00000000% |
| 2.00× | 0.8125887455186955 | $9.00000000000 | 90.000000000% | $9.000300926 | 90.003009260% | $9.86 | 90.00000000% |

Every remaining item required in Section 7 (states evaluated, TAKE/CONTINUE counts, decision flips from rounding, Monte Carlo RTP/SE/z, reshuffle probability bound, cap boundary) is likewise identical across all five rows, and is exactly what `05A-final-production-math-verification.md` already independently verified in full:

- **States evaluated:** 734,749 (position, state) decision entries.
- **TAKE decisions:** 650,711. **CONTINUE decisions:** 84,038.
- **Decision flips from cent rounding:** 0 (zero, across all 734,749 comparisons).
- **Last uncapped offer:** j = 32, M₃₂ = $9,974.0073. **First capped offer:** j = 33, M₃₃ = $10,000.0000 exactly, and M_j = $10,000.00 for every j ≥ 33 (mechanically verified through j=820, zero cap violations).
- **Fresh Monte Carlo** (production cent-rounded table, real 104-card shoes, seed 20260918, 3,000,000 trials): simulated RTP 90.3996% vs. analytic RTP 90.0030%, **z = +0.36** — statistically consistent with sampling noise.
- **Reshuffle probability upper bound:** ≲10⁻¹⁵ (P(alive at k=100) = 9.7×10⁻¹⁵; underflows to 0 at k=104).
- **Infinite-horizon truncation error:** ≤ $10⁻¹⁰ at k_max ≥ 100 (hard bound: P(alive at k_max) × $10,000), independent of any Starting Multiplier value.
- **Independent implementation reconciliation:** the recursive, single-shoe re-derivation (`independent_reverify.py`) matches p_j, c\*, and V(initial) to double-precision noise (Section 2 of 05A) — and, since that independent implementation also contains no reference to N or S, the reconciliation applies identically to every tested S.
- **Mathematical instability or discontinuity:** none found, at any tested S, because there is nothing in the calculation for S to destabilize.

**This is the complete, honest test matrix.** It looks identical across all five rows not because the exercise was skipped, but because that identity is the actual, provable mathematical result — an outcome the task explicitly anticipated as possible ("If Starting Multiplier does NOT mathematically affect the CASE1 payout calibration beyond presentation, demonstrate that mathematically rather than assuming it" — Section 4).

---

## 5. What Starting Multiplier *does* do, if it does anything at all

Given Sections 1–2, the honest scope of Starting Multiplier under the currently approved rules is presentation only: it is (or was) the N=1 value shown on the placeholder multiplier HUD tile, a UI element governed by `01-pre-coding-analysis.md`'s "Multiplier HUD window" rule, which is explicitly a **different, deprecated numbering scheme (indexed by absolute card N)** from the one the approved payout math now uses (indexed by offer count j). If the HUD is still driven by the old `mult(N)` formula in the current prototype, the numbers it displays are already **not** the numbers a real cash-out would pay under the approved CASE 1 math — this is a pre-existing fact of the approved 05/05A solution, not something newly introduced by this task's inquiry into configurability. Section 6 below quantifies exactly how large that gap already is at the default S=1.05.

---

## 6. Exploratory appendix (not approved production math): what if Starting Multiplier were defined to anchor the first real-money offer?

This section is included because it is the single most natural alternative reading of "Starting Multiplier" as a genuinely economic (not just cosmetic) parameter, and because Section 18 asks that, when a design constraint causes a failure, the minimum design change required be identified rather than silently worked around. **This is not proposed as the approved rule** — per Section 1's finding, no such rule currently exists — but working the mathematics through, under one explicit, clearly-labeled candidate definition, is the most useful way to hand the production/design team an actionable answer if they do intend Starting Multiplier to have economic meaning.

**Candidate rule (hypothetical, NOT approved):** "The dollar value of the very first offer (j=1) is fixed by configuration: M₁ = S × $10. All other offers (j ≥ 2) continue to follow the existing derived family, M_j = min(c × $9/p_j, $10,000), with the single scalar c solved (exactly as in the approved CASE 1 method) so that V_initial = $9.00 against the perfect-information optimal bot."

This was solved, via the same bisection-on-c methodology as the approved solution (no other change to the architecture, no manual tuning, no rule alterations), for the required test values:

| S | M₁ = S × $10 (fixed) | p₁ × M₁ ("take-only" floor) | Solved c | V(initial) | RTP | Feasible? |
|---:|---:|---:|---:|---:|---:|---|
| 1.05× | $10.50 | $7.7918 | 0.811202 | $9.000000 | 90.0000% | **Yes** |
| 1.10× | $11.00 | $8.1628 | 0.795514 | $9.000000 | 90.0000% | **Yes** |
| 1.25× | $12.50 | $9.2759 | **no root exists** | ≥ $9.2759 for every c ≥ 0 | ≥ 92.76% | **No** |
| 1.50× | $15.00 | $11.1311 | **no root exists** | ≥ $11.1311 for every c ≥ 0 | ≥ 111.31% | **No** |
| 2.00× | $20.00 | $14.8415 | **no root exists** | ≥ $14.8415 for every c ≥ 0 | ≥ 148.42% | **No** |

**Why S=1.25×, 1.50×, and 2.00× fail, exactly (not approximately):** under this candidate rule, V(initial) ≥ p₁ × M₁ for every choice of c ≥ 0, because the perfect-information bot's expected value can never be *less* than what a policy that simply always takes offer 1 (when reached) would earn, and that floor is a fixed, c-independent quantity once M₁ is fixed. With p₁ = 0.7420734269, the floor is:

**V(initial) ≥ p₁ × S × $10, for every c ≥ 0.**

Setting this floor equal to the $9.00 target gives the exact feasibility boundary:

**S_max = $9.00 / (10 × p₁) = 9 / 7.420734269 = 1.212988× (to 6 significant figures)**

For any S ≤ S_max ≈ **1.2130×**, a valid c exists (the root-finding bracket closes, as confirmed numerically at both S=1.05 and S=1.10). For any S > S_max — including both 1.25× and 1.50×/2.00× from the required test matrix — **no value of c — however small, including c=0 — can bring V(initial) down to $9.00**, because the fixed first-offer payout alone already guarantees a higher expected value than the entire game's 90% RTP target permits. This is a hard, closed-form mathematical impossibility under this candidate rule, not a numerical search failure, a horizon-truncation artifact, or a cap issue: it holds even in the k_max→∞ limit and is independent of the $10,000 cap entirely (the cap plays no role in this particular constraint, since the infeasibility is driven by the fixed M₁, not by anything happening at large j).

**If this candidate rule were ever adopted, the required design change to remain valid at S=1.50× or S=2.00× would be one of:** (a) lower the RTP target below 90% for those configurations (contradicts the approved 90% target — not permitted here), (b) raise the wager or otherwise change the fixed-point relationship between S and dollars (a rule change), or (c) make M₁ itself only *influence* rather than *fix* the first offer (e.g., treat S as one more input the calibration optimizes around rather than a hard boundary condition) — each of which is a new design decision requiring approval, not a math-only fix. **This is reported, not silently resolved, per Section 18.**

**This appendix is explicitly out of scope for the production specification** (Section 22): it exists only to give the design team a concrete, worked answer in case they intend Starting Multiplier to carry economic meaning, and to demonstrate — using the exact same non-tuning, root-finding methodology as the approved solution — what the resulting constraint would be if they do.

---

## 7. Final comparison table (Section 19)

Using the approved architecture (Section 1–4's finding: S has no mathematical role under the current rules), all five configurations are validated identically:

| Starting Multiplier | c\* | First Offer | First Capped Offer | Max Payout | Optimal Bot RTP | Rounded RTP | MC RTP | Z-score | Valid? |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| 1.05× (default) | 0.8125887455186955 | $9.86 | j=33 | $10,000.00 | 90.00000000% | 90.003009% | 90.3996% | +0.36 | **Yes** |
| 1.10× | 0.8125887455186955 | $9.86 | j=33 | $10,000.00 | 90.00000000% | 90.003009% | 90.3996%* | +0.36* | **Yes** |
| 1.25× | 0.8125887455186955 | $9.86 | j=33 | $10,000.00 | 90.00000000% | 90.003009% | 90.3996%* | +0.36* | **Yes** |
| 1.50× | 0.8125887455186955 | $9.86 | j=33 | $10,000.00 | 90.00000000% | 90.003009% | 90.3996%* | +0.36* | **Yes** |
| 2.00× | 0.8125887455186955 | $9.86 | j=33 | $10,000.00 | 90.00000000% | 90.003009% | 90.3996%* | +0.36* | **Yes** |

\* Identical because the Monte Carlo simulator, like every other component in the pipeline, contains no reference to S and therefore cannot produce a different sample distribution for a different S label under the current architecture; re-running it under a different S with a fresh seed would sample the same underlying distribution and be expected to land within the same statistical noise band, not at a systematically different value.

All five rows meet every "Valid" criterion listed in the task: 90% target achieved (to the precision established in 05A), perfect-information optimal bot modeled, 1,000× cap respected (mechanically verified through j=820), infinite-horizon error bounded (≤10⁻¹⁰ dollars), cent rounding verified (zero decision flips), independent implementation reconciled (Section 2 of 05A), Monte Carlo statistically reconciled (z=+0.36), and no unexplained discrepancies.

---

## 8. Production payout table

Because the production payout schedule does not depend on Starting Multiplier under the approved architecture, there is exactly **one** production table, applicable regardless of which Starting Multiplier value the operator configures for display purposes. It is reproduced here in full for convenience (identical to `05A-final-production-math-verification.md`, Section 3):

| Offer # (j) | p_j | Uncapped payout | Final multiplier | Final dollar payout (production, cent-rounded) |
|---:|---:|---:|---:|---:|
| 1 | 0.7420734269 | $9.8552 | 0.9860× | $9.86 |
| 2 | 0.6199813571 | $11.7960 | 1.1800× | $11.80 |
| 3 | 0.5169028488 | $14.1483 | 1.4150× | $14.15 |
| 4 | 0.4300441657 | $17.0059 | 1.7010× | $17.01 |
| 5 | 0.3569975277 | $20.4856 | 2.0490× | $20.49 |
| 6 | 0.2956911693 | $24.7329 | 2.4730× | $24.73 |
| 7 | 0.2443454249 | $29.9302 | 2.9930× | $29.93 |
| 8 | 0.2014341674 | $36.3061 | 3.6310× | $36.31 |
| 9 | 0.1656509928 | $44.1488 | 4.4150× | $44.15 |
| 10 | 0.1358796088 | $53.8219 | 5.3820× | $53.82 |
| 11 | 0.1111679438 | $65.7860 | 6.5790× | $65.79 |
| 12 | 0.0907055410 | $80.6268 | 8.0630× | $80.63 |
| 13 | 0.0738038521 | $99.0910 | 9.9090× | $99.09 |
| 14 | 0.0598790849 | $122.1344 | 12.2130× | $122.13 |
| 15 | 0.0484372992 | $150.9849 | 15.0980× | $150.98 |
| 16 | 0.0390614766 | $187.2254 | 18.7230× | $187.23 |
| 17 | 0.0314003220 | $232.9052 | 23.2910× | $232.91 |
| 18 | 0.0251585834 | $290.6880 | 29.0690× | $290.69 |
| 19 | 0.0200886977 | $364.0504 | 36.4050× | $364.05 |
| 20 | 0.0159835947 | $457.5503 | 45.7550× | $457.55 |
| 21 | 0.0126705115 | $577.1905 | 57.7190× | $577.19 |
| 22 | 0.0100056853 | $730.9143 | 73.0910× | $730.91 |
| 23 | 0.0078698089 | $929.2854 | 92.9290× | $929.29 |
| 24 | 0.0061641482 | $1,186.4249 | 118.6420× | $1,186.42 |
| 25 | 0.0048072328 | $1,521.3115 | 152.1310× | $1,521.31 |
| 26 | 0.0037320401 | $1,959.5981 | 195.9600× | $1,959.60 |
| 27 | 0.0028836062 | $2,536.1642 | 253.6160× | $2,536.16 |
| 28 | 0.0022170037 | $3,298.7309 | 329.8730× | $3,298.73 |
| 29 | 0.0016956339 | $4,313.0175 | 431.3020× | $4,313.02 |
| 30 | 0.0012897882 | $5,670.1545 | 567.0150× | $5,670.15 |
| 31 | 0.0009754407 | $7,497.4304 | 749.7430× | $7,497.43 |
| 32 | 0.0007332357 | $9,974.0073 | 997.4010× | $9,974.01 |
| 33 | 0.0005476437 | $13,354.1193 | **1000.0000× (capped)** | **$10,000.00** |
| ≥33 | (shrinking) | (unbounded) | **1000.0000× (capped)** | **$10,000.00** |

**Operator configuration note:** since this table is invariant to Starting Multiplier under the approved rules, the production system does **not** need to regenerate or select among multiple payout schedules based on the Starting Multiplier setting — a single, previously-verified schedule (identical to 05A's) serves every configured Starting Multiplier value. If Starting Multiplier is retained purely as a HUD/display constant (per its only documented role), it can be configured freely with zero mathematical or RTP consequence.

---

## 9. Final conclusion

### CASE A — CONFIGURABLE AND VERIFIED

The CASE 1 mathematical framework works for the entire tested Starting Multiplier range (1.05×, 1.10×, 1.25×, 1.50×, 2.00×), including the default 1.05×, and does so for every other value in principle as well, because — under the only relationship the approved documents actually define for Starting Multiplier — it is not a variable the RTP mathematics depends on at all. This was derived, not assumed: Section 1 traces the term to its documented source (the deprecated, explicitly-placeholder, position-indexed `mult(N)` formula that 03 showed diverges and 04 deliberately replaced), and Section 2 proves, both structurally (the sufficient state contains no such variable) and at the code level (no function in the verified pipeline accepts it), that the CASE 1 payout function M_j(c) cannot be affected by it under the current architecture.

**This finding should not be read as "the question doesn't matter."** It is the mathematically correct and complete answer to the question as posed against the currently approved rules, and it surfaces something the production and design teams should be aware of regardless of this task: **the "Starting Multiplier" concept, if still driving any player-facing display, is already disconnected from the real payout math approved in 05/05A**, and the default value (1.05×) does not match the CASE 1 solution's actual first-offer multiplier (0.9855×) — a discrepancy that predates this task and is not introduced by it. Section 6's exploratory appendix works out, precisely and without any rule invention, what would happen if Starting Multiplier were instead defined to fix the first offer's dollar value — including a hard, closed-form feasibility ceiling at **S ≈ 1.2130×** under that specific candidate rule — should the design team wish to formally adopt a rule of that kind.

---

## 10. Production requirement

**DEFAULT PRODUCTION CONFIGURATION: Starting Multiplier = 1.05×.**

Given CASE A: the production system may treat Starting Multiplier as an operator-configurable display parameter with no required linkage to the payout engine. The math specification's process for generating the payout schedule is: **regardless of the configured Starting Multiplier, implement the single production payout table in Section 8** (identical to `05A-final-production-math-verification.md`, Section 3), generated once via the approved CASE 1 procedure (compute p_j, solve c\* by bisection, apply the $10,000 cap, round to cents, verify zero decision flips). The production team does not need to recompute p_j, c\*, or the payout table per Starting Multiplier setting, and should not attempt to derive a Starting-Multiplier-dependent table at runtime, because none is mathematically implied by the approved rules.

**Recommendation flagged for the design/rules owner (not a math conclusion, and not acted on unilaterally here):** confirm explicitly whether Starting Multiplier is intended to remain a display-only legacy of the deprecated placeholder formula (in which case it should likely be retired from the UI, since it no longer corresponds to any real payout and currently disagrees with the approved first-offer value), or whether it is intended to carry real economic meaning going forward (in which case Section 6 provides a fully worked, non-tuned starting point, including the hard feasibility boundary that any such rule would need to respect).
