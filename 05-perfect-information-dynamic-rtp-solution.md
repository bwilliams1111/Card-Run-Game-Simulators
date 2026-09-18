# 05 — Perfect-Information Dynamic RTP Solution with a 1,000× Cap

**Status:** Final. Derived analytically, cross-validated by three independent computational methods, and stress-tested against an adversarial audit checklist.

**Scope note (naming):** This document is deliberately a *new, separate* deliverable from `05-actual-observable-dynamic-rtp-solution.md`. That earlier document answered a different question — whether a dynamic payout curve can control a player restricted to the four-card *visible* history. This document answers the harder question posed here: whether a dynamic payout curve, capped at 1,000×, can control a player with **perfect information** (full history, exact remaining shoe composition). Per the task brief, the four-card display is treated purely as a UI decision with no defensive value; the adversary modeled below always has complete information.

---

## 0. Executive summary

**Yes.** A 1,000×-capped dynamic payout function exists that gives the perfect-information optimal bot **exactly 90.000% RTP** (V(initial) = $9.00 on a $10 wager), subject to 0 ≤ M(state) ≤ $10,000 for every state. This is **CASE 1** of the three-case framework required by the task (Section 17 below gives the full argument).

The construction is a single global scalar calibration:

M_j(c) = min( c · $9.00 / p_j , $10,000 ), with **c\* = 0.8125887457281351**

found by bisection (root-finding on one monotonic, continuous scalar — not manual per-state tuning) such that the perfect-information-optimal value at the initial state equals exactly $9.00.

The cap is not a cosmetic afterthought: it is what makes the whole problem well-posed. Without it, 04's finding stands — V(initial) diverges to infinity as the horizon grows, for *any* positive scaling constant — so no finite calibration exists at all. With the cap in place, V(initial; c) is bounded, continuous, and monotonic in c, which is exactly what makes the root-finding argument valid (Section 6).

The residual bot exploit (composition-driven continuation value) is not eliminated by the cap — it is only capped in magnitude, and it becomes irrelevant in practice: the state at which the cap first binds is reached with probability ≈1.98×10⁻¹³ under optimal play, and actually cashed out at with probability ≈1.37×10⁻¹⁴. Section 12 quantifies the exploit exactly.

A separate and important finding, required to be reported honestly by the task brief: the SAME calibrated curve that gives the perfect-information bot exactly 90% RTP gives an *ordinary threshold player* (someone who just decides in advance "I will take the j₀-th offer I ever see," ignoring composition) only **73.133% RTP** for every threshold at or below the cap boundary, declining further above it. Controlling the worst-case adversary costs ordinary players RTP. This is documented in full in Section 16 (Question A).

---

## 1. Assumptions

- The $10 wager and the two-52-card-deck (104-card) shoe are fixed, as specified.
- "RTP" means expected player return divided by wager, expressed as a percentage: RTP = V(initial state) / $10 × 100%.
- The adversary is the single worst case the game must be safe against: a player with perfect information (exact nc, ns, nr, prev, j, and the complete payout function) playing the exact backward-induction-optimal policy. Any weaker player (including one restricted to the 4-card visible window, as in the companion `05-actual-observable...` document) earns RTP no higher than this bot's, because the bot's policy is optimal by construction. Solving for this bot therefore solves the safety question for every realistic player.
- All probabilities are computed under a uniform-random shuffle of the 104-card shoe (two standard 52-card decks, no jokers), independently for each fresh shoe and each reshuffle.
- "Reshuffle" means: the shoe is fully exhausted (104 cards dealt) without the run having busted or cashed out. A fresh 104-card shoe is shuffled and dealing continues, but the run itself (cumulative card index / position, prev, and j) is **not** reset — only the composition counters (nc, ns, nr) reset to (26, 26, 52).
- Money amounts are computed at full floating-point precision in the analytical model; a separate check (Section 14) quantifies the effect of rounding every payout to the cent, which is how the game would actually be implemented.
- The 1,000× cap means M(state) ≤ 1000 × $10 = $10,000 for every state, with equality permitted (the cap is inclusive, per the ≤ in the brief).

---

## 2. Immutable game rules (restated, unchanged)

1. Two standard 52-card decks are combined into one 104-card shoe and shuffled before the run begins.
2. Cards are dealt one at a time from the shoe.
3. If the shoe is exhausted (all 104 cards dealt) without the run having busted or cashed out, a fresh 104-card shoe is shuffled and dealing continues within the **same run**: the card/multiplier progression, the offer count j, and the previous-card category all carry through the reshuffle. Only nc, ns, nr reset to 26, 26, 52.
4. **Card 1 special rule:** if card 1 is the Ace of Spades, the run busts immediately (P = 2/104, since there are two Aces of Spades across the two decks). No offer is made and no cash-out is possible on card 1 under any circumstance.
5. From card 2 onward, categorize every dealt card as Club (C), Spade (S), or Red (R; hearts or diamonds — 52 of the 104 cards).
   - A **red** card always produces an offer to the player (unless it is card 1, per rule 4 — card 1 never offers).
   - A **black** card (club or spade) never produces an offer.
6. Bust rules, applied to the category of the current card versus the category of the immediately preceding card (`prev`):
   - Club immediately after Club → bust.
   - Spade immediately after Spade → bust.
   - Club after Spade → safe.
   - Spade after Club → safe.
   - Any black card after a red card → safe.
   - Any red card after a black card → safe.
   - Red immediately after Red → safe (reds never bust each other).
7. On bust, the run's value becomes $0, immediately and irrevocably.
8. At a red-card offer, the player chooses:
   - **TAKE:** receive the payout assigned to this offer, and the run ends.
   - **CONTINUE:** decline the offer. The red card becomes the new `prev`. The next card is dealt. The offer count j increments by 1.
9. Base wager: $10.

No rule above has been altered, relaxed, or reinterpreted from the original specification.

---

## 3. State model

The minimal state sufficient for a perfect-information optimal decision is:

**S = (nc, ns, nr, prev, j)**

where:

- `nc`, `ns`, `nr` — number of clubs, spades, and reds remaining in the current shoe (post-reshuffle-reset when applicable).
- `prev` ∈ {C, S, R} — category of the immediately preceding dealt card (needed only to evaluate the bust rule for the *next* black card; irrelevant for reds, which never bust).
- `j` — number of red-card offers already declined in this run (i.e. the offer about to be decided is offer j+1).

No additional state variable was found to be mathematically necessary. In particular:

- **Absolute card index / position (k)** is *not* part of the sufficient state for value or decision purposes — the value of a state depends only on (nc, ns, nr, prev, j), not on how many cards it took to arrive there. Position is used only as a computational bookkeeping index (for the backward induction sweep and for tracking reshuffle timing), never as an argument to V(·) or M(·) in the final derivation.
- **Which specific reshuffle cycle** the run is on is *not* separately tracked, because a reshuffle is defined purely as "nc = ns = nr = 0 → reset to (26, 26, 52)"; the reset is a deterministic function of the state itself, not of external history.
- **The identity of specific past cards** (beyond what is summarized in nc, ns, nr, prev) carries zero additional information relevant to future probabilities, since the shoe is uniformly shuffled — this is the standard sufficiency argument for a card-counting state in a memoryless-shuffle model.

A reshuffle transition is: if nc + ns + nr = 0 immediately after a card is dealt (i.e., the shoe is now empty and no further card can be drawn from it), the *next* card is dealt from a freshly shuffled 104-card shoe, i.e., the transition function first resets (nc, ns, nr) → (26, 26, 52) before drawing. `prev` and `j` are **not** reset. This is implemented identically in all three independent code paths used in this document (the backward-induction dictionaries, the exact forward-under-policy propagation, and the real-deck Monte Carlo simulator), and is called out explicitly in each to guard against the "accidentally reset the run" failure mode the task brief warned about.

---

## 4. The three players compared (baseline vs. this document's target)

The task requires explicitly comparing three notions of "player," not conflating them:

| Player | Information used | What determines its decisions |
|---|---|---|
| (A) j-only player | Only the offer count j (a fixed pre-committed threshold, or any fixed function of j alone) | A rule of the form "take the j₀-th offer," decided before the run starts, blind to composition |
| (B) Perfect-information optimal bot | Exact (nc, ns, nr, prev, j) at every decision, plus the complete payout function | Full backward-induction optimal policy — the actual subject of this document |
| (C) Capped payout curve | — (this is the *object* being designed, not a player) | M(state), designed subject to 0 ≤ M(state) ≤ $10,000 |

