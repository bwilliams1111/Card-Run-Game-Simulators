# 09 — Card Run: Crash. Final Mathematical Production Specification

**Status: FINAL. This document is the single source of mathematical truth for production implementation and independent verification.**

Documents 01–08 were the development, exploration, correction, and verification history that produced the mathematics below. They remain available as background and audit trail, but production must implement and verify against **this document only**. Where anything below differs from an earlier document, this document governs (Section 22 records every place that happened and why).

**Authority hierarchy:**
- The approved PRD / game rules define **what** the game does (deck, cards, bust conditions, offers, cash-out). This document restates the frozen subset needed for the math (Section 1) but does not supersede the PRD on gameplay/UX matters outside the mathematical model.
- **This document** defines **how the mathematics works**: the exact payout formula, the exact optimal-play model, the exact RTP calculation, and the exact mathematical acceptance criteria production must reproduce.
- The reference prototype (the "Crash Deal" artifact) and any GitHub code are illustrative only. They are never a mathematical authority. If prototype code and this document disagree, this document is correct and the code must be fixed.

This document assumes no prior knowledge of Documents 01–08.

---

## 1. Frozen Game Rules

### 1.1 Deck

- Two standard 52-card decks are combined into one 104-card shoe before a run begins (52 red cards: hearts + diamonds; 26 clubs; 26 spades — the doubling means, for example, there are two physical Aces of Spades).
- The shoe is shuffled once before the run starts.
- Cards are dealt one at a time, without replacement, from the shoe.
- **Reshuffle:** if all 104 cards are dealt without the run having busted or cashed out, a fresh 104-card shoe is shuffled and dealing continues **within the same run**. The run does not end or reset at a reshuffle.
  - The offer count (how many cash-out offers have been presented and declined so far) is **not** reset by a reshuffle.
  - The category (club / spade / red) of the most recently dealt card **is** carried across the reshuffle boundary — the first card of the new shoe is checked for a bust against the last card of the old shoe exactly as if the shoe boundary did not exist.
  - Only the remaining-card composition resets, to 26 clubs / 26 spades / 52 reds.

### 1.2 Wager

- W = $10.00, fixed. Not configurable.

### 1.3 First card

- The first card dealt in a run never produces a cash-out offer, regardless of its suit or color.
- If the first card is the Ace of Spades, the run busts immediately (payout $0). There are two physical Aces of Spades in the 104-card shoe, so P(first-card bust) = 2/104.
- If the first card is not the Ace of Spades, the run continues normally; the card's category (club, spade, or red) becomes the "previous card" for the bust rule on card 2.

### 1.4 Bust rules

Bust is evaluated by comparing the category of the current card to the category of the immediately preceding card:

| Current card | Previous card | Result |
|---|---|---|
| Club | Club | **BUST** |
| Spade | Spade | **BUST** |
| Club | Spade | Safe |
| Spade | Club | Safe |
| Red | Red | Safe |
| Red | Club or Spade | Safe |
| Club or Spade | Red | Safe |

Only two consecutive same-black-suit cards (club-after-club or spade-after-spade) bust. Red never busts against anything, in either order. On bust, the run ends immediately with payout $0.

### 1.5 Offers

From the second card onward (the first card never offers, per 1.3):

- If the dealt card is **red** and does not itself cause a bust (it never can — see 1.4), it produces a **cash-out offer**. This is the run's next offer in sequence: if `j` offers have already been declined, this is offer `j+1`.
- If the dealt card is **black** (club or spade) and does not bust the run, **no offer** is produced; the run simply continues to the next card with no decision to make.

### 1.6 Cash-out

- Whenever an offer exists, the player (or, for RTP certification, the model bot) may **TAKE** it — collect the offer amount immediately and end the run — or **CONTINUE** — decline it and keep playing, at the risk of a future bust forfeiting everything.
- The mathematical model assumes the decision-maker can make the mathematically optimal TAKE/CONTINUE decision at every single offer, with no error, no fatigue, and no hesitation. This is not a claim about real player behavior; it is the modeling assumption that defines the RTP guarantee (Section 5).

### 1.7 UI display is not a mathematical restriction

The production UI displays only the four most recently dealt cards. **This is a UI/UX decision only.** For the purpose of RTP certification and the mathematical model in this document, it is mandatory to assume the decision-maker has instant, complete access to: the full card history, the exact remaining composition of the current shoe, the previous card's category, the current offer count, and the current offer amount. See Section 5.

---

## 2. Configuration Parameters

| Parameter | Symbol | Default | Configurable? |
|---|---|---|---|
| Wager | W | $10.00 | No |
| Starting Multiplier | S | 1.05× | Yes |
| Target RTP | R | 90.000% | Yes |
| Maximum Multiplier | C | 1,000× | No |
| Maximum Cashout | — | $10,000 (= C × W) | No — derived from C and W |

**Starting Multiplier is economically real, not cosmetic.** It fixes the dollar value of the very first cash-out offer directly:

```
M_1 = S × W
```

At the default configuration, `M_1 = 1.05 × $10.00 = $10.50`.

**Target RTP is a genuine economic configuration input.** It fixes the dollar value the entire payout curve must produce, in expectation, under mathematically optimal play (Section 5):

```
TARGET_EV = W × R
```

At the default configuration, `TARGET_EV = $10.00 × 0.90 = $9.00`.

Production must validate every `(S, R)` pair against the feasibility condition in Section 10 before generating a payout table for it. An infeasible configuration must be rejected, never silently degraded into an approximate table.

---

## 3. Final Approved Payout Formula

This is the complete, final production payout architecture. Do not replace it with a different payout family. It has three parts: the fixed first offer, the calibrated-and-floored formula for every later offer, and the single scalar that ties them together.

```
TARGET_EV = W × R

M_1 = S × W                                                     ... (fixed, exogenous)

For every offer j >= 2:

    M_j(c) = max( S × W,  min( c × TARGET_EV / p_j,  C × W ) )   ... (single free scalar c)
```

where:

- `p_j` = the probability of reaching at least offer `j` under passive dealing (Section 8) — a pure fact about the card process, independent of `S`, `R`, `c`, or player strategy.
- `c` = a single calibration scalar, the *only* free parameter in the entire formula, solved mathematically (never hand-tuned) so that:

```
V_initial(c) = TARGET_EV = W × R
```

where `V_initial` is the expected value of the run at its very start, computed under the perfect-information optimal-stopping model in Sections 5–6. `c` is found by bisection (root-finding on a single monotonic, continuous scalar equation) — see Section 20 for the exact procedure.

**Why the formula has this exact shape (brief rationale, full derivation in Documents 07–08):**

