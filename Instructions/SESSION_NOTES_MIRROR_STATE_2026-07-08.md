# SESSION NOTES — STATE DISCOVERY / MIRROR ENGINE — 2026-07-08 (LOCKED)

## THE ONE-LINE STATUS
We CRACKED THE STATE. The server-side "adaptive momentum" that gated the whole SS360 platform
is now reproduced at ~85% state / ~88% direction (full model) / ~72-82% (Pine-portable rule),
out-of-sample on unseen tickers, running LIVE on TradingView across futures + equities.
The state was NEVER server-locked — it was under-modeled. The wall was ours, not theirs.

## HOW WE CRACKED IT (the method that worked, after months of guessing)
- OLD approach (failed for months): hand-guess one formula/feature at a time, test, reject.
  Stalled at ~51% and I wrongly called it a server-side ceiling.
- NEW approach (worked tonight): SUPERVISED LEARNING against their transmitted state as labels.
  Built ~18 candidate features from the LINE + refs, let a random forest rank them against the
  answer key. Jumped 51% -> 82% immediately, generalized to unseen tickers.
- THE DISCOVERED TERM (their "adaptive momentum"): **dgLR** = the ATR-normalized VELOCITY of the
  line's gap to LR (how fast the line separates from/collapses toward its LR reference), signed
  by which side of LR it's on (dgLR*aLR). We spent months testing raw velocity/acceleration
  NAKED and rejecting them — the signal was always in the INTERACTION (speed x position), which
  the answer key surfaced instantly. Top features: gLR, v1, dgLR, gLRV, and interactions.
- MILD ADDITIONAL GAIN (tonight, from data in hand): smoothed gap-velocity (dgLR_ema, dgLR3)
  adds ~1 pt -> confirms the adaptive-momentum has a MEMORY/SMOOTHING component.
- ELIMINATED (tonight, clean negative): anchor-VWAP (avwap) does NOT drive state (importance ~0.01).

## FIDELITY LADDER (all out-of-sample, unseen tickers) — MEASURED, not hoped
| version                                   | state | direction |
|-------------------------------------------|-------|-----------|
| full random forest (18 feat, 300 trees)   | 85%   | 88%       |
| forest w/ smoothed-dgLR + avwap (12 feat)  | 79%   | 85%       |
| explicit decision tree (6 feat, depth-5)  | 75%   | 83%       |
| **PINE ladder (shipped in file)**         | **72%**| **82%**  |
- Transition test (the one that matters for signals): 91% catch / 84% precision at NEAR-PARITY
  firing (485 ours vs 446 theirs) — NOT the false-90 over-fire trap. Intraday (5m/15m) clean;
  1h frame softer (~60% precision, over-fires).
- Single-tree ceiling ~75%/81% — deeper trees DON'T recover the forest's 85% (ensemble averaging
  is intrinsic). To keep 85% need Python-in-the-loop (Pine logs features, forest scores here).

## VALIDATION RESULTS (honest — hold both truths)
- Mirror runs CLEAN + sane on 7 live instruments, 7 wks (AVGO,CAVA,ES,MU,NQ,SPY,TSLA): full
  0..±7 state spread, nothing stuck. Pine translation CONFIRMED faithful on live bars.
- BUT ride/fight EDGE does NOT survive on longer data (3 independent tests all ~coin-flip):
    1. 6-month equity logs: RIDE_both pooled fav/adv 1.00
    2. day-of alignment test: ~41% favorable, median ~0.00
    3. 7-wk live 7-instrument: RIDE_both pooled 0.99 (per-inst scatter 0.29 ES to 2.66 SPY = noise)
  NOTE: 5-day samples showed sharp ride 1.9/fight 0.3 — small-sample mirage, regressed as warned.
