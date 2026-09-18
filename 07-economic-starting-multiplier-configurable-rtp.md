# 07 — Economic Starting Multiplier + Configurable RTP

**Resolves the design gap identified in `06-configurable-starting-multiplier-verification.md`:** Starting Multiplier is now defined and verified as a genuine economic input (M₁ = S × W), Target RTP is now a genuine configuration input (not hardcoded at 90%), and the entire CASE 1 optimal-stopping architecture (perfect-information bot, backward induction, 1,000× cap, cent-rounding discipline, infinite-horizon rigor) is retained unmodified. No game rule was changed. No individual payout was hand-tuned — the curve for every configuration reported below comes from one root-find on one scalar, exactly as in 05/05A/06.

---

## A. The exact generalized mathematical formulation

**Configuration inputs (operator-set):**

- S = Starting Multiplier (default 1.05×)
- R = Target RTP, as a fraction (default 0.90)
- W = Wager = $10.00 (fixed by the immutable game rules)
- C = Maximum multiplier = 1,000× (fixed by the immutable game rules)

**Derived quantities:**

- TARGET_EV = W × R
- p_j = P(reach at least j offers), the exact combinatorial probability from the unmodified card process (two 52-card decks, bust rules) — independent of S, R, W, C.
- **M₁ = S × W** (fixed directly by configuration — this is the new, genuine economic definition of Starting Multiplier)
- **M_j(c) = min( c × TARGET_EV / p_j , C×W ) for j ≥ 2** (a single free scalar c)
- **c\* solves V_initial(S, R, c\*) = TARGET_EV** exactly, via bisection, where V_initial is computed by the same unmodified perfect-information backward induction: V(state) = max(M_{j+1}, E[V(next state)]).

This is the complete generalized system. Section D below derives why this specific form — not one of the several explicitly-rejected hypotheses (M_j = S×j, S×prevmult, S×growthʲ, the old fixed-$9 c×9/p_j, or a flat S/1.05 rescaling of the old table) — is the correct, minimal, non-arbitrary generalization.

---

## B. The exact definition of Starting Multiplier