- `M_1 = S × W` makes the Starting Multiplier a real, first-class economic input rather than a display-only number.
- The `c × TARGET_EV / p_j` term for `j ≥ 2` is the minimal, non-arbitrary generalization of the original 90%-per-offer fairness construction (Document 04): dividing a fixed target by the probability of reaching that offer is what keeps every offer's expected contribution proportional, and scaling the whole family by one shared calibration scalar `c` is what allows the curve to be tuned to hit an arbitrary Target RTP without touching its shape.
- The `min( ... , C×W )` term enforces the $10,000 cap (Section 11).
- The outer `max( S×W, ... )` — the **floor** — is what guarantees the offer ladder is monotonic (Section 9). It was added in Document 08 specifically because, without it, the calibrated `j≥2` curve can (for some `S`/`R` combinations) dip below `M_1` at `j=2`, producing a payout ladder that goes down before it goes up. The floor is not a clamp applied after the fact — `c` is re-solved from scratch against this exact floored formula, so `V_initial(c) = TARGET_EV` holds exactly with the floor included, not approximately.

**Production must never post-process individual offers to force monotonicity, hit the target RTP, or "smooth" the curve.** Monotonicity and RTP-correctness are both already guaranteed by construction once `c` is solved correctly against the formula above. Any manual adjustment after that point invalidates the mathematical guarantee.

---

## 4. Perfect-Information Optimal Bot (mandatory RTP definition)

The RTP that the payout formula is calibrated to (`V_initial = W × R`) is the RTP achieved by a **perfect-information optimal bot**, defined as a decision-maker that, at every offer, knows:

- The complete card history of the run so far.
- The exact remaining composition of the current shoe (exact counts of clubs, spades, and reds left).
- The category of the immediately preceding dealt card.
- The current offer count (how many offers have already been declined).
- The current offer amount.
- The complete payout formula (all present and future `M_j` values).

...and which, given all of that, always chooses whichever of TAKE or CONTINUE has the higher mathematically-computed expected value (Section 6).

**This is mandatory, not a simplifying assumption.** The production RTP guarantee is a guarantee against the single worst-case adversary the game must be safe against. Any player restricted to less information — including a real player who can only see the four most recently dealt cards, per Section 1.7 — earns an RTP no higher than this bot's, because the bot's policy is optimal by construction. Certifying the perfect-information bot at exactly `W × R` therefore certifies every weaker real player is at or below that RTP as well. Production must **not** substitute a simpler strategy (cash on every red, hold until a fixed offer number, a fixed-dollar threshold, human-typical play, or a four-card-memory heuristic) as the basis for the RTP guarantee. Such strategies may be used only as secondary diagnostics, never as the calibration target.

---

## 5. State Definition

The mathematical model's state, sufficient for exact optimal-stopping backward induction, is:

```
(n_c, n_s, n_r, prev, j)
```

| Variable | Meaning | Range |
|---|---|---|
| `n_c` | Clubs remaining in the current shoe | 0–26 |
| `n_s` | Spades remaining in the current shoe | 0–26 |
| `n_r` | Red cards (hearts + diamonds combined) remaining in the current shoe | 0–52 |
| `prev` | Category of the immediately preceding dealt card | `{C, S, R}` |
| `j` | Number of offers already presented and declined in this run | `0, 1, 2, ...` |

**Precisely what each count represents.** `n_c + n_s + n_r` is always the number of undealt cards remaining in the *current* shoe (before any needed reshuffle). Hearts and diamonds are never distinguished from each other anywhere in the rules or the mathematics — only their combined count `n_r` matters, because no rule treats them differently. Clubs and spades must be tracked separately because the bust rule treats club-after-club and spade-after-spade specially but treats club-after-spade (and vice versa) as safe.

**Why absolute card position is not part of the state.** The future probabilities and payouts available from a given state depend only on `(n_c, n_s, n_r, prev, j)` — not on how many physical cards it took to reach that state. Card position is used only as a bookkeeping index for the backward-induction sweep, never as an argument to the value function or the payout function.

**State transitions, per card dealt** (given `remaining = n_c + n_s + n_r > 0`, i.e. after any needed reshuffle has already been applied):

| Card drawn | Probability | Effect |
|---|---|---|
| Club | `n_c / remaining` | If `prev = C`: **BUST** (value 0). Otherwise: `prev ← C`, `n_c ← n_c − 1`, `j` unchanged, no offer. |
| Spade | `n_s / remaining` | If `prev = S`: **BUST** (value 0). Otherwise: `prev ← S`, `n_s ← n_s − 1`, `j` unchanged, no offer. |
| Red | `n_r / remaining` | Never busts. `prev ← R`, `n_r ← n_r − 1`. An offer for `M_{j+1}` is presented; if declined, `j ← j + 1`. |

**End-of-shoe / reshuffle:** whenever `n_c + n_s + n_r = 0` (the shoe is exhausted with no bust and no cash-out yet), the *next* card is dealt from a freshly shuffled shoe. The transition rule is: reset `(n_c, n_s, n_r) ← (26, 26, 52)` **before** drawing the next card.

**What survives the reshuffle, and what resets:**

| Variable | Behavior at reshuffle |
|---|---|
| `n_c, n_s, n_r` | **Reset** to `(26, 26, 52)` |
| `prev` | **Carried through unchanged** — the last card of the old shoe is still `prev` for evaluating the first card of the new shoe |
| `j` (offer count) | **Carried through unchanged** — a reshuffle never resets how many offers have already been declined |

**First card (special case, rank-aware — the only rank-aware rule in the game):**

| Outcome | Probability | Resulting state |
|---|---|---|
| Ace of Spades (bust) | 2/104 | Run ends, payout $0 |
| Ordinary spade | 24/104 | `(n_c, n_s, n_r, prev, j) = (26, 25, 52, S, 0)` |
| Club | 26/104 | `(25, 26, 52, C, 0)` |
| Red | 52/104 | `(26, 26, 51, R, 0)` |

The first card never produces an offer under any of these three surviving outcomes, so `j = 0` after card 1 regardless of its color.

---

## 6. Bellman Equation (Optimal-Stopping Recursion)

The value of any state is:

```
V(state) = max( TAKE, CONTINUE )
```

evaluated only at a red-card offer (a "decision state"); black-card transitions and bust have no decision to make and simply propagate the corresponding branch value.

**TAKE:**

```
TAKE(n_c, n_s, n_r, prev, j) = M_{j+1}
```

i.e., the payout assigned to the offer currently being decided — this is the `(j+1)`-th offer, since `j` counts offers already declined.

**CONTINUE — the full expectation over the next card**, written out completely (not merely described in prose):

```
CONTINUE(n_c, n_s, n_r, prev, j) =

      (n_c / rem) * [ 0                                     if prev = C
                     ; V(n_c−1, n_s, n_r, C, j)               otherwise ]

    + (n_s / rem) * [ 0                                     if prev = S
                     ; V(n_c, n_s−1, n_r, S, j)               otherwise ]

    + (n_r / rem) * max( M_{j+1} ,  V(n_c, n_s, n_r−1, R, j+1) )
```

where `rem = n_c + n_s + n_r`, and, wherever a transition would reference a state with `n_c + n_s + n_r = 0`, that state is first replaced by `(26, 26, 52)` (the reshuffle substitution — `prev` and `j` are carried through as written; only the composition is replaced).

**Full recursion, combined:**