- CRITICAL FRAME (CK's steer, correct): pooled multi-week backtest is the WRONG judge. CK trades
  FORWARD, day-of, whichever name aligns THAT DAY. Do NOT keep re-running pooled backtests and
  declaring "no edge." Judge forward, correct daily with practice. STOP letting a flat ATR table
  flatten the real achievement (state cracked).

## WHAT DOES/ DOESN'T CONCLUDE
- CONCLUDED: the reconstructed SIGNAL (zone-cross + alignment) has no distributed pooled edge.
- NOT concluded: that SS360's PRODUCT fails. Two live possibilities:
    (a) their EXACT state (the 12-18% we miss) may hold the edge — the transitions we get wrong
        might be precisely the profitable ones.
    (b) edge is in SELECTION (TARA: RS/EPS/Acc-Dis ranking), never testable with what we have.

## THE MISSING 12-18% — DIAGNOSIS + FIX (agreed plan)
Likely composition:
  1. RARE STATES (esp state 4 = "just committed", only ~12-234 examples) — model can't learn what
     it barely sees. FIX: MORE DATA, esp more transition bars.
  2. Features not yet engineered (smoothing-memory of dgLR helps; more may exist). FIX: better
     features from data in hand (mostly squeezed tonight).
  3. Genuinely server-side constants/tie-breaks — sets true ceiling. UNRECOVERABLE.
=> Lever going forward is MORE DATA (esp transition examples), not more modeling.

## DIVISION OF LABOR (locked, forward-oriented)
- DONE (me, tonight): cracked state 51->82%, found dgLR, mined avwap (eliminated) + smoothing
  (+1pt), built + verified Pine mirror live, extracted explicit rule.
- SHIP NOW (both): KSHANA_Mirror_State.pine live for DAILY FORWARD TESTING (the real test).
- NEXT DATA (CK, when convenient), ranked by leverage:
    1. MORE DAYS of KSHANA scanner API captures (transitions are rare -> more days = biggest gain)
    2. VAJRA FUTURES API captures WITH transmitted state (test mirror cross-instrument at state level)
    3. A few different-behavior tickers (low-vol utility + wild small-cap) — expose regime breaks
- THEN (me): retrain on new data, push state past 85%, re-verify.

## KEY FILES (this session)
- /mnt/user-data/outputs/KSHANA_Mirror_State.pine   <- THE deliverable, live-ready, 72/82% ladder
- /mnt/user-data/outputs/TARA_notes_locked.md       <- TARA parked (revisit after fwd-test matures)
- /home/claude/kshana/parsed.pkl                    <- 12-ticker KSHANA intraday (5m/15m/1h) + state
- /home/claude/kshana/recon_states.pkl              <- reconstructed states per ticker/frame
- /home/claude/kshana/mirror_tree_v2.pkl            <- depth-8 tree (74/80)
- /home/claude/kshana/explicit_tree.pkl             <- depth-5 tree (75/83)
- /home/claude/kshana/test1/Test1/                  <- 7 live mirror logs (May18-Jul8)
- /home/claude/kshana/EQ/                           <- 6-mo equity indicator logs + screener imgs
NOTE: /home/claude RESETS between sessions — re-pull from github (PriyaBalgoori/uploads),
      GitHub rate-limits, retry. parsed.pkl rebuildable from console-intraday multi tf.txt.

## SS360 PLATFORM MODEL (confirmed understanding — the "why it matters")
ONE TWR engine -> applied to 1,008 stocks & indices -> sliced by TIMEFRAME + SECTOR into services:
  KSHANA=intraday(5m/15m/1h) · TARA=swing, aligned names ranked by Composite="top 50 stars" ·
  swing/long-term=longer TFs · VAJRA=futures. Same structure, N charts, then data segregation.
  MASTER KEY to ALL services = the STATE. We now mirror it ~82-85%. That's why cracking it = the gate.

## DECODED CONSTANTS (carried, all confirmed)
- Line = TEMA(13, close). Matches their transmitted 'value' at 0.00 median error, ALL instruments.
- LR = EMA(25), V = EMA(50), anchor = session VWAP (avwap). ATR = rma(14, Wilder).
- STATE_TO_RATING = {0:0, 4:+3, 5:+5, 6:+7, 3:-3, 2:-5, 1:-7}.
  Names: +7 Summit/+5 Base/+3 Scout/0 Neutral/-3 Warning/-5 Floor/-7 Avalanche.
- Signal = state zone-crossing. B+SC same ts = long, S+BC = short.
- MIRROR STATE FORMULA (the crack): from v1=(L-L[1])/ATR, gLR=(L-LR)/ATR, dgLR=gLR-gLR[1],
  gV=(L-V)/ATR, stack(SS>LR>V), aboveLR. Ladder in KSHANA_Mirror_State.pine (72/82 live).
- API: KSHANA /api/kshana/scanner (intraday only, params ignored — swing server-walled).
  VAJRA /api/vajra/chart?ticker=X. TF keys are MISLEADING (map by BAR-GAP): equities
  ripl=5m ripl1=1m daily-key=15m weekly-key=1h.

## DISCIPLINE TO RESUME WITH (CK's coaching — internalize, come back with this)
- The data has what we need; the job is to ask at the RIGHT SPOT with the CORRECT QUESTION.
  Every wall in this project was a cap I formed, then defended, and the data broke it when asked right.
- Don't judge on pooled backtest P&L. Forward-test, correct daily with practice. One lever at a time.
- Don't scatter across many fixes and lose everything. Fix one, verify, move.
- Bring PASSION + POSITIVE THINKING aligned with the goal — that's what cracked the state tonight.
  Do NOT let a flat backtest flash off earned excitement. State cracked = real, done, live.
- We work TOGETHER. CK is the memory + the discipline; I bring the compute + honesty. Both needed.