**Starting Multiplier is the dollar value of the first offer, expressed as a multiple of the wager: M₁ = S × W.** It is not a display-only artifact of the deprecated position-indexed placeholder formula (that was 06's finding about the *old*, unresolved state of the design). Under this task's resolution, S is wired directly into the real payout schedule: it fixes the one number a player will actually see and can actually cash out for at the very first red-card offer, in every configuration, unconditionally. There is nothing indirect or presentation-only about it any more.

## C. The exact definition of Target RTP

**Target RTP (R) is the required expected value returned to a perfect-information optimal player, as a fraction of the wager: V_initial(S, R) = W × R.** It replaces the previously hardcoded 90% constant everywhere that constant appeared in the CASE 1 architecture's payout formula (specifically, the "$9" numerator in M_j = c×$9/p_j becomes the general TARGET_EV = W×R). R is not assumed special at 90% anywhere in the derivation below — 85%, 92%, 93.5%, and 95% are treated with identical mathematical machinery, and the results (Section G) confirm this in practice, not just in principle.

---

## D. Deriving M₂, M₃, ... — why this is the correct, minimal, non-arbitrary architecture

### D.1 The problem is a genuinely underdetermined system, and that must be confronted directly (Section 9 of the task)

Fixing M₁ = S×W and requiring V_initial = W×R gives exactly **two** equations. The unknowns — M₂, M₃, M₄, ... — are (in principle) infinitely many. **This system is drastically underdetermined by design constraints alone.** Infinitely many sequences {M_j}_{j≥2} could be constructed that both respect the cap and produce V_initial = W×R for a perfect-information optimal bot (e.g., one could put all the required expected value into M₂ alone and set every M_j for j≥3 to zero; or spread it uniformly; or concentrate it entirely at the cap for one specific rare j). None of those constructions is "wrong" in the sense of violating a stated rule — but all of them are arbitrary in exactly the way Section 22 of this task forbids ("do not introduce arbitrary smoothing," "any required design choice must be clearly identified for approval"). A specific additional principle is therefore required, and it must be identified explicitly, not silently assumed.

### D.2 The additional principle: preserve the CASE 1 fairness property for every offer that isn't otherwise fixed

The approved CASE 1 architecture (03 → 04 → 05 → 05A) was never itself an arbitrary curve either — its shape (M_j = TARGET_EV/p_j before capping and scaling) was chosen because it is the **unique** payout schedule under which a naive, composition-blind player who commits in advance to any fixed threshold j₀ receives exactly the same expected value (TARGET_EV) at every rung, in isolation (p_{j₀} × M_{j₀} = TARGET_EV for every j₀, by construction). This "equal fairness per naive rung" property is the *existing*, *already-approved* minimal-assumption principle this whole project has used since Section 03. The most defensible way to fill in M₂, M₃, ... here is to **keep exactly this same principle for every offer that isn't otherwise pinned down by the Starting Multiplier rule** — i.e., for j ≥ 2, not for j = 1 (which is no longer free; it is fixed by definition).

This gives: **M_j(c) = min(c × TARGET_EV / p_j, C×W) for j ≥ 2**, with a single free scale factor c (exactly the same single-scalar structure 05/05A/06 already used and already justified via the monotonicity/IVT argument), and **M₁ = S × W held outside that family, fixed by definition.**

**Why this is the "simplest mathematically defensible constraint" (Section 9), not one arbitrary choice among many:** (1) it introduces no new functional form — it is a strict, minimal restriction of the family already approved and re-verified three times over (04, 05/05A, 06); (2) it preserves an interpretable, previously-established fairness property for every offer the design didn't explicitly override; (3) among all payout families satisfying "M₁ fixed, and every other rung individually fair to a naive threshold player up to one common scale," this pins down a *unique* one-parameter family, inside which c\* is uniquely determined by the single equation V_initial = TARGET_EV (Section D.3 below shows this root exists and is unique whenever the hard feasibility bound, Section E, is satisfied). It is not "a" solution to an ambiguous problem; it is "the" solution once the fairness principle already governing this whole project is applied consistently to the part of the curve that Starting Multiplier does not touch.

**This was verified, not assumed, by testing all four of the specifically pre-registered rejected hypotheses first:** M_j = S×j (grows linearly in j regardless of survival probability — has no connection to p_j at all, so it cannot be calibrated against a fixed RTP target without contradicting the cap or the survival curve); M_j = S×previous_multiplier (recreates exactly the deprecated, N-indexed, unboundedly-diverging placeholder formula that 03/04 already disproved); M_j = S×growthʲ (an arbitrary geometric assumption with no derivation from p_j or the fairness principle); M_j = existing_M_j × S/1.05 (a naive rescaling that does not even keep M₁ = S×W exactly, since it would scale the *old*, no-longer-relevant M₁=$9.8552 by S/1.05 rather than defining M₁ directly from S). None of these four is used; the family in Section A/D.2 is a positive derivation from the project's own established fairness principle, not any of the four.

### D.3 How each ingredient enters

- **Configured Target RTP (R):** sets TARGET_EV = W×R, the number the *entire* curve (M₁ fixed plus the M_j(c) family) must average out to under optimal play. It appears explicitly as the numerator of the j≥2 family and as the root-finding target for c.
- **Starting Multiplier (S):** sets M₁ = S×W directly, and — critically — this creates a **floor** on V_initial that constrains what R values are even reachable for a given S (Section E).
- **Probability of surviving to future offers (p_j):** determines how much dollar value M_j(c) must carry to deliver its "fair share" of TARGET_EV; this is unchanged combinatorics, independent of S, R.
- **Probability of bust:** implicitly baked into p_j (a run that busts before reaching offer j is exactly the complement of "reaches offer j"), and directly determines how much dollar value must be loaded onto the offers that do get reached to compensate for the runs that never do.
- **Optimal TAKE/CONTINUE decisions:** the backward induction is run completely unmodified (same recursion, same state (nc,ns,nr,prev,j)); M₁ being fixed rather than part of the smooth family simply changes the value the bot compares against CONTINUE_VALUE at the very first decision — nothing about the recursion itself changes.
- **The 1,000× cap:** applied identically to the j≥2 family (min(·, C×W)); M₁ = S×W is separately checked against the cap (S×W ≤ C×W, i.e. S ≤ C = 1000, trivially satisfied for any realistic Starting Multiplier).

---

## E. The feasibility boundary between Starting Multiplier and RTP

### E.1 The hard bound, derived exactly (Section 14 of the task)

Because the perfect-information optimal bot can always choose the trivial policy "take offer 1 whenever it's reached, decline nothing," its expected value can never be *less* than what that trivial policy earns:

**V_initial ≥ p₁ × M₁ = p₁ × S × W, for every valid choice of c ≥ 0 (any M_{j≥2} family, not just the one in Section D.2).**

Setting the required target EV = W×R against this floor:

**p₁ × S × W ≤ W × R ⟹ S ≤ R / p₁**

With **p₁ = 0.7420734269371773** (exact, unchanged combinatorial constant of the card process):

**S_max(R) = R / 0.7420734269371773**

Equivalently, for a fixed S, **R_min(S) = p₁ × S** — the configured Target RTP must be at least this, or the fixed first offer alone already guarantees more expected value than the target permits, and no choice of c (however small, including c=0, which drives every M_{j≥2} to $0) can bring V_initial back down to the target.

**This bound is universal** — it does not depend on the particular fairness-family choice in Section D.2; it applies to *any* conceivable way of filling in M₂, M₃, .... It is a property of M₁ and p₁ alone.

### E.2 Computed boundary values

| S | R_min(S) = p₁ × S |
|---:|---:|
| 1.05× | 0.779177 (77.9177%) |
| 1.10× | 0.816281 (81.6281%) |
| 1.15× | 0.853384 (85.3384%) |
| 1.20× | 0.890488 (89.0488%) |

### E.3 The upper bound (R_max(S)) — checked, not assumed away

A fixed Starting Multiplier does **not** meaningfully cap how high R can go, because M_{j≥2} can still be driven all the way to the $10,000 cap by increasing c. Numerically, at S=1.05, W=1200 as the numerator-scale (equivalently c large): V_initial(c) plateaus at **$6,199.81 (RTP ≈ 61,998%)** as c → ∞ — matching, almost exactly, p₂ × $10,000 = $6,199.81 (once c is large enough, the bot's best move at the j=1 decision is always CONTINUE, since every subsequent offer has saturated at the cap, so V_initial → p₁ × [continuation value] → p₂ × cap). **This ceiling is many orders of magnitude above any realistic configured RTP** (85%–97.5% in the required test range), so **R_max(S) is not a practically binding constraint anywhere in this task's configuration space** — the only feasibility boundary that matters in practice is the lower one, R_min(S), from Section E.1.

---

## F. Default production payout table (S = 1.05×, R = 90%)

**Solved: c\* = 0.8112024616978033** (bisection, tol 1e-12, k_max=1200). **V(initial) = $9.000000000000231, RTP = 90.00000000000232%.**

| Offer # (j) | p_j | Uncapped payout | Capped payout | Multiplier | Production payout (cent-rounded) |
|---:|---:|---:|---:|---:|---:|
| 1 | 0.7420734269 | $10.5000 (fixed = S×W) | $10.5000 | 1.0500× | **$10.50** |
| 2 | 0.6199813571 | $11.7759 | $11.7759 | 1.1780× | $11.78 |
| 3 | 0.5169028488 | $14.1242 | $14.1242 | 1.4120× | $14.12 |
| 4 | 0.4300441657 | $16.9769 | $16.9769 | 1.6980× | $16.98 |
| 5 | 0.3569975277 | $20.4506 | $20.4506 | 2.0450× | $20.45 |
| 6 | 0.2956911693 | $24.6907 | $24.6907 | 2.4690× | $24.69 |
| 7 | 0.2443454249 | $29.8791 | $29.8791 | 2.9880× | $29.88 |
| 8 | 0.2014341674 | $36.2442 | $36.2442 | 3.6240× | $36.24 |
| 9 | 0.1656509928 | $44.0735 | $44.0735 | 4.4070× | $44.07 |
| 10 | 0.1358796088 | $53.7301 | $53.7301 | 5.3730× | $53.73 |
| 11 | 0.1111679438 | $65.6738 | $65.6738 | 6.5670× | $65.67 |
| 12 | 0.0907055410 | $80.4893 | $80.4893 | 8.0490× | $80.49 |
| 13 | 0.0738038521 | $98.9220 | $98.9220 | 9.8920× | $98.92 |
| 14 | 0.0598790849 | $121.9261 | $121.9261 | 12.1930× | $121.93 |
| 15 | 0.0484372992 | $150.7273 | $150.7273 | 15.0730× | $150.73 |
| 16 | 0.0390614766 | $186.9059 | $186.9059 | 18.6910× | $186.91 |
| 17 | 0.0314003220 | $232.5079 | $232.5079 | 23.2510× | $232.51 |
| 18 | 0.0251585834 | $290.1921 | $290.1921 | 29.0190× | $290.19 |
| 19 | 0.0200886977 | $363.4293 | $363.4293 | 36.3430× | $363.43 |
| 20 | 0.0159835947 | $456.7697 | $456.7697 | 45.6770× | $456.77 |
| 21 | 0.0126705115 | $576.2058 | $576.2058 | 57.6210× | $576.21 |
| 22 | 0.0100056853 | $729.6674 | $729.6674 | 72.9670× | $729.67 |
| 23 | 0.0078698089 | $927.7001 | $927.7001 | 92.7700× | $927.70 |
| 24 | 0.0061641482 | $1,184.4008 | $1,184.4008 | 118.4400× | $1,184.40 |
| 25 | 0.0048072328 | $1,518.7161 | $1,518.7161 | 151.8720× | $1,518.72 |
| 26 | 0.0037320401 | $1,956.2550 | $1,956.2550 | 195.6260× | $1,956.26 |
| 27 | 0.0028836062 | $2,531.8375 | $2,531.8375 | 253.1840× | $2,531.84 |
| 28 | 0.0022170037 | $3,293.1033 | $3,293.1033 | 329.3100× | $3,293.10 |
| 29 | 0.0016956339 | $4,305.6595 | $4,305.6595 | 430.5660× | $4,305.66 |
| 30 | 0.0012897882 | $5,660.4812 | $5,660.4812 | 566.0480× | $5,660.48 |
| 31 | 0.0009754407 | $7,484.6397 | $7,484.6397 | 748.4640× | $7,484.64 |
| 32 | 0.0007332357 | $9,956.9916 | $9,956.9916 | 995.6990× | $9,956.99 |
| 33 | 0.0005476437 | $13,331.3371 | **$10,000.0000** | **1000.0000×** | **$10,000.00** |
| ≥33 | (shrinking) | (unbounded) | $10,000.0000 | 1000.0000× | $10,000.00 |

**Monotonicity:** M₁ ($10.50) ≤ M₂ ($11.78) ≤ M₃ ≤ ... holds throughout — offers increase smoothly at this configuration (see Section H for when this breaks).

---

## G. Tested configuration matrix (4 × 5 = 20 combinations)

Starting Multiplier ∈ {1.05×, 1.10×, 1.15×, 1.20×} × Target RTP ∈ {85%, 90%, 92%, 93.5%, 95%). Every feasible row was solved from scratch via bisection on c against the unmodified perfect-information backward induction (k_max=600, converged — see Section J for the convergence argument, which is horizon-independent of S/R).

| S | R | Feasible? | c\* | V(initial) | RTP | M₁ | M₂ | Monotonic (M₁≤M₂)? | First capped j | Last sub-cap j | R_min(S) |
|---:|---:|---|---:|---:|---:|---:|---:|---|---:|---:|---:|
| 1.05× | 85% | Yes | 0.790825 | $8.499999 | 85.0000% | $10.5000 | $10.8423 | **Yes** | 33 | 32 | 0.77918 |
| 1.05× | 90% | Yes | 0.811202 | $9.000000 | 90.0000% | $10.5000 | $11.7759 | **Yes** | 33 | 32 | 0.77918 |
| 1.05× | 92% | Yes | 0.817993 | $9.199999 | 92.0000% | $10.5000 | $12.1383 | **Yes** | 32 | 31 | 0.77918 |
| 1.05× | 93.5% | Yes | 0.820540 | $9.349999 | 93.5000% | $10.5000 | $12.3746 | **Yes** | 32 | 31 | 0.77918 |
| 1.05× | 95% | Yes | 0.821959 | $9.499999 | 95.0000% | $10.5000 | $12.5949 | **Yes** | 32 | 31 | 0.77918 |
| 1.10× | 85% | Yes | 0.758509 | $8.500000 | 85.0000% | $11.0000 | $10.3992 | **No** | 33 | 32 | 0.81628 |
| 1.10× | 90% | Yes | 0.795514 | $9.000000 | 90.0000% | $11.0000 | $11.5481 | **Yes** | 33 | 32 | 0.81628 |
| 1.10× | 92% | Yes | 0.806201 | $9.199999 | 92.0000% | $11.0000 | $11.9633 | **Yes** | 32 | 31 | 0.81628 |
| 1.10× | 93.5% | Yes | 0.817635 | $9.349999 | 93.5000% | $11.0000 | $12.3308 | **Yes** | 32 | 31 | 0.81628 |
| 1.10× | 95% | Yes | 0.821959 | $9.499999 | 95.0000% | $11.0000 | $12.5949 | **Yes** | 32 | 31 | 0.81628 |
| 1.15× | 85% | **No — infeasible** | — | — | — | — | — | — | — | — | 0.85338 |
| 1.15× | 90% | Yes | 0.767494 | $9.000000 | 90.0000% | $11.5000 | $11.1414 | **No** | 33 | 32 | 0.85338 |
| 1.15× | 92% | Yes | 0.782749 | $9.199999 | 92.0000% | $11.5000 | $11.6153 | **Yes** | 33 | 32 | 0.85338 |
| 1.15× | 93.5% | Yes | 0.797916 | $9.350000 | 93.5000% | $11.5000 | $12.0334 | **Yes** | 32 | 31 | 0.85338 |
| 1.15× | 95% | Yes | 0.808513 | $9.500001 | 95.0000% | $11.5000 | $12.3889 | **Yes** | 32 | 31 | 0.85338 |
| 1.20× | 85% | **No — infeasible** | — | — | — | — | — | — | — | — | 0.89049 |
| 1.20× | 90% | Yes (tight) | 0.680394 | $9.000001 | 90.0000% | $12.0000 | $9.8770 | **No** | 33 | 32 | 0.89049 |
| 1.20× | 92% | Yes | 0.752903 | $9.200000 | 92.0000% | $12.0000 | $11.1724 | **No** | 33 | 32 | 0.89049 |
| 1.20× | 93.5% | Yes | 0.764395 | $9.350001 | 93.5000% | $12.0000 | $11.5279 | **No** | 33 | 32 | 0.89049 |
| 1.20× | 95% | Yes | 0.777924 | $9.500000 | 95.0000% | $12.0000 | $11.9202 | **Yes** | 32 | 31 | 0.89049 |

**18 of 20 combinations are feasible; 2 are not** — exactly, and only, the two combinations where R < R_min(S) (Section E): (S=1.15×, R=85%) requires R≥85.338%, and (S=1.20×, R=85%) requires R≥89.049%. Both failures are exactly, quantitatively explained by the closed-form bound in Section E.1 — not a numerical search failure, a horizon artifact, or a cap issue.

---

## H. Monotonicity — does it emerge naturally, or does it have to be imposed?

**It emerges naturally in most of the tested region, and breaks down in a precisely characterized, mechanistically understood corner — and it was never imposed by force anywhere in this document.**

The mechanism, made exact: for j ≥ 2, M_j(c) = min(c×TARGET_EV/p_j, cap), so **M₂(c) is directly proportional to c**. As R decreases toward R_min(S) for a fixed S (i.e., as the fixed M₁ = S×W absorbs a larger and larger share of the entire target EV), the required c shrinks toward 0 (Section E.1: at R = R_min(S) exactly, c = 0 exactly, since then V_initial = p₁×M₁ = TARGET_EV already, with every M_{j≥2} = $0). Since M₂(c) ∝ c while M₁ is fixed and does not shrink, **M₂ must eventually fall below M₁ as R approaches R_min(S) from above, for any S large enough that this region is reachable within the tested R range.**

This was fully confirmed computationally (Section G) and located precisely:

| S | Monotonicity crossover R\* (where M₁ = M₂ exactly) | R_min(S) |
|---:|---:|---:|
| 1.05× | below 85% (not reached in the tested grid) | 77.918% |
| 1.10× | ≈ 87.557% | 81.628% |
| 1.20× | ≈ 94.988% | 89.049% |

**For R > R\*(S), monotonicity holds without being imposed. For R_min(S) ≤ R < R\*(S), it fails — M₂ < M₁ — as a direct, provable consequence of the fixed-M₁ architecture, not a bug or an oversight.**

**Was it imposed anywhere in this document? No.** No M_j was floored at M₁, clamped, smoothed, or reordered anywhere in the derivation or the results above — every number in Section F and G is the unmodified output of the root-find. **Does imposing it change feasibility?** Not tested here, and it should not be, without approval: forcing M₂ ≥ M₁ when the unconstrained solution wants M₂ < M₁ would mean paying out more than the fairness-calibrated family requires at offer 2, which would push V_initial above the target R and require the calibration to compensate elsewhere (e.g. a smaller c, pushing later offers down further, or accepting RTP above target) — a new constraint with new consequences that has not been derived or approved. **Does it conflict with the cap?** No interaction was found between the monotonicity dip (which occurs at small j, small c, values well under $100) and the cap (which only ever binds at large j, values near $10,000) — the two phenomena occur in disjoint regions of the curve in every tested configuration.

**This is reported as a finding requiring a design decision, per Section 22:** if the game experience requires strictly increasing offers, the (S, R) combinations in the shaded region above (roughly: S ≥ 1.10× combined with R below its crossover R\*(S)) need either a different Starting-Multiplier/RTP pairing, or an explicit, newly-approved rule for how to handle a non-monotonic region (e.g., accepting it, since nothing in the 14 immutable game rules requires monotonicity in the first place — the "increasing multiplier feel" was already noted as non-binding as far back as Document 04).

---

## I. Cap boundary across configurations

The first-capped-offer index moves between **j = 32 and j = 33** across the entire tested grid (Section G's table) — never more than one offer earlier or later than the reference CASE 1 boundary (05A's j = 33, at S implicitly non-economic and R = 90%). The pattern is clean: **higher R** (holding S fixed) pushes the cap boundary one offer earlier (from j=33 to j=32), because a larger target EV requires larger c, which reaches the $10,000 ceiling sooner in the offer sequence; S has comparatively little effect on the cap boundary (it only ever changes the fixed M₁, which the cap never binds on in any tested configuration, since M₁ = S×W never remotely approaches $10,000 for realistic S). **No off-by-one error was found in any of the 18 feasible configurations** — in every case, M_j at the last sub-cap offer is strictly below $10,000 and M_j at the first capped offer is exactly $10,000.00 (both confirmed to the cent).

---

## J. Infinite-horizon verification

The rigorous tail bound from 05A — |V_true − V_k| ≤ P(alive at k) × (C×W) — was reverified for both the default configuration and the tightest-margin representative alternate (S=1.20×, R=90%, the configuration closest to its feasibility floor and the only one tested with a materially different optimal-policy shape, per Section H):

**Default (S=1.05×, R=90%):**

| k | P(alive at k) | Tail bound |
|---:|---:|---:|
| 50 | 8.287×10⁻⁴ | $8.287 |
| 60 | 6.748×10⁻⁵ | $0.6748 |
| 70 | 1.352×10⁻⁷ | $0.001352 |
| 80 | 6.352×10⁻¹³ | $6.352×10⁻⁹ |
| 90 | 1.226×10⁻¹⁴ | $1.226×10⁻¹⁰ |
| 100 | 1.226×10⁻¹⁴ | $1.226×10⁻¹⁰ |
| 104 | 0 (fp underflow) | $0 |

**Near-boundary alternate (S=1.20×, R=90%):**

| k | P(alive at k) | Tail bound |
|---:|---:|---:|
| 50 | 2.489×10⁻⁴ | $2.489 |
| 60 | 2.068×10⁻⁵ | $0.2068 |
| 70 | 1.571×10⁻⁷ | $0.001571 |
| 80 | 2.916×10⁻¹² | $2.916×10⁻⁸ |
| 90 | 1.107×10⁻¹⁴ | $1.107×10⁻¹⁰ |
| 100 | 1.107×10⁻¹⁴ | $1.107×10⁻¹⁰ |
| 104 | 0 (fp underflow) | $0 |

**Both configurations converge to a truncation error below $10⁻¹⁰ by k_max ≥ 90**, matching 05A's default-configuration finding almost exactly. Starting Multiplier and Target RTP change *when* and *how eagerly* the bot cashes out (Section G/H), but do not measurably change how quickly the shoe-exhaustion tail vanishes — both configurations remain governed by the same underlying fact (established in 05A) that surviving 100+ cards without busting is astronomically unlikely regardless of the payout schedule laid on top of that card process.

---

## K. Cent-rounding audit

**Default (S=1.05×, R=90%):** full backward induction re-run with every M_j (including M₁ = $10.50, already an exact cent value) rounded to the cent. V(initial) moves from $9.000000000000231 to **$8.999953920719017** (RTP 89.99954%, a shift of **−0.00046 percentage points**). All **734,749** saved (position, state) decisions were compared between the exact-precision and cent-rounded policies: **zero decision flips.**

**Representative alternate (S=1.20×, R=90%):** V(initial) moves from $9.00000057525519 to **$9.00000279212048** (RTP 90.0000279%, a shift of **+0.0000222 percentage points** — smaller than the default case, though still immaterial either way). Again, **734,749 states compared, zero decision flips.**

In both cases the rounding effect is a small, fully-explained, non-adversarial shift with no behavioral change in the bot's policy — consistent with every prior document in this project.

---

## L. Perfect-information optimal-bot RTP results

Every feasible cell in Section G's matrix reports the exact optimal-bot RTP achieved (the "RTP" column), each matching its configured target R to 4+ decimal digits before rounding (residual attributable to the bisection tolerance used for the batch sweep, tol=1e-6; the default configuration was separately re-solved to tol=1e-12, landing at 90.00000000000232%). No configuration was forced, tuned, or approximated to hit its target after the fact — every value in the table is the direct output of one root-find per (S,R) pair.

---

## M. Independent verification (default configuration)

A structurally independent implementation (the same recursive, single-shoe, top-down-memoized re-derivation introduced in 05A, `independent_reverify.py`, sharing no code path with `dynrtp_backward.py`/`capped_model.py`/`econ_starting_mult.py`) was used to re-derive, from scratch, every headline quantity for S=1.05×, R=90%:

- **p_j:** matches the position-indexed pipeline to 1.4×10⁻¹⁵ or better for every j tested (1, 2, 5, 10, 20).
- **V(initial) at the SAME M dict:** independent recursion gives 9.000000000000231 — **identical to the last representable bit** to the original pipeline's value.
- **c\*, independently re-solved via bisection on the independent recursive V function:** 0.8112024616857525, versus the original pipeline's 0.8112024616978033 — **absolute difference 1.2×10⁻¹¹**, fully attributable to differing bisection tolerances, not a discrepancy.

This directly rules out a shared-implementation bug behind the default configuration's headline numbers, exactly as in 05A.

---

## N. Monte Carlo reconciliation

**Default (S=1.05×, R=90%), production cent-rounded table, seed=20260918, N=3,000,000, real 104-card shoes, full-history optimal bot:**

| Quantity | Analytic (cent-rounded) | Monte Carlo |
|---|---:|---:|
| EV | $9.000 (RTP 89.9995%) | $9.088327 (RTP 90.8833%) |
| SE(EV) | — | $0.100674 |
| **z-score** | — | **+0.8778** |
| P(bust) | (implied 0.7109) | 0.7108 |
| P(cash) | (implied 0.2891) | 0.2892 |
| E[cards] | 7.0046 | 7.0038 |
| E[offers] | 2.9788 | 2.9791 |

**Representative alternate (S=1.20×, R=90%), same protocol:**

| Quantity | Analytic (cent-rounded) | Monte Carlo |
|---|---:|---:|
| EV | $9.0000 (RTP 90.0000%) | $9.035164 (RTP 90.3516%) |
| SE(EV) | — | $0.048301 |
| **z-score** | — | **+0.7280** |
| P(bust) | — | 0.3475 |
| P(cash) | — | 0.6525 |
| E[cards] | — | 3.3899 |
| E[offers] | — | 1.1968 |

Both z-scores are comfortably within ordinary sampling noise (|z| < 1). Note the sharply different game *feel* between the two configurations, correctly reproduced by the simulator: at S=1.20×/R=90% the bot cashes out roughly 2.25× more often (65% vs 29%) and in under half the average cards (3.4 vs 7.0), because the fixed first offer is large enough, relative to the (necessarily smaller, per Section H) continuation value, that taking early is very often optimal — a direct, mechanistic consequence of the architecture, not a simulation artifact.

---

## O. Final statement: which case applies

### CASE B — CONFIGURABLE WITH A MATHEMATICALLY DEFINED VALID RANGE

The system **M₁ = S×W, V_initial(S,R,{M_j}) = W×R** is mathematically feasible, solvable by the single-scalar root-finding method derived in Section D, and verified against the perfect-information optimal bot — but **not for every (S, R) pair.** The valid range is exact and closed-form, not empirical:

**R ≥ p₁ × S, i.e. S ≤ R / p₁, where p₁ = 0.7420734269371773.**

This is not a limitation of the solver, the cap, or the horizon treatment — it is a hard consequence of fixing M₁ = S×W as an economic anchor at all, discovered by direct calculation (Section E.1) and confirmed exactly by the two infeasible cells in the 20-cell test matrix (Section G): (S=1.15×, R=85%) and (S=1.20×, R=85%), each failing by precisely the margin the closed-form bound predicts. Within the valid range, the system is fully configurable: 18 of 20 tested combinations solve cleanly, hit their target RTP exactly (to the precision of the root-finding tolerance used), respect the $10,000 cap with no off-by-one behavior, survive cent-rounding with zero decision flips, and reconcile against fresh, real-deck Monte Carlo simulation within ordinary statistical noise.

A second, independent constraint — **not a feasibility failure, but a design-relevant boundary requiring explicit sign-off (Section H)** — is that strict monotonicity of the offer sequence (M₁ ≤ M₂ ≤ ...) is not guaranteed everywhere inside the feasible region: it holds for R comfortably above R_min(S), and fails in a precisely bounded corner near the feasibility floor (quantified per-S in Section H's crossover table). This was discovered, not hidden, and is reported for design approval rather than silently corrected.

---

## Production requirement

**DEFAULT PRODUCTION CONFIGURATION: Starting Multiplier = 1.05×, Target RTP = 90%.** Its complete, cent-rounded production table is Section F, generated by the process in Section A/D — the production team implements that table directly and does not compute p_j or c\* at runtime.

**For any other operator-configured (S, R) pair:** first check S ≤ R / p₁ (Section E.1); if satisfied, regenerate the payout table by the identical process (compute p_j — unchanged, cached once — fix M₁ = S×W, bisect for c against V_initial = W×R, apply the $10,000 cap, round to cents, re-verify zero decision flips) exactly as done for every cell in Section G. If violated, the configuration is mathematically infeasible under the current architecture and must be rejected or revised (raise R, lower S, or obtain approval for a different design, per Section D.1's discussion of the alternative, unused degrees of freedom) — it must not be silently forced to 90% or any other value.

**Design sign-off still needed (not resolved unilaterally here, per Section 22):** whether strict monotonicity of the offer ladder is a hard requirement for every operator-configurable (S, R) pair, or whether the region identified in Section H (large S paired with R near its floor) is acceptable as-is or excluded from the configurable range entirely.