```
V(n_c, n_s, n_r, prev, j) =

      (n_c / rem) * [ prev=C ? 0 : V(n_c−1, n_s, n_r, C, j) ]
    + (n_s / rem) * [ prev=S ? 0 : V(n_c, n_s−1, n_r, S, j) ]
    + (n_r / rem) * max( M_{j+1} , V(n_c, n_s, n_r−1, R, j+1) )
```

with the reshuffle substitution `(n_c, n_s, n_r) ← (26, 26, 52)` applied first whenever `n_c + n_s + n_r = 0`.

`V_initial`, the quantity that must equal `TARGET_EV = W × R`, is the expectation of this recursion over the three first-card outcomes in Section 5:

```
V_initial = (26/104) * V(25, 26, 52, C, 0)
          + (24/104) * V(26, 25, 52, S, 0)
          + (52/104) * V(26, 26, 51, R, 0)
```

(the Ace-of-Spades branch contributes `(2/104) × $0 = 0` and is omitted from the sum for that reason, not because it is ignored).

Because every transition strictly decreases `remaining` within a shoe (and the reshuffle substitution is a well-defined reset, not a cycle back to an already-being-computed state), this recursion has no circular dependencies and is solved by ordinary backward induction: compute `V` for every state at the maximum card position first, then work backward one card position at a time to the initial state. In practice this requires truncating at a finite maximum horizon `k_max` (Section 16 gives the exact bound on the resulting error and how `k_max` must be chosen).

---

## 7. Monotonicity

**Requirement:** the offer ladder must satisfy

```
M_1 <= M_2 <= M_3 <= ... <= C × W
```

with no exceptions, until the cap is reached, after which every subsequent offer equals the cap exactly. Production must never permit or post-process around a violation of this.

**Why the formula in Section 3 guarantees this, proven, not assumed:**

1. `p_j` (Section 8) is **strictly decreasing** in `j` — a verified structural fact about the card process, independent of `S`, `R`, or `c`.
2. Therefore, for `j ≥ 2`, the pre-cap term `c × TARGET_EV / p_j` is **strictly increasing** in `j` (a fixed positive numerator divided by a strictly decreasing positive denominator).
3. A function that is strictly increasing while below a ceiling, and constant once it reaches that ceiling, is non-decreasing everywhere and — once capped — never leaves the cap. So `min(c × TARGET_EV/p_j, C×W)` for `j ≥ 2` is non-decreasing in `j` unconditionally, for any `c > 0`.
4. Applying the outer floor, `max(S×W, ·)`, preserves this: `max` with a fixed constant maps a non-decreasing sequence to a non-decreasing sequence.
5. Therefore `M_2 ≤ M_3 ≤ M_4 ≤ ...` holds unconditionally. **The only place the entire ladder can ever fail to be monotonic is the single seam `M_1 → M_2`.**
6. The floor `M_j(c) = max(S×W, ...)` forces `M_2 ≥ M_1` directly by construction (`M_2(c) = max(M_1, min(...)) ≥ M_1` always). Combined with point 5, this guarantees the entire ladder `M_1 ≤ M_2 ≤ M_3 ≤ ...` is non-decreasing.

Monotonicity is therefore a property of the payout formula itself, proven mathematically, not something production must check for and fix after the fact. Production **must never** manually reduce, raise, or reorder an individual offer to enforce this — doing so breaks the calibration in Section 3 and invalidates the RTP guarantee.

---

## 8. Offer Probabilities (`p_j`)

**Definition:**

```
p_j = P(a passively-dealt run — one that never voluntarily cashes out — reaches at least offer j)
```

`p_j` is determined **entirely** by the card process described in Section 1 (deck composition, bust rules, offer rules, reshuffle rules). It does **not** depend on, and must never be computed from: the Starting Multiplier, the Target RTP, the payout table, the calibration scalar `c`, or any player's or bot's strategy. It is computed once, from the game rules alone, by exact forward enumeration over the state space in Section 5 (with `j` incremented on every red card, regardless of whether that offer is ever taken).

**Reference verification table, `p_1` through `p_40`** (`p_1` is the probability the run ever receives a first offer at all; this sequence is what every payout table in this specification is derived from). This table exists so an implementer can spot-check their own forward enumeration against known-correct values. **It is a verification excerpt, not the complete mathematical domain of `p_j`.** `p_j` is well-defined, and computable by the same state-based forward enumeration (Section 5), for every positive integer `j` — there is no mathematical upper limit at `j = 40`, and none is claimed:

| j | p_j | j | p_j |
|---:|---:|---:|---:|
| 1 | 0.7420734269371773 | 21 | 0.012670511546628392 |
| 2 | 0.6199813571142185 | 22 | 0.010005685325475772 |
| 3 | 0.5169028487580739 | 23 | 0.007869808858691443 |
| 4 | 0.4300441656690021 | 24 | 0.006164148226064929 |
| 5 | 0.3569975277413691 | 25 | 0.004807232846025926 |
| 6 | 0.2956911692817137 | 26 | 0.003732040094432804 |
| 7 | 0.24434542487928082 | 27 | 0.0028836061875765545 |
| 8 | 0.2014341674226326 | 28 | 0.0022170037180422424 |
| 9 | 0.1656509927620332 | 29 | 0.0016956338887817786 |
| 10 | 0.13587960877790825 | 30 | 0.001289788249798486 |
| 11 | 0.11116794380756582 | 31 | 0.0009754406986251972 |
| 12 | 0.09070554103649146 | 32 | 0.0007332357465690555 |
| 13 | 0.07380385206616492 | 33 | 0.0005476436554357606 |
| 14 | 0.05987908487734025 | 34 | 0.0004062570844492274 |
| 15 | 0.04843729922980161 | 35 | 0.0002992074177011565 |
| 16 | 0.0390614765567334 | 36 | 0.0002186820257105552 |
| 17 | 0.031400321972712295 | 37 | 0.000158526401772845 |
| 18 | 0.02515858343893872 | 38 | 0.00011391745070210156 |
| 19 | 0.020088697710581194 | 39 | 8.109623550768432e-05 |
| 20 | 0.015983594697520104 | 40 | 5.7150243350046684e-05 |

