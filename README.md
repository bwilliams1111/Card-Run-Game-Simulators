# Card Run: Crash — Reference Prototype

A single self-contained `index.html` (vanilla HTML/CSS/JS, no build step, no dependencies beyond Google Fonts) implementing the Card Run "Crash" game, with its payout curve wired directly to the approved production mathematics in **[`09-final-production-specification.md`](./09-final-production-specification.md)** ("09 — Card Run: Crash. Final Mathematical Production Specification").

This is a **mathematical/reference tool, not production code** — see the parent project ("Card Run - Crash Math Simulator") for the full derivation history (Documents 01–09).

## What's wired in

- **Deck, bust, offer, and reshuffle rules** exactly as specified in Document 09 §1 / §5 / §19: two 52-card decks (104 cards), Ace-of-Spades bust on card 1, consecutive-club / consecutive-spade bust, an offer on every red card from card 2 onward, and reshuffle continuity (the offer count and previous-card category are never reset at a reshuffle — only the shoe composition resets).
- **The exact approved payout formula** (Document 09 §3): `M_1 = S×W`, and for every offer `j ≥ 2`, `M_j(c) = max(S×W, min(c×W×R/p_j, C×W))`, with the single calibration scalar `c` solved by bisection so that the perfect-information optimal-stopping value equals `W×R` exactly (Document 09 §6, §9).
- **The default production configuration** (`S = 1.05×`, `R = 90.000%`) is embedded exactly as verified in Document 09 §12 (`c* = 0.8112024616977749`, cap reached at offer 33) so the game is instantly playable with zero client-side computation on load.
- **Live recalibration**: changing Starting Multiplier or Target RTP in the Configuration panel re-runs the full pipeline — forward enumeration of `p_j`, Bellman backward induction, and bisection for `c*` — in a background Web Worker (so the page never freezes), and rejects any configuration that fails the feasibility bound `R ≥ p_1×S` (Document 09 §9) outright, rather than silently approximating it.

## What this does *not* do

- It does not implement the perfect-information optimal bot as an in-game opponent or advisor — that model exists only to *calibrate* the payout curve (Document 09 §4), not to play the game for the user.
- It is not certified production code; it is a faithful, verified reference implementation of the approved mathematics for demonstration and further review.

## Running it

Nothing to build. Open `index.html` directly, or serve the repo root with any static file server (this is also zero-config deployable on Vercel — a static `index.html` at the repo root needs no framework detection or build command).

## Verification performed before this was wired in

The JS math engine embedded in `index.html` was checked against the project's original Python reference implementation (`dynrtp_forward_full.py`, `dynrtp_backward.py`, `econ_monotonic.py`) and reproduces, for the default configuration: `c*` to ~9 significant digits, the full cent-rounded payout table exactly, and the cap boundary at offer 33. The card-level bust/offer logic was stress-tested against 2,000,000 simulated passive-dealer trials, reproducing Document 02's exact enumeration result `P(never see an offer before busting) = 0.2579265731` to within Monte Carlo sampling noise.