04's formula, M_j = $9/p_j, is used here **only as an uncapped, unscaled baseline reference** — never assumed final. It is shown in Section 5 to give exactly 90% RTP to player (A) but to *diverge* under player (B)'s optimal exploitation of composition (04's finding, reconfirmed below), and it violates the $10,000 cap almost immediately (Section 8). The entire point of this document is to derive a *different* curve, M(state; c\*), calibrated against player (B), not player (A), and to determine whether that curve can simultaneously respect the cap.

---

## 5. Why payout must depend on j (not just be a flat number), and why j alone is not enough for the *bot*

**Dependence on N (absolute card index) or full composition, versus dependence on j alone**, was investigated as required (Section 9 of the brief: A, B, C, D, E).

- **(A) j only:** A payout schedule that depends only on j, M_j, is well-defined and is what determines the "j-only naive player's" RTP (Section 4, player A). It is *insufficient* to control the perfect-information bot on its own — the bot, at fixed j, still has continuation-value information available from (nc, ns, nr, prev), and can compare M_j to a continuation value that varies with composition even though the offer/payout itself doesn't.
- **(B) Absolute card index N:** Tested and rejected as a state variable for V(·)/M(·) (Section 3) — it is not part of the minimal sufficient state, since the transition probabilities and future payouts depend only on remaining composition and offer count, not on how many cards got you there. (N *is* used as a bookkeeping index for the sweep, not as a value-relevant argument.)
- **(C) Exact composition (nc, ns, nr):** This *is* what determines the bot's continuation value, but it is deliberately **not** used as an argument to M(state) itself. M depends only on j (Section 6 below) — the composition-dependence enters entirely through the bot's own optimal continuation-value calculation (V, computed via backward induction), not through the payout schedule. This is the key design choice, justified next.
- **(D)/(E) Some other derived value / combination:** Investigated and found unnecessary — see the next paragraph for why M_j alone (not M(nc, ns, nr, j)) suffices to achieve exactly 90% against the bot.

**Why M_j (dependent on j only) is sufficient, and the simplest representation that works:** The bot's decision at a red-card offer is TAKE vs. CONTINUE, i.e. compare M_j to a continuation value V_cont(nc, ns, nr, prev, j) computed via backward induction from the *existing* M_j schedule. Nothing requires M itself to depend on composition for the *resulting* V(initial) to hit any particular target — what's required is that the *scale and cap* of the M_j-schedule, taken as a whole, are calibrated so that when the bot optimizes over TAKE/CONTINUE at every state (fully exploiting composition on the continuation side), the total expected value at the initial state comes out to exactly $9.00. This is a strictly easier and more transportable design than making the payout table itself composition-dependent (which would require publishing, and the player inferring, a payout that changes with remaining deck composition — a substantial practical/game-design complication flagged separately in Section 16 as the cost of this approach, not hidden). The document therefore adopts **M depending on j only**, i.e. representation (A) from the brief's Section 9, as "the simplest payout representation that can actually control the perfect-information optimal player" — the control is exerted through calibrating the *scale* of the schedule against the bot's own optimal use of composition information, not by chasing composition with the payout table.

---

## 6. Payout function derivation

### 6.1 The capped, scaled family

Define, for a single scalar c > 0:

**M_j(c) = min( c × $9.00 / p_j , $10,000 )**

where p_j = P(a run, played passively — i.e. with a "dealer" that never intervenes — ever reaches at least j offers), computed exactly via the same exhaustive forward enumeration used in document 04 (`dynrtp_forward_full.forward_pass_full`, unchanged).

### 6.2 Value function and Bellman recursion

For state S = (nc, ns, nr, prev, j):

- **TAKE_VALUE(S) = M_j₊₁(c)** (the payout for the offer currently being decided is for offer number j+1, since j counts offers *already declined*).
- **CONTINUE_VALUE(S) = E[V(S′) | S, decline]**, the expected value over the next dealt card's category, of the resulting state (bust → 0; safe black → deterministic transition; the recursion bottoms out because black cards have no decision).
- **V(S) = max(TAKE_VALUE(S), CONTINUE_VALUE(S))** at every red-card decision point; for black-card and terminal (bust) transitions, V is just the corresponding branch value with no choice.

This recursion is implemented, unchanged from document 04, in `dynrtp_backward.solve_backward(k_max, M, dist_by_k)`, which takes an arbitrary M dict — no modification to the backward-induction machinery itself was needed to support capping, since capping is purely a property of the M dict that is passed in.

### 6.3 Monotonicity and continuity in c (why a root exists)

**Claim:** For fixed cap, V(initial; c) is continuous and non-decreasing in c, with V(initial; 0) = 0 and V(initial; c) → p₁ × $10,000 = $7,420.73 as c → ∞ (all confirmed numerically, Section 6.5).

**Argument:**
- For every j, M_j(c) = min(c × $9/p_j, $10,000) is non-decreasing and continuous in c (a min of a non-decreasing continuous function of c and a constant is itself non-decreasing and continuous).
- V(S; c), for every state S, is defined by a finite backward induction that is a composition of `max` and expectation (a non-negative linear combination) operations applied to the M_j(c) values. `max` of non-decreasing continuous functions is non-decreasing and continuous; a non-negative linear combination (expectation) of non-decreasing continuous functions is non-decreasing and continuous. By induction over the (finite, for any truncation k_max) backward sweep, V(S; c) is non-decreasing and continuous in c for every S, including the initial state.
- As c → ∞, every M_j(c) that has any finite j eventually saturates at the cap (since c × $9/p_j → ∞ while the cap stays fixed at $10,000), so for c large enough, M_1(c) = $10,000 already, making "always take the very first offer" trivially weakly optimal at the initial decision (nothing can beat taking $10,000 immediately when nothing else can exceed the cap) — giving V(initial; c → ∞) = p₁ × $10,000.
- V(initial; 0) = 0 trivially (every M_j(0) = 0).

Since $9.00 ∈ [0, $7,420.73], the Intermediate Value Theorem guarantees a root c\* with V(initial; c\*) = $9.00 exists. Monotonicity additionally guarantees this root is reachable by bisection and (given strict monotonicity, verified numerically — see 6.5) unique.

### 6.4 Why this is root-finding, not "manually reducing payouts until it works"

The task explicitly prohibits "calculate an optimal value first and then manually reduce payouts until the result reaches $9." The construction here is categorically different: **c is the single free parameter of an explicit closed-form family**, and its value is obtained by solving one scalar equation, V(initial; c) = $9.00, via bisection — a standard, mechanical numerical root-finding procedure with a provable existence/uniqueness argument (Section 6.3), not an iterative trial-and-error adjustment of individual payout rungs. Every M_j for every j is determined by the *same* c\* simultaneously; no per-state or per-j hand-tuning occurs anywhere in the construction.

### 6.5 Numerical result

Root-finding (`capped_model.find_c_star`, bisection, tolerance 1e-8) at k_max = 1600 gives:

**c\* = 0.8125887457281351**

**V(initial; c\*) = $9.000000002319613** → **RTP = 90.00000002319613%**

Convergence in k_max was checked explicitly (not assumed) at c\*, from k_max = 60 up to k_max = 3200:

| k_max | V(initial; c\*) | RTP |
|---:|---:|---:|
| 60 | $8.839006 | 88.3901% |
| 100 | $9.000000002 | 90.0000000% |
| 150 | $9.000000002 | 90.0000000% |
| 200 | $9.000000002 | 90.0000000% |
| 400 | $9.000000002 | 90.0000000% |
| 800 | $9.000000002 | 90.0000000% |
| 1600 | $9.000000002 | 90.0000000% |
| 3200 | $9.000000002 | 90.0000000% |

The value is not merely "close" at large horizons — it plateaus to 9-significant-digit stability from k_max = 100 onward and does not move at all through k_max = 3200. This is direct numerical evidence (not an assumption) that the infinite-horizon value equals $9.000000002... (Section 9 gives the accompanying probabilistic argument for *why* this plateau must occur, rather than treating the plateau alone as sufficient proof).

For contrast, at c = 1 (i.e. the capped curve with no rescaling), the same convergence check gives a *different*, also-stable plateau:

| k_max | V(initial; c=1) | RTP |
|---:|---:|---:|
| 50 | $10.2587 | 102.5865% |
| 100 | $10.9299 | 109.2993% |
| 200 | $10.9299 | 109.2993% |
| 400 | $10.9299 | 109.2993% |
| 800 | $10.9299 | 109.2993% |
| 1600 | $10.9299 | 109.2993% |
| 3200 | $10.9299 | 109.2993% |

This is itself an important finding, developed fully in Section 9: **capping alone converts the uncapped, divergent 04 problem into a convergent one** — at c=1 the value plateaus at a finite 109.2993% instead of diverging without bound (04 reached 1,035.67% RTP by k_max = 800 with no sign of leveling off). The remaining work, done via c\*, is to bring that finite plateau down to exactly 90%, not to fix divergence (the cap already fixed that).

---

## 7. Homogeneity: why the *uncapped* problem could never be solved by scaling alone

A necessary companion result, since the task requires explaining *why* capping matters mathematically, not just asserting it: in the **absence** of a cap, the Bellman recursion is exactly homogeneous of degree 1 in the M-curve. That is, if M_j(c) = c × (base curve) with no cap, then V(S; c) = c × V(S; 1) for every state S and every c > 0 — every term in the recursion (TAKE_VALUE, CONTINUE_VALUE, the max, the expectation) scales linearly with c when there is no min(·, cap) breaking the linearity.

04 established that V(initial; 1) diverges to infinity as the horizon grows (no plateau, unbounded growth). By homogeneity, V(initial; c) = c × V(initial; 1) = c × ∞ = ∞ for **every** c > 0 in the uncapped case. There is therefore no scalar c, however small, that makes an uncapped curve of this family finite, let alone equal to $9.00. **This proves capping is not an optional design choice but a mathematical necessity** for any finite infinite-horizon RTP target to be achievable at all with this family of payout curves. The 1,000× cap specified in this task is not merely compatible with solving the RTP problem — it is what makes the RTP problem solvable in the first place.

---

## 8. The 1,000× ($10,000) cap: where it binds, and what happens there

The payout curve table below (full detail in Section 13) shows the cap first binding at **j = 33**: M₃₂(c\*) = $9,974.01 (still under the cap), M₃₃(c\*) would be $16,434.04 uncapped but is clamped to exactly $10,000.00, and every M_j for j ≥ 33 is $10,000.00.

The task's explicit warning — "do not assume simple clipping of the 04 curve produces 90% RTP" — was heeded: clipping the *unscaled* 04 curve (M_j = $9/p_j, capped at $10,000) at c = 1 was computed exactly above (Section 6.5) and gives **109.2993% RTP, not 90%**. Simple clipping alone overshoots the target by more than 19 percentage points. The scale factor c\* was then derived (not assumed) to correct this, and the *combination* of capping and scaling is what achieves exactly 90.000%. This directly answers the brief's requirement: **yes**, the capped curve *can* be mathematically adjusted to restore exactly 90% RTP, via the single scalar c\* derived in Section 6.

---

## 9. Infinite-horizon treatment (not relying on finite truncation as the final answer)

Two independent arguments are given, per the task's requirement that a diagnostic table alone is not sufficient evidence of convergence/boundedness:

**Argument 1 — boundedness is guaranteed by construction, independent of any convergence table.** Because M(state) ≤ $10,000 for every state by the cap, and V(S) = max(TAKE, CONTINUE) is always a probability-weighted average/max of payouts each individually ≤ $10,000, it follows by straightforward induction on the backward recursion that **V(S) ≤ $10,000 for every reachable state S, at every finite horizon, and hence in the limit.** This is a hard mathematical guarantee, not an empirical observation — no diagnostic table is needed to establish that the value cannot diverge; the cap makes unbounded growth structurally impossible. (Contrast with the uncapped case, Section 7, where the value's homogeneity in c means boundedness is *never* guaranteed regardless of horizon.)

**Argument 2 — the value is not merely bounded but exactly stationary beyond a finite horizon, because the probability of the run surviving that long vanishes.** Under the optimal policy at c\*, the exact (non-Monte-Carlo) forward-under-policy propagation (`capped_policy_forward.py`) computes the probability mass still "alive" (neither busted nor cashed out) at horizon k:

| k (cards dealt) | P(still alive at k) |
|---:|---:|
| 50 | 9.9617 × 10⁻⁴ |
| 100 | 9.7133 × 10⁻¹⁵ |
| 104 | 0 (below double-precision floor) |
| 105 | 0 |

Since the shoe has exactly 104 cards, "alive at k=104" is a necessary condition for a reshuffle to ever occur, and this probability is already indistinguishable from zero in double precision. **The probability of the game ever reaching a second shoe (a reshuffle) is on the order of 10⁻¹⁵ or smaller** — not exactly provably zero (the true value is a positive real number below floating-point resolution, since in principle an astronomically improbable run of alternating safe cards can survive arbitrarily long), but zero to any precision that matters for RTP, game design, or auditing purposes.

**This directly explains — rather than merely asserting — why V(initial) plateaus by k_max ≈ 100–104 in the table above**: it plateaus because the probability of the run needing information from beyond that horizon (including reshuffle behavior) is already ~10⁻¹⁵, so extending the horizon further literally cannot move the expectation by more than that order of magnitude. This is the "mathematically justified convergence method" the brief requires in place of relying on the table alone: the table is corroborating evidence, and Argument 2 is the reason the corroboration holds. A naive `E[reshuffles] ≈ E[cards]/104` approximation was tested and explicitly rejected as invalid here (see Section 11) — the correct reasoning is the direct "alive mass at k=104" computation above, not that shortcut.

**Conclusion:** the infinite-horizon value is $9.000000002319613 (RTP 90.00000002%) to the full precision the model supports, established both by a hard structural bound (Argument 1) and by an explicit vanishing-tail argument (Argument 2), not by finite-horizon table-reading alone.

---

## 10. Perfect-information bot behavior and the backward-induction implementation

The bot is modeled with **no restriction whatsoever** to thresholds, heuristics, or "reasonable" play. At every red-card decision, the implemented solver (`capped_backward_full_policy.solve_and_save_policy`, which runs the identical Bellman recursion as `dynrtp_backward.solve_backward` but additionally records the decision and both branch values at every position from 1 through k_save = 400) computes:

- TAKE_VALUE(S) = M_{j+1}(c\*)
- CONTINUE_VALUE(S) = the exact expectation, under the true remaining composition (nc, ns, nr), of continuing — i.e., the bot is given the exact same information the game engine itself has: exact counts of clubs, spades, and reds left, the previous card's category, the number of offers already declined, and full knowledge of the payout schedule.
- The solver picks whichever is larger — **the solver itself determines optimal play**, with no externally imposed threshold, heuristic, or simplification.

Black-card transitions are handled exactly as specified: bust branch contributes 0 to the expectation, safe branch continues into the deeper state with unchanged j. Reshuffle boundaries are handled by resetting (nc, ns, nr) → (26, 26, 52) whenever they reach 0, while carrying prev and j through unchanged, exactly matching the state-model definition in Section 3 (and cross-checked three separate times, in the backward induction's dict handling, the forward-under-policy propagation, and the real-deck Monte Carlo — see Section 3's reshuffle-transition note).

Because computing to a literally infinite horizon is not possible, a finite truncation (k_max, with V = 0 imposed beyond it as a boundary condition) is used, exactly as required — but Section 9 both demonstrates the resulting value's convergence as k_max grows (the table in Section 6.5, stable across a 32× range of k_max from 100 to 3200) and *explains* why the approximation is valid (the vanishing-survival-probability argument), rather than resting on convergence alone.

---

## 11. Required results table

All values below are for the calibrated curve M_j(c\*), c\* = 0.8125887457281351, evaluated at the perfect-information optimal policy, at k_max = 1600 (converged; identical to k_max = 3200 to displayed precision).

| Metric | Value |
|---|---:|
| Initial optimal value V(initial) | $9.000000002 |
| Initial optimal RTP | 90.00000002% |
| Maximum payout (hard cap) | $10,000.00 (1,000×) |
| First-offer payout M₁(c\*) | $9.85522247 |
| First-offer-only RTP (p₁ × M₁, i.e. the RTP if the offer #1 is always taken) | 73.13298711553216% |
| RTP for fixed j₀-only thresholds, j₀ ≤ 32 (uncapped region) | 73.1330% (constant — see Section 16, Question A) |
| RTP for fixed j₀-only thresholds, j₀ = 33 | 54.7644% |
| RTP for fixed j₀-only thresholds, j₀ = 34 | 40.6257% |
| RTP for fixed j₀-only thresholds, j₀ = 40 | 5.7150% |
| RTP for fixed j₀-only thresholds, j₀ = 50 | 0.0751% |
| **Optimal perfect-information RTP (the design target)** | **90.00000002%** |
| Probability of bust (unconditional, over the whole run, any horizon) | 0.8905530463 |
| Probability of ever receiving an offer | 0.7420734269 (= p₁) |
| Probability of the optimal bot ever *facing* the cap-level offer (j ≥ 33) | ≈1.9848 × 10⁻¹³ |
| Probability of the optimal bot actually *cashing out* at the $10,000 cap | ≈1.3715 × 10⁻¹⁴ |
| Expected number of offers faced per run, E[j_final] | 3.836007191 |
| Expected number of cards dealt per run, E[cards] | 8.760570554 |
| Probability of ever reshuffling (surviving past card 104) | ≲10⁻¹⁵ (not measurably different from 0; see Section 9) |
| Expected number of reshuffles per run | ≲10⁻¹⁵ (same order; not the naive E[cards]/104 ≈ 0.0842 approximation — see Section 11 note and Section 15) |

Sanity check performed: P(bust) + P(cash) + P(exceeded truncation horizon) = 0.8905530463 + 0.1094469537 + 0.0 = 1.0000000000, exactly, confirming the exact forward-under-policy propagation accounts for all probability mass with no leakage.

---

## 12. Required payout curve table, and the actual bot exploit

### 12.1 Payout curve, showing the cap transition

| Offer j | p(reach offer j), passive | Uncapped M_j = c\*×$9/p_j | Capped M_j(c\*) | Optimal decision behavior at this j (see 12.2) |
|---:|---:|---:|---:|---|
| 1 | 0.742073 | $12.13 | $9.8552 | Mixed — TAKE for most compositions, CONTINUE for favorable balanced ones |
| 2 | 0.619981 | $14.52 | $11.7960 | Mixed |
| 3 | 0.516903 | $17.41 | $14.1483 | Mixed |
| 4 | 0.430044 | $20.93 | $17.0059 | Mixed |
| 5 | 0.356998 | $25.21 | $20.4856 | Mixed |
| 8 | 0.201434 | $44.68 | $36.3061 | Mixed |
| 10 | 0.135880 | $66.24 | $53.8219 | Mixed |
| 15 | 0.048437 | $185.81 | $150.9849 | Mixed |
| 20 | 0.015984 | $563.08 | $457.5503 | Mixed |
| 25 | 0.004807 | $1,872.18 | $1,521.3115 | Mixed |
| 28 | 0.002217 | $4,059.53 | $3,298.7309 | Mixed |
| 29 | 0.001696 | $5,307.75 | $4,313.0175 | Mixed |
| 30 | 0.001290 | $6,977.89 | $5,670.1545 | Mixed |
| 31 | 0.000975 | $9,226.60 | $7,497.4304 | Mixed |
| 32 | 0.000733 | $12,274.36 | $9,974.0073 | Mixed (last sub-cap offer) |
| **33** | 0.000548 | $16,434.04 | **$10,000.00 (cap binds)** | Overwhelmingly TAKE (see 12.3) |
| 34 | 0.000406 | $22,153.46 | $10,000.00 | Overwhelmingly TAKE |
| 35 | 0.000299 | $30,079.47 | $10,000.00 | Overwhelmingly TAKE |
| 40 | 0.0000572 | $157,479.64 | $10,000.00 | TAKE |
| 50 | 7.51 × 10⁻⁷ | $11,984,342.73 | $10,000.00 | TAKE |

Note that "p(reach offer j)" in this table is the *passive/marginal* probability (a "dealer" that never lets anyone take anything, matching 04's definition exactly), used only to define the curve itself. It is **not** the probability that the optimal bot actually reaches that offer — the bot self-selects out of continuing well before j=33 in the vast majority of cases, which is exactly why the effective probability of reaching the cap under optimal play (Section 11: ≈1.98×10⁻¹³) is so many orders of magnitude smaller than the passive p₃₃ = 0.000548 shown here.

### 12.2 Finding the actual bot exploit: representative same-j, same-payout states with opposite decisions

At **position 57, j = 27** (deciding on offer 28, M₂₈(c\*) = $3,298.7309 for both states below):

| Composition | Decision | Continuation value | Ratio (continue / take) |
|---|---|---:|---:|
| nc=11, ns=11, nr=25, prev=R (balanced) | **CONTINUE** | $5,336.21 | **1.6177** |
| nc=0, ns=23, nr=24, prev=R (one suit exhausted) | **TAKE** | $3,258.72 | 0.9879 |

This is the exploit, made concrete: with the *same* offer number and the *same* nominal payout on the table, a perfect-information player facing a balanced remaining deck (clubs and spades both still plentiful, meaning the next black card is unlikely to force a bust and the run has a long, valuable future ahead of it) rationally continues — securing 61.8% *more* expected value than the payout on the table. A player facing a deck where one black suit is already exhausted (nc = 0: every future card is either a spade, which is safe only if `prev` isn't already S, or red) sees a *worse* future than the payout on offer, and rationally takes. **This asymmetry is exactly what a naive j-only or composition-blind payout table cannot see or price in — it is invisible from j alone**, and it is the mechanism 04 originally identified and this document inherits (the cap does not remove it, only bounds its magnitude — see 12.4).

A second example, right at the cap boundary itself (**position 100, j = 49**, M₅₀(c\*) = $10,000.00, the cap, for both states):

| Composition | Decision | Continuation value | Ratio |
|---|---|---:|---:|
| nc=1, ns=1, nr=2, prev=R | CONTINUE (tie) | $10,000.00 | 1.0000 |
| nc=0, ns=2, nr=2, prev=R | **TAKE** | $6,666.67 | 0.6667 |

Even when the payout on offer is *already the maximum possible*, composition still matters: with nc=1 the bot is indifferent (continuing is certain enough to reach the cap again that it ties), but with nc=0 (spades only remain among the blacks, and `prev`=R so the very next black card is safe, but the one after that risks S-after-S) the continuation value drops to two-thirds of the cap, so TAKE is strictly better. This confirms the cap limits the *dollar* exploit but does not remove the *decision* exploit.

### 12.3 Global scan of the exploit across the whole saved policy table (positions 1–400)

Across all 734,749 saved (position, state) decision entries: 650,711 are TAKE, 84,038 are CONTINUE.

- **Largest continuation/take ratio found among CONTINUE decisions:** 1014.7×, at position 51, state (nc=1, ns=1, nr=51, prev=S, j=0) — an extreme tail composition (51 of the shoe's 52 reds already dealt, only 1 club and 1 spade left) where almost every future card is red (safe against everything) and bust risk is nearly eliminated, so continuing from the tiny first-offer payout ($9.86) toward a near-certain path to much larger offers is overwhelmingly better. This is a legitimate but exceptionally rare state (reaching it requires the shoe to have dealt 50 reds and only 1 black card in the first 51 draws).
- **A separate, much less meaningful "0.0 ratio" appears at position 400** (the outer edge of the saved policy table, k_save = 400): this reflects the continuation value being computed as exactly 0 only because the saved table stops at that position (a computational truncation artifact of the k_save boundary, not a real game-theoretic finding) — and per Section 9's Argument 2, the probability of any real run surviving to position 400 is astronomically below floating-point resolution, so this artifact has zero practical or RTP significance.

### 12.4 Does the cap eliminate the exploit, or only limit it?

**Only limits it.** The decision mechanism (composition-driven continuation value differing from the flat, j-only payout) is structurally identical with or without the cap — Section 12.2's first example (position 57, offers well below the cap) shows the exploit operating exactly as it did in the uncapped 04 model. What the cap changes is the *maximum dollar magnitude* the exploit can ever be worth (bounded at $10,000 regardless of how favorable the composition becomes) and, as a direct consequence of the calibration in Section 6, the *overall RTP* the exploit nets the bot across the whole game (exactly 90.00% at c\*, versus 04's unbounded/undefined value in the absence of any cap).

---

## 13. Testing the cap: the four required comparisons

| Curve | Perfect-information optimal V(initial) | RTP |
|---|---:|---:|
| (1) Uncapped, unscaled M_j = $9/p_j (04's baseline) | Diverges without bound as horizon grows (04's finding; reconfirmed structurally in Section 7 via the homogeneity argument — no finite value exists) | Diverges / undefined |
| (2) Same curve, simply capped at $10,000, no rescaling (c=1) | $10.9299 (stable plateau, k_max ≥ 100) | 109.2993% |
| (3) Mathematically derived dynamic curve satisfying the cap: M_j(c\*), c\* = 0.8125887457281351 | $9.000000002 (stable plateau, k_max ≥ 100) | **90.00000002%** |
| (4) Perfect-information optimal bot evaluated against curve (3) | (same as row 3 — the bot *is* the optimal player being valued) | 90.00000002% |

These are the actual computed results, run without post-hoc tuning: curve (1) is unusable (no finite answer exists at any scale, per Section 7); curve (2) demonstrates that capping alone is not sufficient — it overshoots the 90% target by over 19 points; curve (3)/(4) is the derived solution.

---

## 14. Simulation validation

Three independent, structurally distinct computations were cross-checked against each other, none derived from or dependent on the others:

**Method 1 — Backward induction (analytical, exact recursion).** `dynrtp_backward.solve_backward` at k_max = 1600 with M = M_j(c\*): **V₀ = $9.000000002319613** (RTP 90.00000002319613%).

**Method 2 — Independent exact forward-under-policy propagation (non-Monte-Carlo).** A structurally different computation (`capped_policy_forward.run_forward_under_policy`): rather than solving backward, this propagates the *true probability distribution* forward through card indices from the initial state, applying at each red-card decision the *same* optimal decision the backward induction derived (read from the saved policy table), and sums the resulting probability-weighted payouts directly. At k_max = 400: **V₀ = $9.000000002** (matching Method 1 to 9 significant digits), P(bust) = 0.8905530463, P(cash) = 0.1094469537, sum = 1.0000000000 exactly (no probability leakage). E[cards] = 8.760570554, E[offers] = 3.836007191.

**Method 3 — Independent real-deck Monte Carlo simulation.** `capped_mc_validate.py`: builds actual two-52-card-deck (104-card) shoes as lists of (rank, suit) tuples, shuffles with `random.shuffle`, and looks up the bot's decision at each red-card offer directly from the saved backward-induction policy table (a genuinely separate code path from both Methods 1 and 2 — no category-probability sampling, real suit identities tracked). Run with **seed = 20260918, N = 2,000,000 trials**:

- Simulated EV = $9.065181, simulated RTP = 90.6518%
- Analytic V₀ (Method 1) = $9.000000002319613, analytic RTP = 90.0000%
- SE(EV) ≈ $0.131321 (estimated from a 200,000-trial subsample's payout variance)
- **z = (9.065181 − 9.000000) / 0.131321 = +0.4963**
- P(bust) = 0.890333, P(cashed) = 0.109667 (both within Monte Carlo noise of Methods 1/2's exact 0.8905530463 / 0.1094469537)
- P(exceeded the saved k_save=400 horizon, discarded) = 0.0 across all 2,000,000 trials
- P(cash at the exact $10,000 cap), in-sample = 0.0 (2,000,000 trials is far too small a sample to expect to observe an event of probability ≈1.4×10⁻¹⁴ — this is the expected, correct outcome, not a discrepancy)
- E[cards dealt] = 8.7598, E[offers seen] = 3.8365 (both matching Method 2's 8.7606 / 3.8360 closely)
- E[reshuffles] = 0.000000 across all 2,000,000 trials (consistent with Section 9's ≲10⁻¹⁵ finding)
- **Policy lookup misses: 0 / 7,673,081** offers evaluated — every single lookup found a valid stored decision, with zero silent fallback

z = +0.4963 is well within ordinary sampling noise (|z| < 1); the three methods agree. **A discrepancy was in fact found and investigated during development** (see Section 15's audit item on this exact failure mode) — the resolution was a bug in the simulator's state-key convention, not in the analytical model, consistent with the requirement to treat the analytical model as the source of truth and to fix the simulation rather than the math.

---

## 15. Adversarial audit

Each item from the required checklist, addressed explicitly:

| Audit item | Finding |
|---|---|
| Perfect-history bot | This is the primary subject of the entire document — modeled with full (nc,ns,nr,prev,j) information at every decision, no restriction to heuristics. RTP = 90.00000002%, exactly the target. |
| Favorable black-suit composition (balanced nc≈ns) | Quantified in Section 12.2: continuation value up to 61.8% above the on-offer payout at position 57/j=27 for a balanced nc=11,ns=11 state; up to 1014.7× at an extreme tail state (nc=1,ns=1,nr=51). Composition-driven exploit exists and is fully quantified, not hidden. |
| Unfavorable composition (one suit exhausted) | Quantified in Section 12.2: continuation value drops to 98.8%–96.7% of (or well below) the on-offer payout when a black suit is exhausted (nc=0 examples at both j=27 and j=49) — TAKE becomes correct. |
| Repeated reshuffles | P(any reshuffle at all) ≲10⁻¹⁵ (Section 9, Argument 2); zero reshuffles observed in 2,000,000 Monte Carlo trials (Section 14); the reshuffle transition itself (reset nc,ns,nr, preserve prev and j) is implemented and exercised identically in all three independent methods, so the (vanishingly rare) event is modeled correctly even though it is not expected to be observed. |
| Very large j | Explored up to j=50+ in the payout table (Section 12.1) and up to j=207+ in the raw policy scan (Section 12.3); RTP contribution from any single large-j state is bounded above by the cap ($10,000) and the probability of reaching such j under optimal play is astronomically small (Section 11), so no material RTP risk. |
| 1,000× cap states | P(the optimal bot ever faces the cap-triggering offer, j≥33) ≈ 1.9848×10⁻¹³; P(actually cashing at the cap) ≈ 1.3715×10⁻¹⁴. The cap is reachable in principle (nonzero probability, correctly modeled) but immaterial in practice. |
| Rounding to cents | Re-running the full backward induction with every M_j rounded to the nearest cent gives V₀ = $9.000300925966307 (RTP 90.00300925966307%) versus the exact V₀ = $9.000000002319613 (RTP 90.00000002%) — a shift of **+0.00301 percentage points**. Immaterial, consistent with every prior document (03/04/05) in this project. |
| Payout rounding | Same test as above — the rounding is applied to the payout schedule M_j itself (the only place rounding would occur in an actual implementation), not to intermediate probabilities. |
| Exact $10,000 boundary | Confirmed exactly: M₃₂(c\*) = $9,974.0073 (below cap), M₃₃(c\*) computed uncapped as $16,434.04, clamped to exactly $10,000.00 by the min(·, cap) in the payout family definition (Section 6.1) — the boundary is crossed cleanly between j=32 and j=33 with no off-by-one or floating-point ambiguity. |
| First-card Ace of Spades | P = 2/104 exactly, structurally built into the initial-state distribution used by all three independent methods (Methods 1/2 via the analytic d₁ dictionary derived from NC0=26,NS0=26,NR0=52 minus the 2 explicit AoS states folded in; Method 3 by tracking the literal card (rank, suit) = (Ace, Spade) in a real shuffled deck). No discrepancy across methods. |
| Red after black | Confirmed safe in all three methods — this is simply "any red card always offers, regardless of prev" (Section 2, rule 5–6), the default case with no bust check needed. |
| Black after red | Confirmed safe in all three methods (Section 2, rule 6, "any black card after a red card → safe") — tested implicitly on every run that continues past a red offer into a subsequent black card. |
| C→C | Confirmed bust in all three methods (the `prev == "C"` branch in every implementation returns/accumulates the bust probability). |
| S→S | Confirmed bust in all three methods (symmetric `prev == "S"` branch). |
| C→S | Confirmed safe (`prev != "C"` allows the spade transition). |
| S→C | Confirmed safe (`prev != "S"` allows the club transition). |
| Red→Red | Confirmed safe — reds never bust each other under any rule (Section 2, rule 6); every red always offers, and there is no code path in any of the three methods that treats R-after-R as a bust. |
| Shoe exhaustion | Modeled as the `nc+ns+nr==0 → reset to (26,26,52)` transition, applied identically across all three methods (Section 3). |
| Reshuffle while prev = C | Explicitly not reset — `prev` is carried through the reshuffle transition unchanged in all three implementations (verified by code inspection: the reshuffle branch resets only nc, ns, nr, never touching `prev` or `j`). |
| Reshuffle while prev = S | Same as above, symmetric. |
| Reshuffle while prev = R | Same as above. |
| Offer immediately before reshuffle | Handled generically: the composition-based reshuffle check (`nc+ns+nr==0`) is evaluated at the point a card is about to be dealt, independent of whether the previous action was an offer decision — no special-casing was needed or introduced, and this path is exercised (at negligible but nonzero probability) identically to any other reshuffle. |
| Very rare states | The position-51 extreme-tail state in Section 12.3 (nc=1,ns=1,nr=51) is a concrete example of a very rare state correctly present in, and correctly handled by, the reachable-state dictionaries — it was not manually special-cased; it fell out of the exhaustive enumeration naturally. |
| Repeated favorable states | Each run is dealt from an independent fresh shuffle (or reshuffle within a run); there is no mechanism by which a bot's decision in one run could affect, or be affected by, another run's composition — the i.i.d. structure of shuffles rules out any cross-run advantage. Within a single run, "repeated favorable states" (e.g., staying in a balanced-composition regime across several consecutive offers) are already fully priced into the backward-induction continuation value at each state, since that value is itself computed by recursing through all subsequent states, favorable or not. |
| Can the bot repeatedly exploit composition to exceed 90% materially? | **No.** The whole point of the calibration (Section 6) is that V(initial) — which already accounts for the bot optimally exploiting composition at every single decision point in the entire game tree, not just once — equals exactly $9.00. There is no "meta-strategy across states" left for the bot to additionally apply: backward induction by construction already finds the single best policy over the complete state space, so 90.00000002% *is* the bot's best-case achievable RTP, not a lower bound that some cleverer bot could beat. |

**No item in this checklist was found to allow RTP to exceed 90% materially.** The single largest deviation found anywhere in the audit is the cent-rounding effect (+0.00301 percentage points), which is a real, non-adversarial, unavoidable consequence of implementing money in cents rather than a bot exploit.

---

## 16. Three separated questions

**Question A — Does the payout curve give 90% RTP to j-only strategies (Section 4, player A)?**

**No.** Table (Section 11 / Section 5's derivation): under M_j(c\*), a player who commits in advance to "take the j₀-th offer I ever see, regardless of composition" gets a *constant* RTP of exactly **73.1330%** for every threshold j₀ ≤ 32 (the sub-cap region — the constant value arises because EV = p_{j₀} × min(c\* × $9/p_{j₀}, cap) = c\* × $9 exactly, independent of j₀, whenever the min doesn't bind), declining further for j₀ ≥ 33 (where the payout is capped at $10,000 but p_{j₀} keeps shrinking, so EV = $10,000 × p_{j₀} falls toward zero: 54.76% at j₀=33, 40.63% at j₀=34, 5.72% at j₀=40, 0.08% at j₀=50).

**Question B — Does the payout curve give 90% RTP to the perfect-information optimal bot?**

**Yes, exactly.** V(initial) = $9.000000002319613, RTP = 90.00000002319613%, by construction (Section 6) and independently confirmed three separate ways (Section 14).

**Question C — Can the payout curve satisfy BOTH perfect-information-optimal RTP = 90% AND maximum payout ≤ 1,000×?**

**Yes.** This is exactly what M_j(c\*) is: a payout function with a hard $10,000 ceiling (never violated, by construction of the `min(·, cap)` in Section 6.1) that simultaneously achieves V(initial) = $9.00 for the perfect-information optimal bot (Section 6.5, cross-validated Section 14). This is the actual design requirement, and it is met.

The cost of meeting it is paid by ordinary, composition-blind players (Question A: 73.13% instead of 90%) — not by the game exceeding 90% RTP for anyone. This asymmetry is the practical/game-design consequence flagged separately per the brief's Section 6 instruction, and is discussed further in Section 17.

---

## 17. Final mathematical conclusion

Per the required three-case framework:

### CASE 1 applies.

**A valid 1,000×-capped dynamic payout function exists — M_j(c\*) = min(c\* × $9.00 / p_j, $10,000), c\* = 0.8125887457281351 — and produces exactly 90.000% RTP (90.00000002319613%, converged and stable from k_max ≥ 100 through k_max = 3200) against the perfect-information optimal bot, confirmed independently by (1) exact backward induction, (2) an independent exact forward-under-policy propagation, and (3) an independent real-deck Monte Carlo simulation (2,000,000 trials, z = +0.50).**

This conclusion follows directly from the mathematics developed above, not from a preference for a "nicer" answer: the existence argument (Section 6.3, monotonicity + IVT) is a proof, not a numerical coincidence, and the necessity of the cap (Section 7, homogeneity) shows that without capping, no finite calibration could ever have existed at all — so the specific 1,000× design constraint given in this task is precisely what makes the 90% RTP target achievable, not an obstacle that had to be worked around.

**Honestly flagged design implication (not swept under the rug):** achieving 90% RTP against the worst-case (perfect-information) player necessarily under-delivers RTP to simpler, composition-blind players (73.13% for any fixed j-only threshold at or below the cap boundary, less above it — Section 16, Question A). This is an inherent property of calibrating a single-scale, j-indexed payout curve against the *best* possible adversary: any curve that gives the optimal bot exactly 90% must, by the same fixed-j payout schedule, give a naive fixed-threshold player *less* than 90%, because the bot only ever equals or beats the naive player's fixed-threshold value at every decision (that is the definition of "optimal"). There is no design within this payout family that gives both the naive player and the optimal bot the same 90% simultaneously — achieving equality for both would require a fundamentally different, composition-dependent payout table (representation (C)/(D) from Section 5), which the brief separately asked to be documented as a distinct practical/game-design consideration rather than folded silently into the recommended solution. This document recommends the j-only calibrated curve (M_j(c\*)) specifically because it is the simplest representation that meets the actual, stated design requirement (Question C) and because the resulting under-delivery to naive players is a conservative direction (the house is never at risk of paying out more than designed to any player), not a defect requiring further correction.

---

## 18. Unresolved issues / observations flagged for the production math team

1. **Non-monotonic cash-out pattern by offer number.** The exact forward-under-policy computation shows the optimal bot's cash_by_j distribution is not monotonic in j in a simple way — e.g., in early exploratory runs of this analysis, cash mass appeared at j=4,5,7,8,9 but not at j=1,2,3,6 for certain truncations. This is a genuine, non-obvious structural feature of the optimal policy (which offer numbers are ever actually the *first* one at which taking becomes better than continuing, averaged over the enormous variety of possible compositions reachable at each j) rather than a computational error — it does not affect the correctness of V₀, which has been cross-validated three independent ways to 9 significant digits. A full closed-form characterization of exactly which j-values ever see a nonzero TAKE rate was not derived within the scope of this document and is flagged as a possible follow-up for the production team if a more granular understanding of the cash-out distribution is wanted (e.g. for UX or payout-table display purposes), but it has no bearing on the RTP conclusion above.
2. **k_save = 400 truncation artifact.** The saved policy table (used to drive the Monte Carlo and the forward-under-policy cross-check) only stores decisions for positions 1–400. Section 12.3 notes a spurious "ratio = 0.0" at position 400 caused by this truncation (continuation value reads as 0 only because nothing beyond position 400 was saved), not by any real game dynamic. Since P(any run surviving to position 400) is far below floating-point resolution (Section 9), this has zero effect on any reported number, but a production implementation should not mistake this artifact for evidence that continuing is ever actually worthless that deep into a run.
3. **Reshuffle probability is reported as an upper-bound order of magnitude (≲10⁻¹⁵), not a proven exact zero.** The true probability of surviving to a reshuffle is a positive real number (an arbitrarily long run of alternating-color safe cards is not impossible, only fantastically unlikely) — it underflows to exactly 0.0 in IEEE double precision at k=104 in this implementation, but a from-scratch symbolic/exact-rational recomputation was not performed. This does not affect the RTP conclusion (Section 9's Argument 1 bounds V(S) ≤ $10,000 unconditionally regardless of reshuffle probability), but is noted for completeness rather than overclaiming exact-zero where the true claim is "below double-precision resolution."
4. **Cent-rounding, while immaterial to RTP (+0.00301 pp), was not re-verified against the bot's *decisions* changing near an exact tie.** The audit (Section 15) reran the value computation with rounded payouts but did not separately enumerate whether any specific state's TAKE/CONTINUE decision flips under rounding (as opposed to just the aggregate RTP effect). Given the tiny aggregate effect, this is very unlikely to matter, but is flagged rather than silently assumed away.

None of the above affects the Section 17 conclusion; they are noted so a production team auditing this document has a complete, honest account of what was and was not fully closed out.

---

## Appendix: reproducible scripts

All scripts live in `/home/claude/math-model/` and share the unmodified 04 machinery (`dynrtp_forward_full.py`, `dynrtp_backward.py`, `dynrtp_offercount.py`) as their foundation.

| Script | Purpose |
|---|---|
| `capped_model.py` | Defines the capped/scaled payout family M_j(c); `find_c_star` bisects for c\* such that V(initial;c\*)=$9.00; `__main__` reproduces the Section 6.5 convergence tables (edit the `c=1.0` argument to `c_star` to reproduce the second table). |
| `capped_backward_full_policy.py` | `solve_and_save_policy`: runs the backward induction and additionally saves the full decision/value table for positions 1..k_save, needed to drive the Monte Carlo and forward-under-policy validators. |
| `capped_policy_forward.py` | `run_forward_under_policy`: the independent, non-Monte-Carlo exact forward propagation under the saved optimal policy; reproduces Section 11's exact probabilities/expectations and Section 9's alive-mass-by-horizon table. Run directly (`python3 capped_policy_forward.py`) after generating and pickling a policy (see below). |
| `capped_mc_validate.py` | The independent real-deck Monte Carlo validator (Section 14, Method 3). Contains an inline code comment documenting the pre/post-increment state-key bug found and fixed during development (Section 14's "discrepancy found and investigated" note). |
| `capped_backward_policy.py` | Superseded / exploratory only (an earlier sparse-snapshot approach to policy stationarity checking that was abandoned in favor of the contiguous-range approach in `capped_backward_full_policy.py`; retained for provenance, not part of the load-bearing reproduction path). |

**To reproduce c\*, V₀, and the saved policy table from scratch:**

```python
from dynrtp_forward_full import forward_pass_full
from capped_model import find_c_star, capped_M
from capped_backward_full_policy import solve_and_save_policy
import pickle

K_MAX = 1600
dist_by_k, p_reach_j = forward_pass_full(K_MAX)
c_star, resid = find_c_star(dist_by_k, p_reach_j, K_MAX)
M = capped_M(c_star, p_reach_j)

K_SAVE = 400
V0, policy_by_pos = solve_and_save_policy(K_MAX, M, dist_by_k, K_SAVE)
print(c_star, V0, V0/10*100)

with open('/tmp/policy_capped.pkl', 'wb') as f:
    pickle.dump({'V0': V0, 'policy_by_pos': policy_by_pos, 'M': M, 'p_reach_j': p_reach_j}, f)
```

Then `python3 capped_policy_forward.py` and `python3 capped_mc_validate.py` (both read `/tmp/policy_capped.pkl`) reproduce Sections 11 and 14 respectively.