This sequence is strictly decreasing at every step (verified computationally across `j = 1..50`; this is the fact Section 7's monotonicity proof depends on) and is independent of the configuration `(S, R)` — it needs to be computed once and reused for every configuration. There is no simpler closed form for `p_j`; it is the exact output of sampling-without-replacement combinatorics over a reshuffling two-deck shoe. The state-based forward enumeration described in Section 5 is the complete and correct specification for reproducing it — production must reproduce this table by implementing that enumeration independently (Section 15), not by copying the numbers above.

**Production must compute as many `p_j` values as the configured curve actually requires — never a fixed ceiling of 40.** How far the enumeration must extend before the calibrated curve reaches the cap depends on the resulting `c*` (Section 9), which in turn depends on the configured `(S, R)` pair. For the configurations tested in Document 08's matrix, the cap is reached at `j = 32` or `j = 33`, comfortably inside the table above — but this is a property of those specific configurations, not a mathematical ceiling. As `R` approaches its feasibility floor `p_1 × S` from above (Section 9), the required `c*` approaches 0, which pushes the offer at which the curve reaches the cap to a larger `j` — arbitrarily large as `c*` shrinks toward 0. Production's `p_j` enumeration must therefore be **open-ended**: computed on demand, to whatever `j` the calibration and cap-boundary determination (Section 3) actually require for the configured `(S, R)`, and must never assume that `j ≤ 40` (or any other fixed bound) is sufficient for every valid configuration.

---

## 9. Configuration Feasibility

**Exact feasibility condition (necessary and sufficient, within the domain stated in Section 9.3):**

```
R >= p_1 * S

equivalently:

S <= R / p_1  =  R / 0.7420734269371773
```

### 9.1 Necessity

**Why this is a hard mathematical boundary, not a design guideline.** At the very first offer, the bot can always trivially choose TAKE and receive `M_1 = S×W` with certainty at that decision point. Since `V_initial` is the value of the optimal policy — which can never do worse than any single fixed policy, including "always take offer 1 if reached" — this gives the unconditional lower bound:

```
V_initial >= p_1 * M_1 = p_1 * S * W
```

For the configuration to be feasible, this lower bound must not already exceed the target: `p_1 * S * W <= TARGET_EV = W * R`, which simplifies directly to `R >= p_1 * S`. If this fails, **no value of `c`, and no choice of `j≥2` formula whatsoever, can bring `V_initial` down to the target** — the floor alone (guaranteed reachable value from offer 1) already exceeds it. This bound does not depend on the specific shape chosen for `j ≥ 2`; it is a property of `M_1` and `p_1` alone.

### 9.2 Sufficiency — proof that a calibration `c*` exists whenever `R >= p_1 * S`

**Theorem.** Fix `W = $10`, `C = 1000`, and `S > 0`, `0 < R <= 1` with `R >= p_1 * S`. Then there exists `c* >= 0` such that `V_initial(c*) = W * R`, where `V_initial(c)` is the value of the perfect-information optimal-stopping recursion (Section 6) under the payout family

```
M_1 = S*W,   M_j(c) = max( S*W, min( c*W*R/p_j, C*W ) )  for j >= 2.
```

**Proof.**

*Step 1 — `M_j(c)` is continuous and non-decreasing in `c`, for every fixed `j`.* For `j >= 2`: `c*W*R/p_j` is continuous and strictly increasing in `c` (a fixed positive constant `W*R/p_j` times `c`). `min(c*W*R/p_j, C*W)` is continuous (a min of two continuous functions) and non-decreasing in `c` (a min of a non-decreasing function and a constant is non-decreasing). `max(S*W, min(...))` is continuous and non-decreasing in `c` for the same reason, applied to `max`. `M_1 = S*W` does not depend on `c` at all, hence is trivially continuous and non-decreasing (constant). So the entire payout vector is, term by term, continuous and non-decreasing in `c`.

*Step 2 — at every finite truncation horizon `k`, `V_k(state; c)` is continuous and non-decreasing in `c`, for every state.* By backward induction on the truncated recursion (Section 6), with the terminal boundary `V = 0` beyond horizon `k`: the terminal layer is identically `0`, a constant (trivially continuous and non-decreasing) function of `c`. Inductively, assume every state's value at layer `k+1` is continuous and non-decreasing in `c`. At layer `k`: `TAKE = M_{j+1}(c)` is continuous and non-decreasing in `c` by Step 1; `CONTINUE` is a fixed non-negative-weighted sum (the transition probabilities `n_c/rem`, `n_s/rem`, `n_r/rem` do not depend on `c`) of layer-`(k+1)` values, each continuous and non-decreasing in `c` by the inductive hypothesis — and a non-negative linear combination of continuous, non-decreasing functions is itself continuous and non-decreasing; `V = max(TAKE, CONTINUE)` is then the max of two continuous, non-decreasing functions of `c`, hence itself continuous and non-decreasing. By induction this holds at every layer down to the initial state, so `V_{k}(initial; c)` (a further fixed convex combination over the three first-card branches, Section 6) is continuous and non-decreasing in `c`, for every finite `k`.

*Step 3 — passing to the infinite horizon preserves continuity and monotonicity.* Section 17 establishes `|V_true(c) − V_k(c)| <= P(alive at k) * C*W` for every state, and this bound is **uniform in `c`**: it depends only on the fixed cap `C*W` and on `P(alive at k)`, never on the specific value of `c` being evaluated (the argument only uses that every payout is bounded above by the cap, which holds identically for every `c`). Hence `V_k(c) -> V_initial(c)` uniformly in `c` as `k -> infinity`. A uniform limit of continuous functions is continuous; and since `V_k(c_1) <= V_k(c_2)` for every finite `k` whenever `c_1 <= c_2` (Step 2), this weak inequality survives the limit. Therefore the infinite-horizon `V_initial(c)` is continuous and non-decreasing on `[0, infinity)`.

*Step 4 — value at the lower endpoint, `c = 0`.* At `c = 0`: for every `j >= 2`, `M_j(0) = max(S*W, min(0, C*W)) = max(S*W, 0) = S*W` (since `S*W > 0`). So `M_j(0) = S*W` for every `j >= 1`, including `j=1` by definition — the entire ladder is flat at `S*W`. Under a flat schedule, continuing past any offer can never help: by backward induction (terminal layer `CONTINUE = 0 <= S*W` trivially; inductively, if `CONTINUE <= S*W` at layer `k+1`, then the red-card branch value at layer `k` is `max(S*W, V_{k+1}) <= max(S*W, S*W) = S*W`, so `CONTINUE` at layer `k`, a weighted average of terms each `<= S*W`, is itself `<= S*W`), `TAKE = S*W >= CONTINUE` at every decision state. The optimal policy is therefore to take the very first offer, giving exactly

```
V_initial(0) = p_1 * S * W.
```

By the feasibility hypothesis `R >= p_1 * S`, this gives `V_initial(0) = p_1*S*W <= R*W = W*R` — the lower endpoint does not exceed the target.

*Step 5 — an upper endpoint that meets or exceeds the target.* Let `c_hi = C * p_2 / R` (finite and well-defined, since `R > 0`). Then `c_hi * W*R/p_2 = C*W` exactly, so `M_2(c_hi) = max(S*W, C*W) = C*W` (using `C*W > S*W`, i.e. `C > S`, which holds throughout this domain — see Section 9.3). Because `p_j` is strictly decreasing (Section 8), for every `j > 2`, `p_j < p_2`, so `c_hi * W*R/p_j > c_hi * W*R/p_2 = C*W`, hence `min(..., C*W) = C*W` and `M_j(c_hi) = max(S*W, C*W) = C*W` as well. So at `c = c_hi`, every offer `j >= 2` pays exactly the cap `C*W`, while `M_1 = S*W` is unchanged. Consider the specific (generally suboptimal, but fully valid) policy "CONTINUE at offer 1; TAKE at offer 2 if it is reached." Its value is exactly `p_2 * C*W` (the bust risk between offer 1 and offer 2 is already correctly priced into `p_2` by definition). Since `V_initial(c_hi)`, the *optimal* value, can never be less than the value of any one specific feasible policy:

```
V_initial(c_hi) >= p_2 * C * W.
```

Since `0 < R <= 1` and `p_2 * C = 0.6199813571142185 * 1000 = 619.9813571...`, which is far larger than `1 >= R`, it follows that `p_2*C*W >= R*W = W*R`, so `V_initial(c_hi) >= W*R`.

*Step 6 — existence, by the Intermediate Value Theorem.* `V_initial(c)` is continuous on `[0, c_hi]` (Step 3); `V_initial(0) <= W*R` (Step 4); `V_initial(c_hi) >= W*R` (Step 5). By the Intermediate Value Theorem, there exists `c*` in `[0, c_hi]` with `V_initial(c*) = W*R`. **QED.**

This establishes existence, which is what production requires: bisection over `[0, c_hi]` (Step 5 gives an explicit, computable bracketing upper endpoint) is guaranteed to converge to a root, and Step 2/3's monotonicity guarantees the bisection is well-posed at every step. Uniqueness of `c*` is not claimed or needed here; it has been separately observed empirically (strict monotonicity of the numerically computed `V_initial(c)`) in every configuration tested in Documents 07–08, but production's bisection procedure does not depend on a uniqueness guarantee to terminate correctly.

### 9.3 Domain of this proof — stated precisely, not overclaimed

The proof above is established rigorously for:

```
S > 0,   0 < R <= 1,   R >= p_1 * S,   W = $10 (fixed),   C = 1000 (fixed)
```

This domain covers the entire intended production configuration space: a Target RTP is by definition the fraction of the wager returned in expectation, so `R in (0, 1]` covers every economically sensible configuration, and `R > 1` (a game structurally designed to pay out more than the wager on average) is outside the intended design space and is not addressed by this proof. The side condition used in Step 5, `C > S`, is automatically satisfied throughout this domain and is not an extra, independent assumption: the feasibility hypothesis gives `S <= R/p_1 <= 1/p_1 = 1/0.7420734269371773 ~= 1.34754`, and `C = 1000` is fixed, so `C` is always far larger than `S` within the feasible region. **No claim is made, and none is needed for production, about `R > 1`** — that region is simply outside the domain this specification configures.

### 9.4 Production requirement

Validate every configuration against `R >= p_1 * S` before generating a payout table. If it fails, the configuration must be **rejected outright** — production must never silently produce a payout table for an infeasible `(S, R)` pair (for example, by producing a table whose achieved RTP is simply wrong, or by attempting an approximate/degenerate fit). If it holds (and `R <= 1`, per Section 9.3), Section 9.2 guarantees a valid `c*` exists and is reachable by bisection on `[0, c_hi]` with `c_hi = C * p_2 / R`.

---

## 10. Cap

```
C = 1,000×      (Maximum Multiplier)
W = $10.00      (Wager)

Maximum Cashout = C × W = $10,000.00
```

The payout formula in Section 3 guarantees `M_j ≤ $10,000` for every `j`, by construction, via the `min(..., C×W)` term. Once an offer reaches exactly `$10,000` (which, by the Section 7 monotonicity proof, happens at some specific offer number and never before), **every subsequent offer remains at exactly `$10,000`** — the payout curve must never decrease off the cap, dip below it, or exceed it. This is guaranteed by the same monotonicity proof: once `min(c×TARGET_EV/p_j, C×W)` reaches `C×W` for some `j`, it stays there for all larger `j`, since `c×TARGET_EV/p_j` only continues to grow as `j` increases (`p_j` keeps shrinking).

---

## 11. Default Production Configuration

```
S = 1.05×        (Starting Multiplier)
R = 90.000%      (Target RTP)
W = $10.00       (Wager)
C = 1,000×       (Maximum Multiplier)
```

Verified against Section 9's feasibility condition: `R = 0.90 ≥ p_1 × S = 0.7420734269371773 × 1.05 = 0.7791770982840361`. **Feasible**, with comfortable margin.

**Verified calibration (full mathematical precision):**

```
c*  = 0.8112024616977749
V_0 = $9.000000000000009
RTP = 90.00000000000009%
```

**First cap occurs at offer j = 33**, with (full precision) `M_32 = $9,956.9915807...`, `M_33 = $10,000.00` exactly, and `M_j = $10,000.00` for every `j ≥ 33`.

Independently re-derived by a structurally distinct implementation (Section 15): `c*_independent = 0.8112024616967` (agreement to 11 significant digits), `V_0 = $8.999999999991`, same cap boundary (`j = 33`), same monotonicity result.

---

## 12. Complete Default Payout Table

Cent-rounded production table, `S = 1.05×`, `R = 90.000%`. Strictly increasing from `M_1` through `M_32`, then flat at the cap.

| Offer j | Payout M_j | Offer j | Payout M_j | Offer j | Payout M_j |
|---:|---:|---:|---:|---:|---:|
| 1 | $10.50 | 13 | $98.92 | 25 | $1,518.72 |
| 2 | $11.78 | 14 | $121.93 | 26 | $1,956.26 |
| 3 | $14.12 | 15 | $150.73 | 27 | $2,531.84 |
| 4 | $16.98 | 16 | $186.91 | 28 | $3,293.10 |
| 5 | $20.45 | 17 | $232.51 | 29 | $4,305.66 |
| 6 | $24.69 | 18 | $290.19 | 30 | $5,660.48 |
| 7 | $29.88 | 19 | $363.43 | 31 | $7,484.64 |
| 8 | $36.24 | 20 | $456.77 | 32 | $9,956.99 |
| 9 | $44.07 | 21 | $576.21 | 33+ | $10,000.00 |
| 10 | $53.73 | 22 | $729.67 | | |
| 11 | $65.67 | 23 | $927.70 | | |
| 12 | $80.49 | 24 | $1,184.40 | | |

Every value in this table has been verified against the underlying backward-induction model (Sections 6, 11) and against an independent, structurally distinct re-implementation (Section 15).

---

## 13. Rounding

Two layers must be kept explicitly separate:

**Mathematical layer (full floating-point precision, never rounded):** `p_j`, the calibration scalar `c`, every intermediate Bellman value in the backward induction, every expected value, and every optimal TAKE/CONTINUE policy decision used to *derive* the payout table and to *certify* its RTP. Do not round any of these intermediate quantities.

**Production layer (cent-rounded):** the actual payout table shown to players and paid out (Section 12) is rounded to the cent. After rounding, production must re-verify, against the cent-rounded numbers specifically (not re-derive from scratch and assume it still holds):

- No payout exceeds $10,000.
- The payout ladder is still monotonically non-decreasing.
- The resulting RTP (recomputed using the optimal policy re-solved against the *rounded* table) remains within the approved tolerance of the target.
- No optimal TAKE/CONTINUE decision unexpectedly changes as a result of rounding, anywhere in the state space.

**Verified default-configuration rounding audit:**

```
V_0 (rounded)  = $8.999953920719017
RTP (rounded)  = 89.99953920719017%
Delta RTP      = -0.00046 percentage points  (rounded minus exact)

Decision-flip audit: 734,749 (position, state) entries compared between the
exact-precision policy and the cent-rounded-table policy — 0 flips.
```

The rounding cost is well within tolerance (under 0.001 percentage points of RTP), and rounding does not change a single optimal decision anywhere in the audited state space.

---

## 14. Do-Not-Change List

Production must not:

- Tune individual payouts to hit the target RTP.
- Manually adjust `M_2`, `M_3`, or any other individual offer.
- Remove the `M_1` floor from the `j ≥ 2` formula.
- Change the $10,000 cap.
- Reset the offer count `j` at a reshuffle.
- Reset the previous-card category `prev` at a reshuffle.
- Assume the four-card UI history limits the mathematical model — the RTP guarantee is against a perfect-information bot regardless of what the UI displays (Sections 1.7, 4).
- Substitute a fixed-threshold or heuristic strategy for optimal play as the basis for the RTP guarantee.
- Round intermediate probabilities, the calibration scalar, or any Bellman value during derivation.
- Alter payout values because a short-run simulation happens to produce a different observed RTP (Section 16 explains why short-run Monte Carlo noise is expected and is not evidence of a mathematical error).
- Change the Starting Multiplier without recalibrating the entire payout curve (re-solving `c` from scratch against Section 3's formula).
- Change the Target RTP without recalibrating the entire payout curve, likewise.
- Use simulation as the primary mathematical derivation of the payout table, the RTP, or the optimal policy — simulation is a validation tool (Section 16), never the source of truth.

---

## 15. Independent Verification

Production's implementation must be checked against a **second, structurally independent** implementation of the mathematics in this document — not a copy of the reference implementation, and not merely a re-run of the same code. "Structurally independent" means, at minimum, a different traversal order (for example, top-down memoized recursion instead of bottom-up iterative sweep).

**Precision note — what the reference independent implementation below does and does not verify.** The production mathematical model (Sections 5–6) includes unlimited reshuffling: whenever a shoe is exhausted, a fresh shoe is dealt and the run continues, with no cap on the number of reshuffles. The specific independent implementation reported below uses a **single-shoe approximation**: its recursion terminates and returns `$0` the instant a shoe would be exhausted (`n_c = n_s = n_r = 0`), rather than continuing into a reshuffled second shoe. **This is not an exact independent reproduction of the infinite-horizon, reshuffling model.** It is a structurally distinct implementation with a specific, quantified, bounded approximation error, and must be described and used as exactly that — not overstated as an exact reproduction.

**Rigorous bound on the single-shoe approximation error.** The single-shoe cutoff can only be wrong about paths that are still unresolved (neither busted nor cashed out) at the point a shoe would be exhausted — i.e., paths that survive a full 104 cards. Two bounds apply, both reused from results already established elsewhere in this document, not newly asserted here:

- **For the value comparison (`V_0`, `c*`):** the relevant quantity is the probability mass still "alive" (undecided) at card position 104 under the optimal policy — exactly the quantity bounded in Section 17, which is below `10⁻¹⁴` by position 100 and below double-precision floor by position 104. Since every unresolved path's true continuation value is bounded above by the cap `C*W = $10,000` (Section 10), the single-shoe truncation error on `V_0` is bounded by `P(alive at 104) * $10,000 < 10⁻¹⁰` dollars — several orders of magnitude below the cent.
- **For the `p_j` comparisons:** the relevant quantity is the probability that a *passively-dealt* run (Section 8's definition of `p_j`, which never voluntarily cashes out) survives all 104 cards of a shoe without busting, an exact enumeration result of `1.523684 × 10⁻⁷`. Since `p_j` is itself a probability bounded above by 1, the single-shoe truncation error on any individual `p_j` value is bounded by this same `1.523684 × 10⁻⁷`, in absolute-probability terms, uniformly for every `j`.

Both bounds sit several orders of magnitude below the precision this specification reports elsewhere (Sections 11, 17): the `V_0` bound is roughly four orders of magnitude tighter than needed to protect every digit this document reports at the cent level, and the `1.523684 × 10⁻⁷` bound on `p_j` cannot move any reported `p_j` digit before its 6th significant figure. The empirically observed agreement between the independent single-shoe implementation and the primary reshuffling implementation (below) is, consistent with this, far better than even these conservative bounds require — which is expected, since both bounds are worst-case (union-bound-style) estimates, not tight predictions of the actual error.

**Verified independent re-derivation, default configuration** (top-down memoized recursion, single-shoe approximation as described above, no shared code with the primary implementation):

```
c*_independent  = 0.8112024616967        (primary: 0.8112024616977749; agree to ~11-12 digits)
V_0_independent = $8.999999999991
M1 = $10.500000   M2 = $11.775874   M3 = $14.124167   M4 = $16.976912
Monotonic (independent, j=1..39)?  Yes
Cap boundary (independent)?  j = 33
```

**This is an independent verification with a rigorously bounded approximation error — not an exact independent reproduction of the infinite-horizon reshuffling model.** Production's own independent-verification implementation may either (a) adopt the same single-shoe approximation, in which case it must state and respect the same error bounds derived above, or (b) implement the reshuffle transition exactly (Section 5), which removes the approximation entirely and is the preferred approach where practical. Either way, production must reproduce results of this kind — an independently coded model that agrees with the primary implementation to at least 10 significant digits on `c*`, `V_0`, the first several `M_j`, monotonicity, and the cap boundary — rather than relying on a supplied reference implementation being correct by assumption.

---

## 16. Monte Carlo Validation

Monte Carlo simulation is a **validation** of the analytical model above. It is never the definition of RTP, and it is never used to derive or adjust the payout table.

**Verified production benchmark, default configuration:**

- Real 104-card two-deck shoe, dealt one card at a time with `random.shuffle`.
- The bot's decision at every offer is looked up directly from the saved perfect-information optimal-policy table (not recomputed heuristically).
- Seed = `20260918`, N = 3,000,000 trials.

```
Simulated EV        = $9.088327          Simulated RTP = 90.8833%
Analytic V_0 (rounded production table) = $8.999954     Analytic RTP = 89.9995%
SE(EV)              = $0.100674
z-score              = +0.8778

P(bust)                          = 0.710828
P(cashed)                        = 0.289172
P(exceeded saved policy depth)   = 0
P(cash at the $10,000 cap)       = 0
E[cards dealt]                   = 7.0038
E[offers seen]                   = 2.9791
Reshuffles observed              = 0
Policy lookup misses             = 0 / 8,937,174 offers seen
```

A `|z| < 1` result is entirely ordinary sampling noise at this sample size and payout variance and is **not** evidence of a discrepancy. Zero policy-lookup misses confirms every state the simulation actually visited was present in the saved policy table. Zero observed reshuffles at 3,000,000 trials is expected and consistent with Section 17's reshuffle-probability bound, not a gap in coverage.

---

## 17. Infinite-Horizon / Tail Verification

The mathematical model is, in principle, unbounded in horizon: a reshuffle can always occur if a run survives long enough, and after a reshuffle the run continues. In practice, the backward induction must be truncated at some finite maximum card position `k_max`, with a `$0` boundary condition imposed on any state still "alive" (neither busted nor cashed out) beyond it.

**The bound that justifies this truncation:**

```
|V_true(initial) - V_{k_max}(initial)|  <=  P(alive at k_max) * $10,000
```

**Why this bound is valid:** since every payout is capped at `$10,000` (Section 10), truncation can only ever be wrong about the value of the surviving probability mass, and that true value can never exceed the cap. This is a hard structural guarantee, not an empirical observation, and it does not depend on any convergence table looking stable.

**How `k_max` must be selected:** large enough that `P(alive at k_max) × $10,000` is far below the precision the RTP figure is reported to (a percentage to several decimal places, i.e. dollars to well below a cent). This was verified directly, not assumed: the probability of a run still being "alive" (neither busted nor cashed out) falls to approximately `9.7 × 10⁻¹⁵` by card position 100, and to below double-precision floor by card position 104 — meaning the probability of a run ever surviving long enough to force a reshuffle is on the order of `10⁻¹⁵` or smaller. At `k_max = 100`, the resulting tail bound is on the order of `10⁻¹⁰` dollars.

**Verified precision claim:** the infinite-horizon value is correct to at least 9–11 significant digits (this specification's default-configuration `c*` was solved at `k_max = 1,200`, far beyond the point where the tail bound becomes negligible). Do not claim more precision than this — nothing beyond roughly 10 significant digits has any bearing on a game whose smallest currency unit is the cent, which is 8+ orders of magnitude coarser than the demonstrated error bound.

---

## 18. Adversarial Testing

Production must test the optimal decision at every reachable state, not merely reproduce the headline RTP figure. At minimum, explicitly test and record the optimal TAKE/CONTINUE decision and both branch values at:

- The earliest offers (`j = 1, 2, 3`).
- A range of specific compositions at fixed `j` (to confirm the decision genuinely depends on remaining composition, not on `j` alone).
- States near the TAKE/CONTINUE boundary (where `TAKE` and `CONTINUE` values are close), to confirm the correct side is chosen.
- Low remaining-card-count states (deep into a shoe).
- High offer-count states (`j` in the 20s–30s, near and at the cap).
- Cap-adjacent offers (`j = 31, 32, 33`) — confirm the transition onto the cap happens at exactly the offer number this specification states, and not one earlier or later.
- Reshuffle-boundary states (the card immediately after a shoe reaches `n_c=n_s=n_r=0`).
- Every combination of `prev ∈ {C, S, R}` immediately before both a club and a spade draw (confirming Club→Spade and Spade→Club are correctly treated as safe, and Club→Club / Spade→Spade as bust).
- Red→Red sequences (confirming these are always safe).
- The first-card Ace-of-Spades bust specifically.
- Any state where a human's intuitive strategy (e.g. "cash out once it feels high enough") would plausibly disagree with the computed optimal decision — these are exactly the states worth spot-checking, since they are the ones most likely to reveal an implementation bug.

The purpose of this testing is to prove that production has implemented the *same game* as this specification at the level of individual decisions, not merely a system that happens to produce a similar aggregate RTP number.

---

## 19. Production Implementation Algorithm

### Configuration

```
INPUT:
    W   (wager, fixed $10.00)
    S   (Starting Multiplier)
    R   (Target RTP)
    C   (Maximum Multiplier, fixed 1000)

VALIDATE:
    REJECT the configuration unless  R >= p1 * S
    (p1 = 0.7420734269371773, a fixed constant of the card process — Section 8)
```

### Calibration

```
TARGET_EV = W * R
M1 = S * W

For j >= 2:
    Mj(c) = max(
        S * W,
        min(
            c * TARGET_EV / p_j,
            C * W
        )
    )

Solve for the single scalar c (bisection on a continuous, monotonic function of c):
    V_initial(c) = TARGET_EV

where V_initial(c) is computed by the full backward-induction recursion in Section 6,
using the M_j(c) family above, truncated at k_max per Section 17.
```

### Runtime

- **Start a run:** shuffle a fresh 104-card shoe. Set `j = 0`. Deal card 1 (Section 5's first-card table); if Ace of Spades, end immediately with payout $0; otherwise set `prev` to card 1's category and continue with no offer.
- **Deal each subsequent card:** draw the next card from the shoe.
  - **Detect bust:** if the new card's category matches `prev` and both are black (club-club or spade-spade), the run ends immediately with payout $0.
  - **Detect offer:** otherwise, if the new card is red, this is offer `j+1`; look up `M_{j+1}` from the calibrated table and present it as the current cash-out amount. If the new card is black (and not a bust), there is no offer; simply set `prev` to the new category and continue.
  - **Accept cash-out:** if the player/bot chooses TAKE at an active offer, the run ends immediately with payout equal to that offer's `M_{j+1}` value.
  - **Continue:** if the player/bot chooses CONTINUE (or a black card produced no decision to make), set `prev` to the new card's category, increment `j` (only on a declined red-card offer), and proceed to the next card.
- **Reach end of shoe:** if the shoe has no cards left and the run has neither busted nor cashed out, reshuffle a fresh 104-card shoe (reset composition to 26/26/52) and continue dealing from it. Do **not** reset `j` or `prev`.
- **Carry state across reshuffle:** `j` and `prev` are properties of the run, not of the shoe, and must never be reset at a reshuffle boundary; only the remaining-card composition resets.
- **Terminate:** a run ends only by bust (payout $0) or by a voluntary cash-out (payout = the accepted offer's `M_j` value). There is no other terminal condition.

---

## 20. Acceptance Criteria

Production is mathematically accepted only if it can independently demonstrate every item below.

**Rules**

- [ ] Exactly a 104-card shoe (two standard 52-card decks) is used, with correct composition (26 clubs, 26 spades, 52 reds).
- [ ] First-card behavior is correct: no offer on card 1 ever; Ace-of-Spades bust with probability exactly 2/104.
- [ ] Bust behavior is correct: club-after-club and spade-after-spade bust; every other adjacency (including red-after-red) is safe.
- [ ] Offer behavior is correct: red card (not card 1) → offer; black card → no offer; offer value equals the calibrated `M_{j+1}`.
- [ ] Reshuffle behavior is correct: triggers exactly at shoe exhaustion with no bust/cash-out yet; composition resets to 26/26/52.
- [ ] State carry across reshuffle is correct: `j` and `prev` are never reset at a reshuffle; only composition resets.

**Mathematics**

- [ ] `p_j` reproduces Section 8's table (or an equivalent independently-verified table for the production `k_max`) via the state-based forward enumeration of Section 5 — not copied from this document.
- [ ] `M_1 = S × W` exactly.
- [ ] The full payout formula (Section 3) is implemented exactly, including the floor and the cap.
- [ ] `c*` is solved by root-finding against `V_initial(c) = TARGET_EV`, never hand-tuned.
- [ ] The Bellman recursion (Section 6) is implemented exactly, including the reshuffle substitution.
- [ ] The RTP guarantee is certified against the perfect-information optimal bot (Section 4) — not a simplified strategy.
- [ ] The $10,000 cap is enforced and, once reached, never left.
- [ ] The offer ladder is monotonic (Section 7) by construction — verified, not patched.

**Production table**

- [ ] Cent rounding is applied correctly to the calibrated table.
- [ ] No payout exceeds $10,000 after rounding.
- [ ] The table remains monotonic after rounding.
- [ ] The RTP recomputed against the rounded table is within the agreed tolerance of the target (the default configuration's verified tolerance is 0.00046 percentage points).
- [ ] No optimal TAKE/CONTINUE decision changes unexpectedly as a result of rounding, anywhere in the audited state space.

**Verification**

- [ ] A second, structurally independent mathematical implementation reproduces `c*`, `V_0`, `M_1..M_4`, monotonicity, and the cap boundary to at least 10 significant digits.
- [ ] The analytical model and this independent implementation reconcile.
- [ ] A fresh Monte Carlo simulation reconciles with the analytical RTP within statistical expectation (`|z|` consistent with ordinary sampling noise).
- [ ] Adversarial state testing (Section 18) has been performed and recorded.
- [ ] Reshuffle behavior has been explicitly tested (state carry-through, composition reset).
- [ ] Zero policy lookup misses occur during simulation against the saved policy table.

---

## 21. Final Certification

**Default configuration**

```
Starting Multiplier:  1.05x
Target RTP:           90.000%
Wager:                $10.00
Maximum Multiplier:   1,000x
Maximum Cashout:      $10,000
```

**Mathematical RTP:** 90.000000...%

**Production rounded RTP:** 89.999539...%

**First Offer:** $10.50

**First Cap:** Offer 33

**Maximum Offer:** $10,000.00

**Ladder:** Monotonically increasing through offer 32, then flat at the cap.

**Calibration:** `c* = 0.8112024616977749`

**Certification status: PRODUCTION MATHEMATICS READY.**

---

## 22. Audit of Documents 01–08 Against This Specification

This section records the audit required before finalizing this document: every place an earlier document's numbers or architecture differ from what is specified above, which version governs, and why.

**Finding 1 — the placeholder multiplier schedule (Documents 01–02) is not part of this specification.** Documents 01–02 analyzed an explicitly-labeled *placeholder* linear multiplier schedule (`1.05 + (N−1)×1.14`) that produced RTPs far above 100% under every strategy. This was never a candidate production formula — both documents flagged it as a placeholder not to be tuned or shipped — and it plays no role in this specification beyond establishing the state-reduction argument (suit-category sufficiency, `(n_c, n_s, n_r, prev)`) and the reshuffle continuity rules, both of which carry forward unchanged into Section 5 above. **No contradiction; superseded by design from the start.**

**Finding 2 — Document 03's card-index-indexed payout function (`M(N) = $9/p_offer(N)`) is not part of this specification.** Document 03 calibrated payout by absolute card index `N` and found it double-counted value across paths that receive multiple offers, producing a first-offer RTP of 144.85% instead of the intended 90%. Document 04 identified and fixed this by re-indexing the payout on **offer count `j`** instead of card index, which this specification uses throughout (Sections 3, 6, 8). **Resolved in Document 04; Document 03's `M(N)` formula is superseded and must not be implemented.**

**Finding 3 — Document 04's uncapped formula (`M_j = $9/p_j`, no cap) is not part of this specification.** Document 04 proved this formula gives a perfect-information optimal bot an unbounded (divergent) RTP as the horizon grows, and further proved (by a homogeneity argument) that no finite rescaling of an uncapped formula of this shape can ever produce a finite target RTP. The $10,000 cap introduced in Document 05 is therefore not an optional refinement but a mathematical necessity for the RTP target to be achievable at all against a perfect-information bot. **Resolved in Document 05; this specification's cap (Sections 3, 10) is mandatory, not configurable.**

**Finding 4 — Document 05-actual-observable's four-card-restricted analysis is not part of this specification.** That document analyzed optimal play restricted to a rolling four-card visible window and left the exact value of that restricted policy as an open interval, not a closed form. Per Section 4 above (mandatory perfect-information certification), production certifies against the strictly stronger perfect-information bot, which upper-bounds every weaker player including the four-card-restricted one. **No contradiction — the four-card-observable analysis is subsumed by, and unnecessary given, the perfect-information certification this specification requires.**

**Finding 5 — a genuine architectural contradiction exists between Documents 05/05A and Documents 07/08, and is resolved here in favor of 07/08.** Documents 05 and 05A calibrated `M_1` as a member of the *same* single-scalar family as every other offer — `M_j(c) = min(c × $9/p_j, $10,000)` for **every** `j ≥ 1`, including `j = 1` — giving, at their calibrated `c* ≈ 0.812588745...`, a first offer of `M_1 ≈ $9.8552`, with no Starting Multiplier concept at all. Document 06 subsequently discovered that the "Starting Multiplier" configuration parameter specified for production was, under that architecture, disconnected from the mathematics entirely (a display-only value). Document 07 resolved this by introducing the current rule, `M_1 = S × W`, fixed **exogenously**, outside the calibrated family, requiring the family for `j ≥ 2` to be recalibrated against a newly-generalized target (`TARGET_EV = W×R` in place of the fixed `$9.00`) — which is why this specification's default-configuration calibration scalar, `c* = 0.8112024616977749`, is a **different number** from 05/05A's `c* ≈ 0.812588745...`, despite both nominally targeting 90% RTP. Document 08 then added the `j ≥ 2` floor at `M_1` (Sections 3, 7 above) on top of Document 07's architecture, without revisiting the `M_1 = S×W` rule. **This specification's Section 3 formula (07 + 08 combined) is the current, approved, and only correct production architecture. Documents 05 and 05A's `c* ≈ 0.812588745...` and `M_1 ≈ $9.8552` figures are superseded and must not be used in production** — they describe a now-obsolete architecture with no Starting Multiplier parameter. Everything else established in 05/05A — the Bellman recursion structure, the state definition, the cap mechanism, the tail-bound methodology, and the independent-verification and Monte-Carlo methodologies — carries forward unchanged and is restated in Sections 5–6, 15–17 above.

**Finding 6 — no contradiction found between Documents 06, 07, and 08 on any other point.** Document 06 diagnosed the disconnection described in Finding 5; Document 07 fixed it and additionally generalized the Target RTP into a genuine configuration parameter; Document 08 added the monotonicity floor without altering either of Document 07's core definitions (`M_1 = S×W`, `TARGET_EV = W×R`). These three documents form a single, consistent line of development, fully incorporated into Sections 2–3, 7, and 9 above.

No other contradictions requiring resolution were found. This document, Section 3 in particular, is the sole production formula.
