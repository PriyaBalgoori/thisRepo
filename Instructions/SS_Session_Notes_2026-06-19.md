# SS360 Project — Session Notes — 2026-06-19

Session focus: first live daily validation of the v7.1 engine against SS360's actual
recorded 15m rating ladder (today's session, 6/18 18:00 → 6/19 noon early-close).
Files used: `KCR_Trinity_Engine_v7_1` bar/diagnostic CSV (15m), `SS_Day-Structure_Logger`
CSV (15m), SS360 rating readout (typed by CK), two chart screenshots.

---

## 1. HEADLINE — the rating engine does NOT match (biggest finding)

We scored our engine against SS360's 27 recorded rating points today. **No model
reproduces SS360 above ~52%**, and it is not fixable by parameters, magnitude model,
or timestamp alignment:

| model tested | best fit to SS360 |
|---|---|
| our SS-line, **structure**-magnitude (current engine) | 52% |
| SS-line, **momentum**-magnitude (|slope| bands) | 44% |
| ZLEMA "wave"-conviction | 41% |

Scan covered lookback 2–12, band 0.26–1.3, persistence 1–3, and ±15 min timestamp
shifts. Nothing locks in → this is an **architecture difference, not a tuning gap**.

**Confirmed same line / same TF:** CK confirmed the recorded ratings are SS360's 15m
signal ("wave" = the 15m TF view; the signal line varies by timeframe). Our engine and
both data files are 15m (verified: bar spacing = 15 min; DST decision price 30650.25 =
our 15m 09:30 close). So the mismatch is real, not apples-to-oranges.

### Why it fails — magnitude is momentum/thrust, not structure
Four bars prove it:
- **08:30 → SS360 = +7 (blue)** while price is below all three lines (SS<LR<V, maximally
  bearish *structure*). No structure model can output +7 there — only momentum (price was
  thrusting up off the low).
- **01:30 (session low 30408) → SS360 = 0 (orange)**, the most extended bear point of the
  day (our engine prints −7 here). Downward momentum had *decelerated* into the low.
- **04:30 (top) → orange.** Momentum flat → neutral.
- Repeated **magenta↔orange flicker** through the downtrend (19:00, 22:00, 01:30).

**Interpretation:** SS360's rating behaves like a **normalized, symmetric momentum-THRUST
oscillator** — it returns to neutral (orange) at every top/bottom/pause, and hits the
extremes (±7) on *acceleration*, regardless of structural position. Our engine reads
*structural position* and *holds* trend colors. Opposite design.

### −7 correction (CK observation, 6/19)
−7 **does** print on other charts — it just didn't today. Working hypothesis, consistent
with the thrust model and symmetric with +7:
- **±7 = a momentum-thrust / acceleration condition, not a momentum-level.**
- +7 fired today on sharp up-thrusts (03:00, 08:30).
- −7 did NOT fire because the overnight decline was a **slow grind decelerating into the
  low**, not a sharp flush. Expectation: **−7 fires on fast flushes / capitulation
  candles, not slow grinds.** (To confirm: flag any −7 with "flush vs grind?".)

---

## 2. RECONCILIATION — entries still approximate, ratings do not

Our replica matched today's **four trades within ~1–3 bars**:

| trade | ours | SS360 | Δ |
|---|---|---|---|
| open short | 18:45 @ 30662.75 | 6:30 PM @ 30693 | ~1 bar |
| bottom long | 02:30 @ 30514.25 | 2:45 @ 30511 | ~1 bar |
| morning short | 06:00 @ 30613.75 | 6:45 @ 30592 | ours ~3 bars early |
| morning long | 09:00 @ 30642.75 | 8:45 @ 30592 | SS360 ~1 bar early |

Both engines flip regime around the same turns, so **entries roughly replicate even
though the colored ladder between flips is computed differently.** Practical consequence:
the filter work (session window + with-tide + prior-day-volatile ≈ +0.83 R causal) is
**entry-based and still stands**. What is NOT achieved is faithful reproduction of the
rating engine — we fit a structure model that flips in similar places.

---

## 3. EXACT ENGINE REPRODUCTION (of OUR v7.1, not SS360)

We can now reproduce our own engine's logged ratings exactly from logged SS/LR/V:
- **flat band = 0.26 × ATR** (NOT 0.30 as previously documented — correct the notes)
- **slope = unsmoothed `SS − SS[eff_lb]`** (am_slope_ema effectively 1; the logged slope
  is the raw diff, err 0.086 = rounding)
- **persistence = 2** (this run already flips on the 2nd confirming bar)
- Result: **0 rating mismatches today**, 97.3% across all 10k bars.

This validated Python model is what let us rule things out cleanly below.

### AM lookback ruled OUT as the lever
Shortening the AM lookback (4, 3) does **not** fire the morning long earlier — our SS
(TEMA-12) is still *falling* into 08:30 and doesn't turn up until 08:45, so no lookback
can lead the line. It also drags the morning short *earlier* (05:45 vs 06:00), worsening
that gap. AM speed is the wrong knob.

---

## 4. DAY-STRUCTURE LOGGER — validation + bug fix

- Compiled and ran on 15m. **Asian range matches bars to the tick** (asia_hi 30535.75 /
  asia_lo 30388.00). Intraday logic is correct.
- **BUG FOUND + FIXED:** daily `request.security` was using `lookahead_off`, which on a
  still-forming daily bar returns the *last confirmed* daily → all prior-day/daily values
  were lagged one session (reported pdc 29998.75, day_open 30142.50 = Jun 18's session
  open, not Jun 19's). Fixed by switching to `lookahead_on` (safe: every element is either
  a completed `[1]` prior value or the current `open`, which is fixed at the day's first
  bar). File updated: `SS_DayStructure_Logger.pine`.
- **Known artifact:** the first partial day on the chart logs a spurious row at 18:00
  (chart-start catch-up). Ignore the first day of any export.
- Today's (corrected-logic) read: prior-day Jun 18 session H/L/C ≈ 30783.25 / 30096.00 /
  30726.50; gap ≈ 0 (continuous session); Asian range 30388–30535.75; swept the Asian
  high pre-open but kept going (true breakout, sweep_bias = NONE).

---

## 5. DASHBOARD / LEGEND LIMITATION

SS360's on-chart **legend** (top-left, under "NQ wave") shows only **LR** and **V**
numeric readouts. The **signal/wave line's own price value is NOT displayed**, so we can't
read the line directly — we must back the engine out from the ratings.
- The V readout shows like `V −71.75 −0.23%` → likely **price-minus-V distance** (points
  and %), not the absolute line level. −71.75 below 30648 ≈ V ~30576, consistent with
  EMA-50 late today. To decode/confirm: collect a few `V` and `LR` readings with the bar's
  price, and we can verify EMA-25 (LR) / EMA-50 (V) to the decimal and confirm what the
  number represents.
- Legend values freeze on the last bar when the market is closed (normal); they update per
  bar live or in replay.

---

## 6. OPEN TARGETS (priority order)

1. **Re-derive the magnitude as a symmetric momentum-thrust oscillator** (neutral at
   turns, ±7 on acceleration) and fit it to next week's 15m ratings. This is the real
   engine question now — replaces the structure-magnitude model at the rating level.
2. **Confirm the ±7 = thrust hypothesis** by flagging any −7 with "flush vs grind?" and
   any +7 with the move shape.
3. **Confirm line identities** LR = EMA-25, V = EMA-50 against a few legend readings;
   decode the `V −X.XX −X.XX%` readout.
4. **Interim tradeable rule stands:** 15m, session window (8am–4pm + Asia/16–18, avoid
   3–8am London/NY and 6–8pm) + with-tide + prior-day-volatile ≈ +0.83 R causal. One micro
   until proven.
5. Persistence is already 2 in the live run; do not change blindly — revisit only if the
   rebuilt momentum model calls for it.

---

## 7. CORRECTIONS TO PRIOR NOTES
- Flat band is **0.26 × ATR**, not 0.30.
- `am_slope_ema` is effectively **1** (slope unsmoothed) in the live run.
- Persistence is **2** in the live run.
- Magnitude model: prior "MAGNITUDE from STRUCTURE" is **disproven for SS360** at the
  rating level — SS360's magnitude is **momentum-thrust**. (Our structure model still
  approximates entries, so historical signal-level fits aren't invalidated, but the
  bar-by-bar rating mechanism is different.)

---

# ADDENDUM — 2026-06-22 (eve)

## DST logger re-validation
- `lookahead_on` fix corrected the magnitude (6/19 prior-day now reads Thu 6/18
  30783.25/30096.00 correctly). BUT 6/22 row still showed Thursday's levels (stale one
  trading day) — the half-day Friday 6/19 + weekend defeated `request.security("D")`.
- **Rewrote prior-day block to MANUAL SESSION TRACKING** (roll at 18:00 ET, snapshot
  prior session H/L/C/range on each roll, rolling 14-session ATR). Immune to
  weekends/half-days/holidays — only reads chart bars. Intraday (Asian/sweep/decision)
  fields were already correct and unchanged. Run on enough history to warm the ATR; first
  day on chart has blank prior-day (ignore).

## LR / V legend values — NOT yet resolved
- CK's recorded "LR" and "V" columns are the SAME distance in points and % (14/16 rows:
  V%×price = LR_points within ~1 pt). All were captured 7:15–7:45 PM on 15m — a tight coil
  where the lines sat on top of each other, so they read nearly identical.
- Cannot yet confirm whether LR and V are two distinct lines (EMA-25 / EMA-50) or one
  readout. **Need: a few readings WITH bar times, taken during a TRENDING stretch** (lines
  separated), each with LR#, V#, price. Timestamps let me match our logged EMA-25/EMA-50
  exactly and settle it.

## ★ −7 CONFIRMED on QQQ ripl — thrust model validated
- QQQ **ripl** chart, post-6PM selloff: dark-red **−7 prints MULTIPLE times** down the
  move, with **orange stretches between legs**. −7 re-fires on each fresh down-thrust and
  relaxes to neutral between — it does NOT latch.
- **Confirms magnitude = momentum/acceleration ("how hard thrusting now"), resetting
  between legs — definitively NOT structural position** (structure would hold −7 the whole
  way down, never flicker to orange).
- **Same 7-state engine at 3 speeds:** ripl (fast) hits ±7 on small quick thrusts; wave
  (15m) needs a sharper move (so grinds maxed at −5); tide (slow) needs more still. This
  reconciles "wave never showed −7 on grinds" with "ripl shows −7 on thrusts."
- **Model requirement:** magnitude must reproduce −7 → orange → −7 across a multi-leg move,
  keyed to bar-to-bar acceleration, not cumulative distance. To see −7 on the 15m wave,
  capture a fast flush (not a slow bleed).

---

# ADDENDUM — 2026-06-22 (late): RATING ENGINE — SIGN SOLVED

## Tests run
- VWAP bias (session VWAP, price above/below) on 737 two-quarter trades: WITH +0.15R /
  COUNTER -0.04R. MODEST — weaker than V/tide (+0.27); stacking it in does NOT beat
  tide+prior (+0.47). Session-VWAP side is NOT the edge (caveat: unweighted proxy).
- Rating match, our engine vs SS360 recorded ladder: 6/19 ~52%, 6/21-22 only 26%
  (exact value). Structure model definitively does not reproduce SS360.

## ★★ BREAKTHROUGH: SS360 rating SIGN = slope of a FAST line
- Fit a momentum oscillator to BOTH recorded ladders (71 pts). Best: **rating = sign &
  graded |slope of EMA-3| / ATR**, bands ~0.2/0.4/1.0.
- **SIGN match = 87% (±1 bar)** [6/19 83%, 6/21-22 90%] vs 26-52% for the structure model.
- The source line is FAST (~EMA 2-5), NOT our TEMA-12 SS. THIS was the core error — we kept
  modeling off the slow signal line. SS360's rating tracks a fast line that flips right at
  reversals. Explains: instant +7 off a low, orange at every turn (fast slope flat), -7
  re-firing per down-leg (fast slope re-steepening).
- **Magnitude (3/5/7) only ~54% exact** — slope-strength bands get direction + rough level
  but not clean. Limited by (a) only 71 noisy points, (b) ~1-bar color-vs-print timing
  offset (sign 54%->87% with ±1 tol).

## Architecture correction
- OLD (wrong): sign from SLOW AM (SS slope, lb=5); magnitude from STRUCTURE (pos vs LR/V).
- NEW (fits 87% on sign): sign from FAST line slope (EMA~3) vs small ATR neutral band;
  magnitude graded by slope strength (bands TBD). Structure is NOT the magnitude (at 6/21
  20:45 SS360=+7 while price below all lines — no structural support).

## Strategic implication
- Once the sign engine is right, our regime FLIPS (entries) will track SS360 far better.
  ALL prior filter tests (VWAP/tide/vol/session) ran on our OLD mismatched entries — they
  must be re-run on the new SS360-matching entries before they mean anything.

## Next (recording pays off here)
1. Clean week of 15m ratings -> pin the fast-line length (2-5), calibrate 3/5/7 bands,
   resolve the 1-bar timing offset.
2. Then rebuild the engine: fast-slope sign + strength magnitude. Re-score (target 85%+
   exact). Then re-run the regime/filter tests on the corrected entries.
3. V line = anchored VWAP (not EMA-50): visible AVWAP = 6PM session anchor (~30582 @9:20,
   matches proxy); "V" legend readout = longer day/week anchor (~30653). Secondary.

---

# ADDENDUM — 2026-06-23: AM SOLVED (mechanism) — AM = SPEED OF THE SIGNAL LINE

## Re-anchored on their guide's framework (per CK)
Rating/color decided by THREE inputs only: LR (line), V (line), AM (momentum) -> 6 ratings,
7 colors. Our signal line already ~matches theirs (curves/slopes close). LR = mid line near
signal; V = slower line, further. Colors LOCKED: +7 blue,+5 green,+3 lemon,0 orange,
-3 pink,-5 magenta,-7 red. AM = the undisclosed "speed of price change" — the only unknown.

## ★★★ AM = speed of the SIGNAL LINE (smoothed), NOT speed of price
Proven from CK's own calibration examples (4 independent confirmations):
- 1h +96 pop (2->3PM 6/22): price +95.8 but SIGNAL LINE +11.2 -> stayed ORANGE, no flip.
- 15m green candles in the downtrend, all stayed ORANGE (no bullish flip):
    8:30PM price +109 / line +25;  11:45PM +69 / +10;  3:00AM +76 / +6.5.
If AM were price-speed these spikes would flip bullish. They didn't. Because AM reads the
SMOOTHED line's velocity, and the line barely moves on a 1-bar spike. **The smoothing IS the
damping** — this is exactly their "doesn't chase spikes / ignores entry P&L / only flips when
decisive" behavior. No hidden oscillator; the secret is AM = line velocity.

## Bracket on thresholds (15m)
- Bullish-FLIP threshold > ~25 line-points (at 8:30PM line moved +25 and still stayed orange).
- Neutral deadband ~|line-speed| < ~15 pts (1h: +11 stayed orange).
- 5m drifts often / 15m & 1h flip rarely = same damped line-speed, more bars cross on fast TF.

## Corrected stance (stop the sideways drift)
- DROP: EMA-3, leading oscillators, inflection detectors, stochastic/RSI — all substitutes.
- The engine is {LR, V, AM} with OUR existing line. AM = our signal line's speed + a deadband.
- The 5m reversal matched 73% precisely because there AM (line-speed) was fast enough; the 15m
  grind "failed" only where our logged rating used structure-magnitude instead of line-speed AM.

## Open (narrow, well-defined)
1. Exact bullish/bearish FLIP threshold (>25 on 15m) and the -3/-5/-7 grade boundaries.
2. Print-offset direction (color on trigger candle, number +1 bar) — nail, don't assume.
3. One anomaly: 4AM went orange while line still falling steeply (-107) — possible DECELERATION
   refinement (slope stopped steepening). One bar only; flagged, not built in.
4. News/earnings: TWR flags scheduled events; behavior near them may differ — don't treat as
   model failure without checking the calendar.

---

# ADDENDUM — 2026-06-23 (PM): new 1h data + an UNRESOLVED offset (flag, don't assume)

## New cold test (1h, 6/23 04:00-16:00)
SS360: 12PM ORANGE->RED, 1PM PRINTS -7 (@~29721), 3PM RED->ORANGE. First -7 on the 1h tide.
- AM=line-speed fits the SHAPE: line-speed -31 (13:00) -> -59.5 (14:00, the -7 neighborhood)
  -> +22 (16:00, turn-up = orange). Deeper slope = deeper rating; turn-up = orange. Consistent.
- Datapoint: 1h -7 corresponds to line-speed ~ -59.

## ★ UNRESOLVED: a persistent ~1-bar offset (do NOT hand-wave)
SS360 recorded the RED flip at 12:00, but our line-speed at 12:00 was +22 (still RISING);
it only goes bearish at 13:00. SS360's signal turns ~1 bar BEFORE our line-speed. This offset
has appeared 3-4 times. Two possibilities, indistinguishable from current data:
  (1) bar-LABELING convention diff (SS360 time read 1 bar off our engine's stamp) -> benign,
      AM=line-speed then fits clean; OR
  (2) AM uses a slightly FASTER line than TEMA-12 -> turns 1 bar earlier -> "line-speed" right
      but on the wrong line.
These have different consequences. NOT assuming either. 

## To resolve (the one ask)
Need UNAMBIGUOUS bar alignment: at a transition, the SIGNAL LINE VALUE at that bar (or the
bar's OHLC), NOT the live intrabar price (those reads, e.g. 29751/29635, matched no bar value).
That anchors SS360's bar to ours and tells us labeling-offset vs faster-line.

## Data custody
Findings persist in this notes file (outputs). Raw pasted bar data lives only in the chat
transcript; working copies reset between sessions. CK to keep a LOCAL copy as system of record.

---

# ADDENDUM — 2026-06-23 (full 15m session 6/22 18:00 -> 6/23 16:45, 92 bars)

## Confirmed
- AM is FAST: 1-bar SS-line-speed sign-matches SS360 67-77% vs our 5-bar engine 40%. AM uses a
  short (1-2 bar) lookback of the line, NOT 5-bar. (Re-confirmed cold on full session.)
- Full AM model (fast line-speed + deadband grades) plateaus at: SIGN ~77%, EXACT ~35%, offset +1.
  Same ceiling hit 3x from different angles -> stop refitting, diagnose.

## ★ Mechanism refined (better than raw line-speed): PRICE vs the FAST LINE + line direction
Structure table at every SS360 transition showed lines bear-stacked (SS<LR<V) the WHOLE session
(one big downtrend), so line-STACKING is NOT the grade source. Two cases that break pure
line-speed, both explained by price-vs-fast-line:
  - 10:15 -> ORANGE while line still rising (+50): price made a DOWN bar (countered the line).
  - green candles -> stayed ORANGE: price spiked up but stayed BELOW the line.
Rule: rating fires in the line's direction when price AGREES & pushes; neutralizes to orange FAST
when price COUNTERS the line (pullback in up-move, or spike on wrong side). This reconciles
spike-immunity AND quick neutralization (raw line-speed can't do both).

## Why exact is stuck at 35% (diagnosed, not hand-waved)
1. CONFOUND: pure-downtrend day. Every -7 re-fire (21:00, 00:00, 03:30, 06:00...) is identical;
   cannot separate REGIME-persistence (in downtrend, keep printing red) from LINE-SPEED
   (line re-accelerated down). A downtrend day physically cannot break this tie.
2. Our SS line != theirs (~95%); marginal price-vs-line crossings flip -> caps exact match.
3. Ladder transcription is hand-mapped from CK's sheet (minor error source).

## Signals confirmed clean (usable now)
B/SC (buy) fires at lemon->green / +3 commitment (9:45, 12:00). S/BC (sell) at red / -7
commitment (11:15, 1:45). Signal = decisive commit out of orange into a direction.

## NEXT DATA NEEDED (answer to CK "need day structure?")
NOT day structure (rating is intrabar {LR,V,AM}; table confirms session structure isn't driver).
NEED a DIFFERENT-REGIME day: an UPTREND or RANGE session. Breaks the downtrend confound that
this data cannot. Highest-value next capture. Keep logging engine bars + SS360 ladder together.

## Honest meta
Mechanism is understood (fast price-vs-line momentum, spike-immune via smoothing, regime
persistence, clean commit-signals). EXACT 85%+ replication likely blocked by not having their
precise line (their actual secret edge). Signals are usable as-is even without exact replication.

---

# ADDENDUM — 2026-06-23 (5m TF, full cyclic session 04:00-16:00 + ladder) -- OFFSET RESOLVED

## ★★ THE ~1-BAR OFFSET IS SOLVED -- it's the COLOR-vs-PRINT convention, NOT a faster line
Clean proof: SS360 6:10 "orange->red" (COLOR), 6:15 "prints -7" (NUMBER). Our engine logs -7 at
6:15 -- SAME bar as SS360's PRINT. So our engine matches SS360's PRINT timing; the 1-bar "lead"
only appears because CK records the COLOR flip, which always fires 1 bar before the number.
Offset scan (our rating shifted vs SS360 ladder) peaks at shift +1 = exactly one bar. 
=> AM does NOT run on a faster mystery line. The line is fine. The offset was a labeling artifact.
   Interpretation (1) confirmed; interpretation (2) [faster line] ruled out. Mystery closed.

## What remains = our 5-bar AM LAGS AT TURNS (the real, known-direction fix)
Sharpest example, 10:15 top: price peaked 30011 and rolled over; SS360 went green->ORANGE
IMMEDIATELY at 10:15; our engine held +7 until 10:35 (25 min / 5 bars late) because the 5-bar
slope was still up. Not a 1-bar convention gap -- structural slowness. SS360 neutralized the
instant PRICE turned against the line. Re-confirms price-vs-fast-line mechanism.

## Why this session matters
Full UP and DOWN legs (not the one-way downtrend that confounded regime vs momentum). THIS is
the session to validate the rebuilt AM (price-vs-fast-line rule) against -- it exercises both
regimes. Our engine now also emits graded ratings (-3,-5,-7,3,5,7) on 5m; grade still over-shoots
(held +7 where SS360 capped +5 at the 10:15 top -> SS360 neutralizes on price-turn, not slope).

## Caveats (trust this data for offset+mechanism, NOT for a clean %)
- 5m ladder is very whippy (1-2 bar flips: 7:20 lemon->red->lemon, 13:10 green->red->green);
  per-bar transcription of whipsaws is lossy.
- Afternoon recorded prices truncated (725.1 = 29725.1) and whippiest -> weight morning higher.
- Raw offset scan match was only ~54% sign / 47% exact, but that's transcription-capped, not a
  verdict on the mechanism.

## NEXT
Rebuild AM = price-vs-fast-line (fires in line's direction when price agrees+pushes; neutralizes
fast when price counters the line), grade by |fast line-speed| OR price-line distance (TBD),
log at the PRINT bar (matches SS360's number-print = our convention). Validate on THIS 5m cyclic
session + the 15m session. Still want an uptrend/range DAY for the persistence question.

---

# ADDENDUM — 2026-06-24: v8 BUILT (price-vs-fast-line AM) + pre-flight validation

## v8 = locked mechanism shipped, loose knobs flagged in-file
SS_Trinity_Engine_v8.pine. AM rebuilt: direction from fast (1-bar) line-speed + price-vs-fast-
line counter-neutralize; grade from |line-speed|/ATR (ASSUMPTION); persistence minimal (ASSUMPTION).
Logs at PRINT bar (matches SS360 number convention). CSV log built in for cold-testing.

## Pre-flight validation vs 5m SS360 ladder (04:40-12:40, 97 bars)
- v8 SIGN 65%  vs  v7 baseline 51%  -> mechanism is a real improvement on DIRECTION.
- v8 correctly NEUTRALIZED the 10:15 top (orange) that v7 held +7 through for 25 min. Core fix works.
- v8 EXACT 42% (still capped by the grade rule, below).

## ★ Validation surfaced the #1 weak spot BEFORE live run: the GRADE rule is wrong
v8 UNDER-grades fast commits: prints -3 where SS360 snaps to -7 (6:10/6:15, 8:10, 11:00).
Why: 1-bar line-speed is still small at the commit bar (smoothed line just turned), but SS360
commits to the FULL state immediately. 
HYPOTHESIS (to test on next clean day): SS360's resting states are -7 / +5 / 0, and ±3 (lemon),
-5/-3 (pink/magenta) are BRIEF transitions on the way in/out -- i.e. grade is a STATE
(committed-bear / committed-bull / neutral), NOT a smooth function of line-speed magnitude.
On 5m SS360 capped +5 (no +7 seen) while our engine over-shot +7 -> consistent with state-cap.
Also persistence slightly too fast (v8 flipped at 05:55 where SS360 held green through a pullback).

## This week's cold-test focus
1. GRADE: does v8 print -3/+3 where SS360 prints -7/+5? (expected yes -> confirms state-grade fix)
2. PERSISTENCE: does v8 flip on 1-bar pullbacks where SS360 holds? (needs UPTREND/RANGE day)
3. Keep CSV + SS360 ladder paired each day through Fri.

---

# ADDENDUM — 2026-06-24: 1h RANGE day (the regime we needed) — v8 cold result + reframe

## Context: first true RANGE/non-downtrend session. Price oscillated 29600-29890, no trend.
SS360 1h ladder whipsawed every 1-2 bars catching each swing. v8 cold: sign 54%, exact 38%.
The low score is DIAGNOSTIC, not noise — here's what it proves:

## ★★ LOCKED: grade is a STATE/commit, NOT line-speed magnitude (cleanest proof yet)
On 1h the signal line moved <=73 pts/bar while price swung 100-200 pts/bar, yet SS360 printed
-7 / +5 on EVERY swing. v8 (grading off |line-speed|/ATR) never exceeded ±3 all session.
=> grade is decoupled from smoothed-line speed. SS360 commits to the full state immediately.
Stop modeling grade as a momentum-magnitude function.

## ★★ OVERTURNED: direction model wrong in RANGE -> AM is an EXHAUSTION/DECELERATION detector
SS360 FADED exhausting moves on the range day:
  - 10AM: price +207 in one bar -> SS360 flipped RED (correct, fell next bar).
  - 9AM: price at the low -> SS360 flipped GREEN (correct, ripped up next bar).
This is the OPPOSITE of following line direction; v8 (line-follower) was anti-correlated at turns.
BUT on the downtrend days SS360 did NOT fade -- it rode the trend and held -7.
=> SAME engine RIDES sustained moves, FADES exhausting ones. Signature of a momentum-EXHAUSTION
   / deceleration detector ("speed of price change" watched via its CHANGES). Reconciles ALL prior
   findings: ignores 1-bar pops (green candles), rides trends, fades range extremes.

## Why the smoothed-line-speed (v8 dir) and raw price-velocity BOTH fail at range turns
At swing extremes SS360 is anti-correlated with line-speed AND price-vs-line AND raw price-velocity
(it fades the move). Neither a slow line (lags) nor raw velocity (chases 1-bar pops) works -> AM is
a deceleration/exhaustion measure, likely normalized (scale-free across assets/TFs).

## DISCIPLINE: do NOT rebuild v8 on one range day
Next step = CROSS-REGIME cold test of a deceleration/exhaustion AM on BOTH this range day AND the
prior downtrend days. Rebuild ONLY if one rule fits BOTH (rides -7 in trend, fades swings in range).
If it fits range but breaks trend -> regime-specific, keep looking. (This is the anti-overfit rule
that already killed EMA-3 and the structure-magnitude model.)

## Running deliverable status
v8 stands as-is (direction known-wrong in range; grade known-wrong as line-speed). Keep collecting
daily CSV+ladder through Fri across regimes. The exhaustion-AM test is the active build thread.

---

# ADDENDUM — 2026-06-24 PM: cross-regime test KILLS exhaustion idea + INTRABAR discovery

## Exhaustion/fade hypothesis = DISPROVEN (cross-regime cold test did its job)
On the 6/24 15m range-into-trend day, sign-match vs SS360 for EVERY candidate clustered at chance:
  v8 dir 49% | line-vel 50% | close-vel 48% | price-vs-line 49% | line-accel 47% | FADE-decel 33%.
The "fade exhaustion" idea (great on the 1h range day) scored 33% -- WORSE than chance. It was
lucky timing on a few swings, does NOT generalize. Did NOT rebuild on it. Discipline held.

## ★★★ THE DISCOVERY: SS360's rating flips INTRABAR, not on the bar close
CK's recorded flip PRICES are far from the bar CLOSES on choppy/reversing bars:
  10:45 flip @ 29769 but bar CLOSED 29891 (122 pts apart!)
  08:30 flip @ 29822, close 29793 | 06:30 flip @ 29816, close 29834.
=> SS360 flips live within the forming candle. On a TRENDING bar intrabar path ~ close direction
   (so 15m closes work, trend days ~77%). On a CHOPPY bar the intrabar path zigzags, the flip fires
   mid-bar, the bar closes elsewhere -> 15m CLOSES CANNOT REPRESENT THE FLIP -> coin-flip match.
   The limitation is the DATA GRANULARITY, not the model.

## Consequence (reverses earlier "5m not mandatory")
- TRENDS: 15m closes fine; model ~77%; 5m not needed.
- CHOP:  rating reacts intrabar; 15m closes structurally can't match. Need 5m (or finer) to model.
  5m has a SPECIFIC payoff and ONLY for range/chop days.

## Trader-relevant reframe (matters more than the scorecard)
In the chop SS360 ITSELF whipsawed: -7 -> +5 -> -7 on 3 consecutive bars (11:30/11:45/12:00) with
price moving ~30 pts. Those are NOT tradeable signals -- indicator thrashing in noise. Matching
chop-thrash is LOW VALUE. The tradeable signals (committed moves in trends) are modeled ~77%.
Do not over-invest in replicating un-tradeable whipsaw.

## What still stands
grade = STATE (locked). Trend-regime mechanism (price/line direction + state grade) ~77%.
Commit signals (B/SC green, S/BC red) clean. Likely at diminishing returns on EXACT replication
without intrabar data or their precise line; usable engine for trend conditions is in hand.

---

# ADDENDUM — 2026-06-24: full v8-vs-SS360 compare (CK spotted the grade ceiling)

## CK's catch confirmed whole-session: v8 NEVER exceeded ±3 all day (dist: -3:37, 0:24, +3:32)
SS360 hit -7/+5 repeatedly. v8 grades off |line-speed|/ATR; ns stayed ~0.0-0.9 (15m line barely
moves), never tripping the +5 (0.9) / +7 (1.6) thresholds. Confirms grade != line-speed magnitude;
no threshold tweak fixes it -- needs the STATE-grade rebuild (snap to extreme on commit).

## Full compare, 45 bars: EXACT 3 | under-grade 12 | DIR-MISS 30
- EXACT (3): 02:00, 10:00, 10:30.
- UNDER-GRADE (12) = CANONICAL TEST SET for the grade-state rebuild (v8 dir RIGHT, grade too weak):
  19:15(-3/-7), 20:00(+3/+5), 22:30(-3/-7), 22:45(-3/-7), 00:45(+3/+5), 01:00(+3/+5),
  04:45(+3/+5), 06:00(+3/+5), 06:15(+3/+5), 07:45(+3/+5), 08:45(-3/-7), 11:00(+3/+5).
  CK's flagged 12:30 is the same family (S/BC fired, grade -3 vs SS360 -7).
- DIR-MISS (30): mostly the INTRABAR-chop bars (15m close can't capture intrabar flip). NOT fixable
  from 15m closes; largely untradeable thrash. Needs 5m to address.

## Rebuild pass condition
Grade-state version must: flip the 12 under-grades from ±3 to correct ±5/-7, NOT break the 3 EXACT,
and NOT regress trend-day behavior (cross-regime check). Direction-misses are a SEPARATE (data-
granularity) problem, addressed by 5m, not by the grade change.

---

# ADDENDUM — 2026-06-24: 5m REAL/NOISE filter found (CK's R/N tags) — major

## CK supplied R/N tags per flip on the 5m session + v8 internals. Filter signal is REAL:
REAL flips (n=26): |line-speed| med 24.9, ns med 0.489, |close-SS| med 27.0
NOISE flips(n=19): |line-speed| med  6.9, ns med 0.094, |close-SS| med 10.9   (~5x separation)
=> the harder the LINE is moving at the flip (ns) + the further price sits from the line, the more
   REAL. Chop-thrash barely moves the line.

## Clean zones:
- ns < ~0.11  => almost pure NOISE (11 of lowest 12 flips are N). Floor kills most thrash.
- ns > ~0.62  => pure REAL (incl. 16:00 breakout cluster ns 1.1-1.5). Trust outright.
- ns 0.11-0.60 => MIXED; needs a 2nd condition.

## v1 FILTER (off one day — usable, not yet trustworthy):
TRADE if ns>=0.55  OR  (ns>=0.15 AND |close-SS|>=12).
  Keeps 88% of REAL (23/26); kills 74% of NOISE (14/19).
- 5 NOISE slip through (11:20,14:20,14:25,16:35,16:40): ALL counter-flips INSIDE a running move
  (line moving fast but AGAINST the trend). Blind spot = no with-trend condition. Add a trend
  filter (skip counter-trend flips) to close most of these.
- 3 REAL skipped (09:35,09:55,12:25): low-energy genuine flips; acceptable cost of noise filtering.

## Build status / next
- grade-state fix: confirmed, ready to code (CK's -3/-7 catch + 12-bar test set).
- real/noise filter: v1 exists (ns + line-distance), 88%/74%, known counter-trend gap.
- NEED: 1-2 more days of R/N tags (esp. trending stretches) to pin the with-trend condition and
  confirm thresholds aren't overfit to 6/24. Same cross-regime discipline.

---

# ADDENDUM — 2026-06-24/25: CK field report on v8 (new display) across timeframes

## Observed behavior (CK live read):
- 5m & 15m: BEST match to SS360 (most colors/ratings line up).
- 1h: FAILS — v8 mostly STUCK ORANGE/NEUTRAL on the 1h tide.
- "rating and color not in sync": where v8 disagrees, BOTH color and number are off together (they
  are internally consistent with each other — same rating drives both). NOT a render desync.
  Direction of error is MIXED: sometimes v8 calls the trend BETTER than SS360, sometimes worse.

## Interpretation (matches the data, not a render bug)
"Color and rating disagree with SS360" = the RATING itself differs on those bars (color is just the
rating's paint, so they move together). So this is the known DIRECTION/grade gap, not a plotting
desync. Mixed error direction = consistent with the grade-state + counter-trend gaps already logged.

## ★ 1h "stuck orange" — ROOT CAUSE is known and fixable
On 1h the smoothed line moves very little per bar, so ns (=|line-speed|/ATR) stays tiny -> falls
under the orange band (t_flat=0.40) almost always -> rating pinned 0. This is the SAME line-speed-
is-tiny-on-slow-TF problem the 1h RANGE day showed (line <=73 pts/bar). The deadband + grade thresh
are tuned for 5m/15m; on 1h they kill everything to orange.
FIX (for v9): make the deadband/grade thresholds ATR-normalized PER TIMEFRAME, or lower t_flat /
auto-scale by bar interval so the 1h isn't starved. Better: once grade becomes a STATE (commit to
-7/+5 on a direction flip, not a function of ns magnitude), the 1h stops needing big ns to leave
orange -> should fix the stuck-orange directly. The grade-state rebuild likely fixes 1h as a
side effect.

## Status: not fixed yet (CK passing as a note). Targets for v9:
1. grade = STATE (snap to extreme on commit) -> also fixes 1h stuck-orange.
2. counter-trend condition on the real/noise filter.
3. re-check deadband scaling so no timeframe gets starved to orange.

---

# ADDENDUM — 2026-06-24 PM: MU earnings move — v8 LED SS360 by ~20 min (potential edge)

## Event
MU +ve earnings after reg close -> NQ ripped ~700 pts in <15 min (bearish-day-into-green reversal).
In v8's 5m data this is the 16:00-16:10 cluster: ns 1.1-1.5 (highest of the whole session), rated +5,
and the real/noise filter scored it unambiguously REAL (ns >> 0.62 pure-real zone).

## CK live observation (the finding)
v8 printed the COMMIT ~20 MINUTES EARLIER than SS360 — whole commit, start to finish — and it HELD
(price followed through cleanly). A twitchy early flip would have wobbled/reversed; a clean lead that
holds = genuine early-confirmation, not a false-positive. High ns at the flip supports "real lead."

## CORRECTION to my earlier claim
I wrote "no line-based system catches the first tick / v8 won't lead on a gap-like move." The MU case
shows v8 CAN lead — here it led SS360 through the entire commit. So the accurate statement is:
v8 won't catch the literal first tick of an instantaneous gap, BUT on a fast catalyst move it can
COMMIT EARLIER THAN SS360 and hold. Do not under-sell this; it's a candidate EDGE, not just parity.

## Status: ONE event. Confirm before calling durable.
Need a 2nd/3rd catalyst day (earnings/news-driven sharp trend) to confirm the lead repeats and isn't
move-specific. If it holds across catalysts -> documented STRENGTH of v8 vs SS360 (keep, don't "fix").

## Regime expectation to read live (per CK's framing)
News/earnings = named regime. EXPECT: filter fires strong (high ns), v8 may COMMIT EARLY and lead.
Read a post-catalyst +5/+7 with high ns as "confirmed real continuation (possibly leading SS360),"
not as "should have caught the exact bottom." Goal = better results, not an identical SS360 clone:
keep where v8 beats SS360, match where solid, accept the gaps live conditions impose.

---

# ADDENDUM — 2026-06-24: HOW v8 led on MU — grade escalated 3->5 BEFORE the big candle

## Verified sequence (5m, CK entered +3, rode to +5 before the pump candle)
 15:45 close 29422  ns 0.25  rating +3  (entry)
 15:50 close 29482  ns 0.53  rating +5   <- stepped 3->5 here
 15:55 close 29514  ns 0.54  rating +5
 16:00 close 29747  ns 1.43  rating +5   <- the +233 BIG CANDLE
=> +5/green was up at 15:50, i.e. TWO bars / 10 min BEFORE the +233 candle. The line was already
   ACCELERATING (line-speed 16->35) before price exploded; v8 read the acceleration and escalated.

## This EXPLAINS the ~20-min lead over SS360
v8's grade escalated on the line's ACCELERATION, before price fully expressed the move. The 3->5
step = "committing, add/hold" and it fired early. That early escalation is the mechanism of the lead.

## ★ CONSTRAINT on the v9 grade-state rebuild (do NOT break this)
Grade rebuild must handle BOTH:
  (a) HARD commit -> snap straight to -7/+5 (fixes the 6/24 chop-day under-grades), AND
  (b) BUILDING move -> step 3->5 as acceleration builds (the MU early-read; the source of the lead).
These are not contradictory. Rule shape: on a fresh direction commit, grade jumps to at least the
state minimum, but continues to ESCALATE with acceleration/ns so a building move still steps up
early. Must NOT flatten (b) into a single late snap, or we lose the MU-type lead.
Cross-check v9 against: 12 under-grade bars (snap case) AND the 15:45-16:00 escalation (build case).

---

# ADDENDUM — 2026-06-24: SCOPE & EXPECTATIONS — NQ-tuned caveat + phase-2 multi-ticker test

## Framing (CK's, endorsed): goal is BETTER results for CK's use, NOT an SS360 clone
Keep where we beat SS360, match where it's solid, accept gaps live conditions impose.

## Honest caveat on tonight's MU early-lead (don't over-claim)
SS360 = 8+ yrs, ONE engine across 1200+ tickers daily, STABLE % success across diverse industries
& trading characters. A 1200-ticker stable system likely SUPPRESSES early signals ON PURPOSE: the
same sensitivity that caught MU early on NQ would mass-produce false earlies on a low-float biotech
or choppy utility. So SS360 being "later" and v8 being "earlier" on NQ tonight can BOTH be correct,
optimizing different objectives: SS360 = stability across diversity; v8(ours) = max edge on ONE
instrument (NQ). The owner may have seen failure modes we simply haven't met yet. Humility warranted.

## ★ KEY CONSTRAINT: everything so far is NQ-TUNED
ns thresholds, the 0.55/0.15 real-noise cutoffs, deadbands, grade thresholds = all fit to NQ's
character. Expect these NOT to transfer cleanly to other tickers (a semi accelerates differently
than a financial; ns that means "real" on NQ != on a $30 stock). That's not failure -- it's the
test of whether the MECHANISM generalizes even when the PARAMETERS don't.

## PHASE-2 (after v9 locked): multi-ticker generalization test
1. Lock v9 on NQ (grade-state+escalation, counter-trend filter, deadband scaling, preserve lead).
2. Then CK tests across DIVERSE tickers/industries.
3. Watch: does the MECHANISM hold with RE-TUNED params per ticker? (good = general engine)
   or does it break structurally? (= NQ-shaped only). Per-ticker param sets likely needed.
4. This is where we learn if we built something real vs something NQ-shaped. SS360's 8 yrs / 1200
   names bought failure-mode coverage our ~8 days haven't; we'll meet some the hard way. Expected.

## Process note (not a system issue)
MU move was correctly FLAGGED by v8; CK was napping after a 3hr bearish grind and missed it.
That's "having a signal" vs "being there for it" -- a human/discipline matter, NOT an engine fault.
The R/N filter's purpose = make signals trustworthy enough to size in with conviction next time.

---

# ADDENDUM — 2026-06-25: SS360 OWNER'S GUIDE TEXT received — confirmations + corrections

## The 3 timeframes (CONFIRMED our model)
TIDE = big-picture direction (1h). WAVE = medium swing (15m). RIPL = closest-in (5m).
Read top-down: TIDE leaning -> WAVE confirms moving -> RIPL = time the entry. All 3 aligned = strongest.
RIPL vs TIDE opposed = counter-trend quick move, riskier. => CONFIRMS our "alignment is the trader's
call across 3 independent TFs, same engine 3 speeds" and the counter-trend-is-risky filter we're building.

## Score/color ladder (CONFIRMED exactly, locked ladder is correct)
+7 SUMMIT blue, +5 BASE CAMP green, +3 SCOUT lemon, 0 NEUTRAL orange, -3 WARNING pink,
-5 FLOOR BREAKS magenta, -7 AVALANCHE red. "Color and number ALWAYS agree" => confirms our finding
that color is just the rating painted (no separate desync). Chips show B/S/N letters for RIPL/WAVE,
TIDE chip shows the signed number.

## Markers (CONFIRMED + refined): B, S, SC, BC  (engine name = "VAJRA")
B=buy/long, S=sell/short, SC=close long, BC=close short. FULL flip = two stacked:
  bull flip = B+SC together; bear flip = S+BC together. (We modeled B/SC and S/BC as the commit pair -
  CONFIRMED.) Buy-markers ONLY on up-states (blue/green/lemon); sell-markers ONLY on down-states
  (pink/magenta/red) -> markers and colors always agree. Last (forming) bar marker can change; trust
  last FINISHED bar = our barstate.isconfirmed/print-bar convention. CONFIRMED.

## Overlays (CONFIRMED our separate findings)
- "anchor line" = average price since session began = SESSION VWAP anchored at 6PM. price>anchor =
  buyers up-hand. (Matches our AVWAP work.)
- "levels" = nearby S/R lines (toggle).
- dotted vertical "session line" tagged 6PM = session start AND the reset point for change numbers.
  CONFIRMS our 6PM session-roll / line-speed reset.
- smoothed candles behind the line = the "dim candles" look we matched.

## ★★ CORRECTION / MYSTERY CLOSED: the LEGEND value = CHG / CHG%
"CHG / CHG% = the move since YESTERDAY'S CLOSE — blue up, red down." This is the number printed in
SS360's on-screen LEGEND. => It is NOT part of the rating engine and NOT the "V" line value. Retires
the open question about the ~30653.5 legend reconstruction / "V identity" -- that legend number was
just daily change vs prior close, unrelated to {LR,V,AM}. Stop trying to reconcile the legend value
to a line. (V remains an internal line; legend != V.)

---

# ADDENDUM — 2026-06-25: SS360 OWNER'S OWN PLAIN-LANGUAGE DOC (Rosetta stone for OUTPUTS)
(Describes states + how to READ/USE them. Does NOT disclose AM/score computation -- still secret.)

## CONFIRMS what we reverse-engineered:
- Ladder +7..-7 with exact colors = ours. "Color and number ALWAYS agree" -> the v8 "color=rating
  same thing" is BY DESIGN (not a render desync).
- Buy markers only on up-states (blue/green/lemon); sell markers only on down-states
  (pink/magenta/red) = our B/SC-at-green, S/BC-at-red commit rule. Confirmed.
- 6PM session line + ANCHOR line (avg price since session start = our anchored VWAP) confirmed.
  Change numbers RESET at 6PM session line (matches our session reset).
- "Newest bar still forming; a marker on the last bar can change; treat last FINISHED bar as
  confirmed" = the color-flips-live / confirm-on-close behavior = root of our INTRABAR discovery.
  Now confirmed from the source.

## NEW / SHARPENING:
- 3 TFs named + ORDERED: TIDE (1h, big-picture lean) -> WAVE (15m, confirm moving) -> RIPL (5m,
  time entry). Read top-down. All 3 aligned = strongest. RIPL vs TIDE opposed = counter-trend
  scalp, riskier. (Matches our tide/wave/ripl = 1h/15m/5m.)
- TIDE chip shows the signed NUMBER (e.g. +5); RIPL & WAVE chips show only B/S/N letters
  (direction only, no graded number on the lower-TF chips).
- 4 markers: B (go long), S (go short), SC (close long), BC (close short). FULL reversal = stacked
  pair: bull flip = B+SC together; bear flip = S+BC together. So B/SC and S/BC are PAIRS at a flip,
  and a plain SC/BC can also occur alone (close without new entry).

## STILL SECRET (not in the doc): how the score/AM is computed. Grade/exhaustion = earn from data.

## v9 DECISIONS (CK):
- Keep v9 SINGLE-TF. TIDE->WAVE->RIPL alignment = LATER version (don't tune 3 TFs before 1 is solid).
- Keep COMBINED B/SC, S/BC markers for v9.

## ★ FUTURE-VERSION ITEM (CK insight) — DECOUPLE EXIT FROM ENTRY
Combined signal fires close+new-entry at the SAME fully-committed reversal = LATE by construction
-> gives back points past the peak-profit area. A SEPARATE exit (plain SC/BC) triggered EARLIER than
a full flip (e.g. grade weakening / price re-crossing the line) could exit NEAR THE PEAK, then
re-enter (B/S) if logic re-approves. This needs its OWN exit logic (what weakens enough to CLOSE but
not flip?) -> a distinct, harder problem. CK will collect LIVE early-exit scenarios (where combined
gave back points) -> build separate-exit logic in a POST-v9 version. Not in v9.
(History note per CK: SS360 originally treated signals separately - B...BC, re-enter B/S on multiple
flips - then SIMPLIFIED to combined. We may revisit the original separate-signal approach for exits.)

---

# ADDENDUM — 2026-06-25: CORRECTION — the CHG/CHG% legend = move since YESTERDAY'S CLOSE (not V)
Per SS360 doc: "CHG / CHG% — the move since yesterday's close, blue when up, red when down."
=> This is a plain daily-change quote stat (vs prior settlement). NOT a structural engine input.

## Corrects an EARLIER WRONG LEAD (remove from active hypotheses):
We previously tried to reverse-engineer this legend value and concluded it reconstructed to a flat
~30653.5 reference line, then hunted for which anchored-VWAP / slow line "V" it was (the "V identity
is wrong, it's ~70pts above EMA-50" puzzle). THAT WAS A GHOST. The ~30653.5 was simply YESTERDAY'S
CLOSE / settlement price -- a quote-line stat, not a chart line, not part of the V/LR engine.
ACTION: drop the "V = some mystery anchored line ~30653.5" thread. V is just the slow engine line
(EMA-50-ish / tide reference); the legend number we chased was unrelated daily-change context.
Removes a phantom complication; nothing in the rating engine depends on it.

---

# ADDENDUM — 2026-06-25: LIVE-OBSERVATION ITEM — v8 intrabar steadiness (edge or blind luck?)

## CK observation
SS360's line FLICKERS intrabar: turns red + fires a signal, then vanishes and resolves to the
opposite color / orange while the candle is still forming (its own doc: last bar can still change).
v8 instead HOLDS its state steadily through bar formation and continues. CK reads v8's steadiness as
potentially BETTER for NQ.

## Two indistinguishable causes (only one is a real edge) -- MUST disambiguate by price
1. EDGE (good): v8 correctly judged the intrabar wobble was NOISE and held -> filtered a head-fake
   SS360 flashed then retracted. Repeatable advantage.
2. BLIND (lucky): v8 held only because it computes on CONFIRMED CLOSE and CANNOT SEE the intrabar
   move. SS360 flickered because it DID see a real intrabar probe; v8 got lucky the probe reversed.
   On another bar, same blindness = v8 MISSES a real intrabar commit SS360 caught.
From outside both look identical ("SS360 flickered, v8 held, v8 right").

## The tell (CK to tag live):
When SS360 flickers-and-vanishes AND v8 holds -> did PRICE follow v8's held direction (VINDICATED),
or did v8 just dodge a bullet it couldn't see (LUCKY-BLIND, i.e. SS360's flicker was catching
something real that just happened to fail this time)? Tag each occurrence VINDICATED / LUCKY.
A handful of tagged cases tells us which regime we're in.
- If mostly VINDICATED -> real NQ filtering edge; keep/lean on v8's confirmed-bar steadiness.
- If mixed/LUCKY -> v8 is just lower-resolution; steadiness is not an edge, and 5m/intrabar matters.

## Status: CANDIDATE, not confirmed. Track alongside the early-exit live-observation item. Post-v9.

---

# ADDENDUM — 2026-06-25: DAY-REGIME MAP for incoming 6/24 session data (CK summary)
Session runs 6/24 18:00 (6PM open) -> 6/25 RTH close. CK one-line regime map w/ timestamps:
  - 6/24 6PM  -> 6/25 8AM   : BULL trend
  - 6/25 8AM  -> 11AM        : BEAR trend
  - 6/25 11AM -> 2PM         : CHOPPY
  - 6/25 after 2PM           : DOWN trend
Use this to label each stretch trend/chop when scoring the incoming CSV+ledger.

## How to apply when data arrives:
- BULL (6PM-8AM) & BEAR (8-11AM) & DOWN (post-2PM) = TREND stretches -> here is where to test the
  counter-trend filter condition: any N-flip AGAINST the prevailing direction = the "ctr" case to
  teach "fast flip + against run = skip." Also where MU-type early-lead / grade-escalation shows.
- CHOP (11AM-2PM) = expect intrabar thrash; v8 mismatch here is the known data-granularity limit,
  largely untradeable. Don't over-weight chop mismatches.
- CK will add inline notes alongside R/N flags in the ledger (bonus context).

## Coverage I currently hold (so CK only uploads NEW, no gaps):
  5m : CSV thru 6/24 18:10 ; ledger thru 6/24 ~4:45PM
  15m: CSV thru 6/24 14:00 ; ledger thru 6/24 1:45PM
  1h : CSV thru 6/24 12:00 ; ledger thru 6/24 11:00AM
=> NEW data = each TF from those points forward (or just restart at 6/24 6PM open; small overlap ok).

---

# CORRECTION/REFINEMENT — 2026-06-25: regime maps are PER-TIMEFRAME (the earlier map was 1h)

## The earlier 6/24 regime map was the 1h (TIDE) read. The 15m (WAVE) read differs:
1h (TIDE):  6PM-8AM bull | 8-11AM bear | 11AM-2PM chop | post-2PM down
15m (WAVE): 6-8 bull | 8-10:30 bear | 10:30-4AM bull | 4-9:15AM chop | 9:15-11 bear | 11-2:30 bull | end bear
(times ET; 15m shows MORE shifts than 1h, as expected: faster TF = more, shorter regimes.)

## WHY this matters (conceptual, not just bookkeeping)
This is the TIDE->WAVE->RIPL structure made concrete: the SAME session reads as fewer/longer regimes
on 1h and more/shorter regimes on 15m. So "trend vs chop" labels MUST be tied to a timeframe -- a
stretch that is one clean 1h trend contains several 15m sub-swings. When scoring v8 + the
counter-trend filter, use the map for the MATCHING timeframe (score 15m CSV against the 15m map,
1h CSV against the 1h map). Mixing them would mislabel trend/chop.

## 5m (RIPL): CK judges a 5m day-summary low-value -- AGREED
5m would be many short, shifting regimes (even more fragmented than 15m), so a coarse day-summary
adds little. For 5m the useful unit is NOT a day-regime label but the PER-FLIP R/N tag (+ optional
"ctr" when a flip opposes the local move). So: 5m -> rely on R/N tags, skip the day-summary.
15m & 1h -> day-regime maps are useful (done above). This matches their roles: TIDE/WAVE = regime
context; RIPL = entry timing where individual flips (R/N) are the right granularity.

---

# ADDENDUM — 2026-06-25: 1h data confirms STATE-GRADE fix (and the 1h bug live)

## The 1h CSV came in with v8 rating column = ALL ZEROS (the stuck-orange/compute bug, live).
Could not score v8 directly; instead tested the STATE-GRADE prototype vs SS360's 1h ladder.

## Result: current ns-grade vs STATE-grade on 6/24-25 1h (29 bars)
- CURRENT (ns-based): pinned ±3 the WHOLE session. Through the bullish overnight run (SS360 = +5),
  v8 never exceeded +3 because 1h ns stays < 0.40 band. = the "1h fails" report, quantified.
- STATE-GRADE prototype (snap to +5/-7 once a direction holds; +3 = 1-bar transition; ->7 if ns big):
  overnight bull = +5 (matches SS360 +5); 9AM crash = -7 (matches SS360 -7 + S/BC); 6/24 PM selloff
  = -7 (matches red). Tracks SS360's actual ladder where ns-grade could not.
- Preserved MU constraint: 16:00-19:00 stepped +3 -> +5 as move built (escalation kept). GOOD.

## Caveat (first-cut state logic): single counter-bars inside a trend dipped it to ±3 (07:00, 13:00)
= the counter-trend flicker. STATE-grade fixes the GRADE CEILING; still needs the COUNTER-TREND
condition layered so a 1-bar mid-trend wiggle doesn't break the held state. Both already on v9 list;
this confirms they work TOGETHER (grade-state + counter-trend = the 1h fix).

## Also fix in v9: the 1h compute/zero bug (rating logged 0s) — ensure engine computes on 1h
(likely the deadband/session-reset starving 1h to 0; the session_gap or ATR warmup on 1h). Verify
v9 emits non-zero ratings on 1h.

## SS360 1h ladder (6/24-25) recorded for reference:
2PM red->orange | 4PM orange->green | 6PM +5 B/SC | 7PM->orange | 8PM green | 9PM +5->orange |
10PM green | 11PM +5 | 3AM orange | 5AM green | 6AM +5->orange | 7AM green | 8AM +5->red |
9AM -7 S/BC | 11AM red->orange | 2PM orange->red | 3PM -7.

---

# ADDENDUM — 2026-06-25: 15m data (BLUE/+7 + R/N + N+CTR tag) — MAJOR grade reframe

## Scores low for BOTH (32 bars): v8-current exact 22% / sign 38%; state-grade exact 28%.
Neither close. The spot-check shows WHY, and it overturns a grade assumption:

## ★★★ +7 (BLUE) is about PERSISTENCE, not SPEED
SS360 printed +7 through almost the ENTIRE overnight bull grind (22:30 -> 6AM, repeated +7) -- but
that was a SLOW, low-velocity grind: ns sat 0.07-0.27 the whole time. Yet SS360 = max +7.
Our state-grade (snap to extreme) only reaches +5; current v8 stuck +3. 
=> +7 is NOT reserved for fast/high-ns moves (we'd assumed it was, from the MU ns~1.4 spike). +7 is
   EARNED BY DURATION: a state that has PERSISTED many bars in one direction reaches +7 regardless
   of per-bar speed. Grade has a PERSISTENCE axis we never modeled.
This reframes grade: GRADE ~= CONVICTION = (how LONG + how CLEANLY the direction has persisted),
NOT how fast the line moves in a single bar. Explains everything: chop never reaches high grades (no
state persists); trends do (state holds). 
NEW grade model for v9: escalate the grade the LONGER a direction holds uninterrupted:
  fresh commit -> 3 ; holds a few bars -> 5 ; holds many bars / clean -> 7. Plus a FAST override
  (big ns can jump to 5/7 immediately, e.g. MU/crash). So grade = max(persistence-ramp, ns-jump).

## ★ 12:15 N+CTR tag CONFIRMED — counter-trend + v8 steadiness VINDICATED
SS360 flipped -7 at 12:15 (CK tagged N+CTR), reverted to orange 12:30 — a counter-poke INSIDE the
11-2:30 bull. v8 IGNORED it (held +3/+5). So here v8's steadiness = the VINDICATED kind (not blind
luck). One data point: intrabar-steadiness can be a real edge. AND confirms the counter-trend filter
target: a fast flip against the prevailing run = skip (exactly what CK tagged).

## ★ 9:30-9:45 crash — v8 OVER-HELD -7 (the sticky side of steadiness)
-400pt drop: both hit -7 at 9:30 (ns 2.0, correct). At 9:45 SS360 already back to ORANGE; v8 STILL
-7. SS360 RELEASES -7 fast once velocity spends; v8 clings. v9 needs a faster RELEASE when ns
collapses after a spike (deceleration -> exit the extreme), not just a faster entry.

## v9 grade design (updated):
1. grade = max(PERSISTENCE-ramp 3->5->7 by bars-held, ns-JUMP for fast moves). [+7 = persistence!]
2. faster RELEASE from extreme when ns collapses post-spike (9:45 case).
3. counter-trend condition: ignore fast flip against the established run (12:15 N+CTR case).
4. preserve MU 3->5 escalation; fix 1h zero-compute bug.

---

# ADDENDUM — 2026-06-25: 9:15 crash — v8's longer HOLD won here (but it's a TRADE-OFF, not a strict win)
CK observation (correct on facts): on the -400 drop, SS360 cut to NEUTRAL early (9:45) but price kept
falling 200+ MORE points after. v8 gave pink ~9:00, red at 9:15 (30151), printed -7, held longer.
Entering at the signal and exiting on neutral, v8's later release captured the extra 200 pts SS360 left.
=> On THIS move, v8's stickiness was BETTER (this morning I'd flagged the same stickiness as a flaw at
   9:45 — opposite verdict, same trait).

## ★ The honest framing: v8 HOLDS STATES LONGER than SS360 = a TRADE-OFF, not a strict improvement.
- WINS on CONTINUATION moves (today's -400 kept going -> hold paid +200).
- LOSES on sharp REVERSALS (hold to -7, then V-bounce -> gives back what SS360's early release protected).
Same mechanism, opposite outcome; depends on continue-vs-reverse, unknown at the hold moment.
NOT "v8 release is a flaw" (AM framing) NOR "v8 hold is the edge" (PM framing) — it's a tunable
trade-off. The real question = ACROSS MANY MOVES on NQ, does the longer hold win more than it loses?
Today = won. n=1. Need the count.

## MEASUREMENT PLAN (turns "felt better" into points): per signal, log
  entry price | SS360-neutral exit price | v8-neutral exit price -> points delta.
Over ~2 weeks / ~30 signals this quantifies whether v8's longer hold is net-positive on NQ in POINTS.
That number is what would justify trusting v8's hold over SS360's release (or tuning the release).

## Tie-in: this is the SAME trade-off as the intrabar-steadiness item (hold = edge on continuation,
liability on reversal). Track together. v9 release tuning (9:45 fast-release) should be made
ADJUSTABLE, not hard-coded, until the win/loss count says which way to lean on NQ.

---

# CORRECTION — 2026-06-25: v8 ALSO flashes intrabar (my "v8 is close-only/blind" assumption was WRONG)
CK observed (last night + today, esp. 5m): v8 itself FLASHES intrabar rating changes that print only
if they hold, and VANISH if they don't — same live behavior as SS360.

## Mechanics (why I was wrong):
TradingView recalcs the indicator on EVERY TICK of the forming bar, so v8's rating DOES update live
intrabar, then settles at close. My barstate.isconfirmed gate only controls CSV LOGGING — it does NOT
stop the on-screen color/number from flickering. So v8 has flashed all along; I only ever saw the
confirmed CLOSES in CK's CSVs and wrongly inferred the engine was close-only/intrabar-blind.

## What this overturns / clarifies:
- The "v8 intrabar steadiness = filtering edge OR blind luck?" item: it was NEVER "blind." v8 SEES the
  forming-bar action and flickers too. When v8 "held" where SS360 flickered, it's because v8's
  persistence/deadband SETTLED to the same place by close — a tunable behavior, not a resolution edge.
  Reframe the question to: whose SETTLE-logic is better (not who sees more).
- Scoring is NOT invalidated: CSV logs CONFIRMED closes; we compare confirmed bars vs SS360's confirmed
  read. Apples-to-apples holds.
- BUT: v8's LIVE on-screen flicker has never been measured (CSV only has closes). If CK reads signals
  live off the screen, there is flicker the CSV never recorded -> a gap between "what flickers live" and
  "what gets logged." Worth tracking; live discretionary reads see more noise than the CSV implies.

## v9 implication:
The intrabar flicker is inherent to live TV recalc; can't fully remove. Can REDUCE on-screen flicker
with confirm-persistence on the DISPLAY (e.g. only repaint the line/number after N ticks or on close)
if CK wants a calmer live read. Make it a toggle. Does not change the logged/confirmed logic.

---

# ADDENDUM — 2026-06-25: two more v9 DISPLAY items (CK requests)

## 1. Display-only persistence toggle (APPROVED — add to v9)
On-screen flicker reduction: repaint the line/number/marker only AFTER a change holds N ticks or on
bar close. Toggle (default on for calmer live 5m read). Does NOT touch confirmed logic or CSV logging.

## 2. BUG: B/SC · S/BC labels print far from the SS line — FIX in v9
Cause: in the v8 display rewrite I anchored signal labels to bar low (B/SC) / high (S/BC). On a tall
candle the label flies far from the colored SS line (where the eye is). SS360 prints markers ON the
line.
FIX: anchor signal labels to the SS VALUE at the bar (like the rating number), small offset:
  - B/SC -> just BELOW the SS line (where a long is read)
  - S/BC -> just ABOVE the SS line (where a short is read)
  - offset so they don't collide with the rating number (which prints above the line).
This is a placement bug, not cosmetics — markers must sit next to the line to be readable live.

## Running v9 build list (consolidated):
A. grade = max(PERSISTENCE-ramp 3->5->7 by bars-held, ns-JUMP for fast moves). [+7 = persistence]
B. faster RELEASE from extreme when ns collapses post-spike — make ADJUSTABLE (hold-vs-release is a
   trade-off; let CK's live points-count decide).
C. counter-trend skip: ignore fast flip against the established run (12:15 N+CTR case).
D. preserve MU 3->5 escalation.
E. fix 1h zero-compute bug (engine must emit non-zero ratings on 1h).
F. display-only persistence toggle (this item 1).
G. B/SC below line / S/BC above line, anchored to SS (this item 2).
H. keep: dark palette, no-0-print, price-at-signal toggle, combined markers, single-TF.

---

# ADDENDUM — 2026-06-25: 5m real/noise filter CONFIRMED OUT-OF-SAMPLE + reduce ledger effort

## ★ The ns filter GENERALIZES (cross-day confirmation — was the missing piece)
On the NEW 6/24-25 5m flips (CK's R/N tags): REAL median ns 0.69, NOISE median ns 0.14 — same clean
separation as 6/24. Simple ns>=0.40 gate: kept 100% REAL (11/11), killed 90% NOISE (9/10).
=> filter is REAL, not overfit to 6/24. Two-day hold. This was the cross-day check we needed.

## REDUCE 5m LEDGER EFFORT (CK: "collecting 5m ledger is real pain, ~135 lines")
The 5m ledger's two jobs are DONE: (1) build filter [6/24], (2) confirm it generalizes [6/25]. Full
135-line 5m ledgers now = diminishing returns. NEW capture plan:
- 5m CSV: keep (export, no manual work) — gives bars/ns.
- 5m ledger: ONLY the handful of flips CK actually TRADED or seriously considered, with R/N (~10
  lines, not 135). Enough to keep checking "do real trades have high ns?".
- 15m + 1h ledgers: keep full (short, high-value).
Cuts ~90% of 5m transcription pain while preserving the signal.

## Note: CK's 6PM ledger comment "should be -7/-5/-3" = the GRADE issue again (v8 orange/-3 where it
should be deeper bear). v9 persistence-grade targets exactly this; now have this session to test on.

## ns filter v1 (confirmed): TRADE if ns>=~0.40-0.55 (real), skip ns<~0.15 (noise); mid-band needs
the counter-trend condition (item C). Two days now support the ns separation.

---

# ADDENDUM — 2026-06-26: DEEP DIVE — the real v8 bug is the DIRECTION SOURCE (not grade)

## Diagnosis (measured across full 6/25 5m session, bucketed by regime):
v8 direction (1-bar SS slope) vs SS360: GRIND 3/9, CRASH 2/2, SWING 10/18, overall 62%.
- CRASH: perfect (real velocity -> 1-bar slope fine).
- SLOW GRIND: 3/9 = the core failure. SS360 holds +1 through the overnight grind; v8 flips to -1 on
  single-bar line wiggles (ns 0.05-0.30). The line grinds UP over many bars but wiggles bar-to-bar;
  v8 reads direction off the 1-bar wiggle. SS360 reads the PERSISTENT slope.
- DAY-SWING: 10/18 ~ coin flip = the known INTRABAR problem (5m closes can't see SS360's mid-bar
  flip). Largely untradeable chop; not fixable from close data.

## Root cause: layering persistence ON TOP of a NOISY 1-bar direction can't work. Fix the SOURCE.
(This is why last night's patch-stream kept moving the bug — wrong layer.)

## ★ FIX (validated across regimes, not curve-fit): DIRECTION = 3-BAR LINE SLOPE
Tested 6 direction definitions vs SS360:
  1bar SS slope (v8 now): 62%  (grind 3/9)
  3bar SS slope:          72%  (grind 6/9, crash 2/2, swing 13/18)   <-- WINNER
  5bar SS slope:          58%  (too slow, hurts swings)
  price vs SS line:       55%
  SS vs EMA(SS):          68%  (grind 7/9 best, swings worse)
  3bar slope + hold:      72%
=> 3-BAR slope fixes the GRIND without breaking CRASH or SWINGS = uniform improvement, not a
   one-spot fit. Direction reads the line's PERSISTENT lean, riding through 1-bar wiggles.
   Headline 72% includes unfixable intrabar swing bars; tradeable-regime accuracy is higher.

## v9 CHANGE (principled, single): set am_lb = 3 (direction = SS - SS[3]) as the DIRECTION basis.
Keep ns (for grade/jump) on the FAST 1-bar speed (velocity magnitude is correctly fast). I.e.
DIRECTION from 3-bar slope; GRADE/ns-jump from 1-bar speed magnitude. They are different axes.
Then re-validate the full chain (persistence grade + ns-jump + release + counter-trend) on top of
the stable 3-bar direction, across ALL sessions, before shipping. NO patch-by-patch.

## Status: diagnosis + fix VALIDATED on 6/25 5m. Next: apply to v9, re-run across 1h/15m/5m, cold-check.

---

# CORRECTION — 2026-06-26 (AM): two ground-truth errors CK caught before the v9 build

## 1. NO CAP on ratings. +7/-7 are real, just rarer.
- 1h DID hit +7 (on 6/22, not in the 6/25 sample). I wrongly inferred 1h "caps at +5" from one
  session. Same mistake pattern as the earlier "no -7" assumption that CK's data later disproved.
- DISCIPLINE: never infer a structural CAP from a rating's absence in one sample. If the engine
  lists a rating, it reaches it. Don't lock to unproven setups; reason across scenarios.

## 2. ★ "+7 = persistence (HELD)" was WRONG — it FLICKERS top↔orange. Reshapes Layer 3.
CK: during the 11PM-3AM grind it FLICKERED between top color and orange on ALL timeframes:
  - 1h flickered between +5 and orange
  - 15m and 5m flickered between +7 and orange
=> It did NOT sit locked at the top grade. The top grade is REACHED on each clean re-push, then
   RELAXES to orange between pushes. So:
   - OLD (wrong): grade = duration ramp; hold long -> climb to +7 and STAY.
   - NEW (correct): each clean RE-PUSH in a sustained direction re-prints the top grade, then
     decays toward orange between pushes. The "grind to +7" is really a FLICKER top<->orange,
     not a held plateau.
   This is much closer to the early "fast line slope re-steepens per leg -> re-prints extreme,
   flat between -> orange" finding than to a persistence plateau.

## 3. ★ TIMEFRAME scales GRADE: same grind, same clock -> 1h tops +5, 15m/5m top +7.
The faster TF reads the SAME move as a STRONGER grade. Grade magnitude is TF-dependent (faster TF =
higher peak grade on the same underlying move). Layer 3 grade must account for this, not use one
fixed band set across TFs.

## Impact on the v9 BLUEPRINT (SS_v9_ARCHITECTURE.md):
- Layer 3 (GRADE) redesign: NOT a duration ramp-and-hold. Instead: grade peaks on each clean
  re-push (re-acceleration of the fast line in the held direction) and decays to orange between.
  This unifies with Layer 1's fast detector — grade is driven by the SAME fast re-pushes, scaled
  by TF. Re-examine whether grade is even a separate "slow" component or just the MAGNITUDE of the
  fast detector's re-pushes. (May simplify the 2-component model.)
- Layer 1 (DIRECTION) unaffected: still the fast flip detector. But the grind is FLICKER not hold,
  so Layer 1 testing must expect top<->orange oscillation in grinds, not a steady sign.
- Ladder fix: do NOT fill grind stretches as "held +7". Mark them as flicker top<->orange.

---

# ★ CK RULINGS on v8-vs-SS360 disagreements — 2026-06-26 (all 5m, 6/25)
Reframed goal (CK): do NOT match SS360 where SS360 is WRONG. If our engine rates a weak
slow-grind push LOW and SS360 prints HIGH, WE are right (better, more honest read). CK judges
genuine-vs-mistake from live data.

## RULINGS (ground-truth confirmed by CK):
- 6/25 03:00 5m: 3-4AM ~100pt move but SLOW GRIND w/ multiple variations. v8 rated 3->5->3->-3->3
  (honest, tracked the chop). SS360 printed +7 repeatedly (3:00/3:40/3:55, flicker blue<->orange).
  CK RULING: ★ WE DID BETTER. SS360's +7 would make a trader bet high-confidence and GET HURT —
  no clean momentum to justify +7. Our variable lower grade = correct. CONFIRMS ns-magnitude grade.
- 6/25 12:00 5m: SS360 +5 on dead bar (ns 0.06). CK RULING: OVER-GRADE by SS360. Our 0 = correct.
- 6/25 grind cluster 00:30/02:00/02:25/06:00: CK RULING: SS360 was FLICKERING (orange in between
  colors several times), NOT holding bullish. => our "disagreements" here are NON-ISSUES; both
  flicker. No Layer-1 direction fix needed for these.
- 6/25 07:00 5m: CK saw PINK on v8. Verified: v8 went -3/PINK at 07:10 (price rolled 30187->30116);
  at 07:00 it was a 1-bar transitional orange before committing. v8 & SS360 BOTH bearish at 07:10.
  => NOT a real disagreement (my table caught the transitional bar). Comes off the list.

## NET: none of the flagged grind disagreements are v8 direction FAILURES. They are either
## (a) us correctly under-grading SS360's over-grades, or (b) both-flicker non-issues. 
## => Layer 1 (fast direction) does NOT need a special grind-trend fix; the grind is FLICKER and
##    v8 already flickers with it. The earlier "grind 3/9" panic was scoring against a WRONG
##    ground truth (I had marked the grind as HELD bull; it was flickering). 

## DESIGN CONFIRMED: grade = magnitude of the fast push (ns-scaled). Low push -> low grade even in
## a long move = FEATURE not bug (CK: "if we have solid technical reasoning for our low rating,
## we did better"). Do NOT chase SS360's inflated grind grades.

---

# ★★ LAYER 1 BUILT & PASSED — 2026-06-26 (scored vs CORRECTED flicker-aware ground truth)

## Bake-off (5m 6/25, clean regimes = held+cascade):
  1bar SS slope (v8 current):  84%
  3bar SS slope:               80%   (last night's "fix" — actually WORSE here)
  EMA3 slope:                  84%
  1bar SS slope + NEUTRAL BAND: 92%  <== WINNER
  cascade (crash) 2/2 perfect for all.

## ★ THE REAL FIX = a NEUTRAL BAND on the 1-bar slope, NOT a slower lookback.
v8's 1-bar slope was ~right all along (84% vs correct truth). Adding a small ATR deadband
("if line barely moved, call it neutral, don't pick a side") -> 92%. Slower lookbacks (2/3-bar)
scored WORSE. This INVERTS last night's 3-bar-slope conclusion, which was scored against a
CONTAMINATED (held-not-flicker) ladder. Direction never needed a rebuild — just a deadband.

## v8 was "fair strength" exactly as CK said. The "62% / grind 3/9" panic was a ground-truth
## artifact. Real v8 direction = 84% clean; +band = 92%.

## BAND SENSITIVITY (overfit check): monotonic PLATEAU.
  band x ATR:  0.08-0.15 -> 92% (stable plateau); 0.20-0.25 -> 96%; 0.30 -> 100%.
  - Plateau 0.08-0.15 = ROBUST (insensitive to exact value). TRUST THIS.
  - 100% at 0.30 is at the EDGE of tested range + monotonic-to-edge = possible single-session
    overfit. Do NOT bank on it. VERIFY wider-is-better on a 2nd day before widening.
  DECISION: lock band ~0.12 x ATR (middle of stable plateau, 92%) for now. Revisit with day 2.

## STATUS: Layer 1 = sign(SS[i]-SS[i-1]) vs band(0.12*ATR). PASSED gate (>=80% held+cascade,
## cascade 2/2). Ready for Layer 2 (price-vs-line neutralize) — must not DROP this.
## CAVEAT: one session. Band value + the whole layer need a 2nd-day confirm before lock.

---

# ★★★ LAYER 1 CROSS-DAY VALIDATED — 2026-06-26 (3 independent days: 6/23, 6/24, 6/25)
CK supplied 6/23 + 6/24 5m SS360 ladders. Normalized all 3 file formats into the harness.

## RESULT: direction = 1-bar SS slope matches SS360's committed sign 95-100% PER DAY.
  band 0.00: 6/23 97% | 6/24 100% | 6/25 100% | COMBINED 106/107 = 99%
  band 0.08: 6/23 95% | 6/24 100% | 6/25 100% | COMBINED 98%
  band 0.12: 6/23 95% | 6/24  89% | 6/25 100% | COMBINED 95%
  band 0.30: 6/23 86% | 6/24  85% | 6/25  94% | COMBINED 88%
(Scored on bars where SS360 COMMITTED a sign; orange bars skipped — that's a Layer-2 question.)

## ★ THE BAND CONCLUSION FLIPPED once a 2nd/3rd day was added:
- On 6/25 ALONE: wider band looked better, up to 100% @0.30. (Looked great, was OVERFIT.)
- Across 3 DAYS: wider band is WORSE. Combined PEAKS at band 0.00-0.08, DECLINES to 88% @0.30.
  6/24 specifically gets HURT by a wide band (100%->85%). 
=> The "wider is better" was 6/25-specific overfit, KILLED by one more day. Exactly the trap I
   flagged. The cross-day discipline did its job.
=> LOCK band SMALL: 0.05-0.08 x ATR (tiny deadband to drop dead-flat bars; not wide). NOT 0.12+.

## Layer 1 direction is EXCELLENT and ROBUST across 3 days (99% on directional bars). v8's
## 1-bar slope was the right detector all along; needs only a TINY deadband, not a rebuild and
## not a wide band. This is the strongest validated result of the project.

## HONEST SCOPE: "99% direction" = when SS360 picks a side, we pick the same side. Does NOT yet
## test whether we go ORANGE when SS360 does (the neutralize) — that's Layer 2. Not a hidden win.

## Layer-1 misses to revisit (few): 6/23 13:05,16:00; 6/24 11:45,12:00,12:15 (all flicker-window
## bars, likely intrabar). Check during Layer 2.

## STATUS: Layer 1 = sign(SS[i]-SS[i-1]) with band ~0.06*ATR. CROSS-DAY PASSED (99%/3 days).
## Ready for Layer 2 (price-vs-line neutralize) — gate: must not drop this direction score AND
## must add correct orange behavior.

---

# LAYER 2 (neutralize) — PARTIAL pass, hits a STRUCTURAL frontier — 2026-06-26 (3 days)
Goal: add correct ORANGE behavior without losing Layer 1's 99% direction.

## Result: orange and direction TRADE OFF monotonically; no setting gives both high.
  L1 only:                direction 99%, orange 26%   (L1 never goes orange)
  L2 strict price-counter: direction 89%, orange 82%   <- best orange, big gain
  L2 counter+margin 0.5:   direction 99%, orange 51%
  Variant A (line-flat/ns): worse (orange caps 68% @ dir 88%)
  Variant B (flat OR counter): dir 90% / orange 73% (best 'sum' 163)
Tried 3 mechanisms (price-counter, line-flatness, combo). ALL land on the same frontier
~ (dir + orange) ≈ 160-171. The tradeoff is STRUCTURAL, not a tuning artifact.

## Interpretation: SS360 orange is NOT cleanly predictable from price-vs-line, line-flatness,
## or ns (alone or combined) on 5m CLOSES. Candidates for the missing factor:
##  1. INTRABAR — orange set on within-bar conditions 5m closes can't see (the flicker wall).
##  2. Orange = TRANSITION/RESET state (between pushes), a timing rule not a level rule. None of
##     my variants model "just flipped -> reset to orange".
##  3. Some scored orange bars are mid-FLICKER (momentary), arguably not firm targets.

## Direction cost is concentrated in FLICKER bars (6/24 11:45/12:00/12:15 etc). On CLEAN bars
## L2 likely holds direction far better than the blended 89%.

## DECISION (pending CK): accept STRICT price-counter as Layer 2 v1 (best validated mechanism;
## orange 26->82%; cost mostly flicker), LOG the orange ceiling + 3 hypotheses, move to LAYER 3.
## RATIONALE: Layer 3 (grade=push magnitude) may EXPLAIN orange naturally — orange could just be
## "grade rounds to 0", i.e. we may be solving orange in the WRONG layer. Test that in L3 before
## over-engineering L2.

## Contrast w/ Layer 1: L1 = clean 99% cross-day pass. L2 = honest PARTIAL (frontier), not a
## clean pass. Not calling it done.

---

# ★★★ LAYER 3 — CROSS-DAY OVERTURNS THE MORNING'S "grade = push magnitude" CONCLUSION
# 2026-06-26. This is the key correction of the day.

## The decisive numbers (median OUR ns at each SS360 grade level, 3 days, n=58):
  SS360 grade 3 -> our median ns 0.62
  SS360 grade 5 -> our median ns 0.48
  SS360 grade 7 -> our median ns 0.41
=> BACKWARDS. SS360's HIGHEST grade (+7) is where OUR push (ns) is LOWEST. If grade were push
   magnitude these would ASCEND (7=highest ns). They DESCEND. Grade is NOT ns magnitude.
   Exact grade match only 27% across 3 days.

## => THE MORNING'S 8.6x "grade tracks ns" TEST WAS A SINGLE-SESSION (6/25) ARTIFACT.
   It did NOT generalize. Same failure pattern as the band-overfit: decisive on 6/25, dead across
   3 days. The single-component "grade = magnitude of fast detector" architecture is WRONG.
   Cross-day discipline caught it AGAIN. (Lesson reinforced: NEVER lock architecture on 1 session.)

## RECONCILED ARCHITECTURE (two components, correctly this time — they don't fight):
  - DIRECTION = FAST 1-bar slope, flickers (resets to orange between pushes). 99% cross-day. SOLID.
  - GRADE = SLOW / PERSISTENCE. +7 grind peak occurs at LOW per-bar ns => grade builds with the
    DURATION the move has persisted, NOT per-bar velocity. The grind flickers (low ns per bar) but
    the grade each re-push REACHES climbs with how long the overall move has lasted.
  They measure DIFFERENT things (fast=when to flip; slow=how strong by duration). This morning I
  WRONGLY collapsed them because I misread "grade tracks ns". Cross-day says grade does NOT track
  ns -> two components are back, correctly.

## ALSO CONFIRMED (Layer 3 Test 1 + bug): SS360 commits a DIRECTION even on tiny ns. So orange is
   NOT "small push / grade-zero". Tried to gate direction by grade -> direction collapsed 99->53%
   (violated the one-way rule: grade must NEVER override direction). Orange is its own thing,
   still unexplained by price-counter, line-flatness, ns, OR grade-zero. Leading hypothesis now:
   orange = TRANSITION/RESET state between pushes (a timing/sequence rule), + intrabar.

## NEXT TEST (falsifiable, not yet built): does SS360 grade track DURATION (bars since last real
   reversal / length of persistent run) rather than ns? If grade rises with persistence length
   across all 3 days -> Layer 3 = persistence ramp (build it). If not -> grade source still open.

## STATUS: Layer 1 SOLID (99%/3day). Layer 2 partial (orange frontier). Layer 3 grade source
   RE-OPENED: NOT ns. Test persistence/duration next. Do NOT build grade until the source holds
   across 3 days.

---

# ★ LAYER 3 RESOLVED — by REFRAMING (CK: "our own honest grade is fine, even if different")
# 2026-06-26

## The unlock: I was solving the WRONG problem.
I spent an hour trying to REVERSE-ENGINEER SS360's grade and failed (no close-data feature
correlates: ns +0.04, duration +0.09, move +0.19, cumulative +0.20 — all ~noise across 3 days).
But CK confirmed (twice) the GOAL is NOT to copy SS360's grade — it's a SOUND grade of OUR OWN.
SS360 over-grades weak grinds (+7 on the choppy 3-4AM move that would hurt a trader, CK-ruled).
=> I don't need to predict SS360's number. I need an honest push-strength grade. We HAVE that.

## LAYER 3 DECISION: grade = magnitude from ns (push strength), 3/5/7 by ns thresholds.
   - It is our OWN honest grade. ~27% "match" to SS360 is mostly US being more honest (feature).
   - One-way rule: grade sets MAGNITUDE only, never overrides direction.
   - Committed direction => grade at least 3; bigger ns => 5/7.
   - Do NOT chase SS360's grade-source mystery (likely intrabar and/or flicker; not wanted anyway).

## What SS360's grade actually is (for the record, NOT to replicate):
   Resists every close-data explanation across 3 days. Leading theory: partly INTRABAR (computed
   from within-bar action 5m closes can't see) and/or FLICKER (grind prints 3/5/7/orange irregular).
   CK has NEVER observed grade during candle FORMATION — only settled values; will check recordings
   to see if grade flickers intrabar like colors. If yes -> confirms grade is intrabar-ish ->
   confirms it's not close-reproducible and not worth chasing. (Pending CK recording check.)

## CROSS-DAY LESSON (reinforced today, twice): morning's "grade=ns, 8.6x" AND the band-width
   optimum were BOTH single-session (6/25) artifacts that died across 3 days. NEVER lock a
   finding on one session. The discipline caught both.

## ENGINE STATUS (all 3-day validated unless noted):
   LAYER 1 direction = 1-bar SS slope + tiny band (~0.06 ATR): 99% cross-day. SOLID.
   LAYER 2 neutralize = price-vs-line strict counter: orange 26->82%, dir cost mostly flicker.
           Partial (orange frontier ~82% ceiling; rest likely intrabar). ACCEPT as v1.
   LAYER 3 grade = ns magnitude (3/5/7), OUR honest grade (not SS360's). DECIDED.
   LAYER 4 release = still to build (fast exit from extreme when ns collapses; adjustable).

---

# 1h + 15m MULTI-DAY COMPARISON — 2026-06-26 (CK supplied 15m back to 6/21 + full 1h ladder)

## Comparison results (±1 bar tolerance), all 3 timeframes now tested:
  5m  (6/23-25): MATCH 62% + TIMING 25% = 87% same call; 18 REAL_DIFFER / 148 bars
  15m (6/24-25): MATCH 64% + TIMING 25% = 89% same call;  5 REAL_DIFFER / 48 bars
  1h  (6/22-25): MATCH 53% + TIMING 18% = 71% same call;  8 REAL_DIFFER / 28 bars
=> 5m & 15m STRONG and consistent (engine generalizes across these TFs). 1h is WEAKER.

## 1h diagnosis — I made TWO WRONG GUESSES, data rejected both (logging honestly):
  - WRONG guess 1: band must scale down on 1h. TEST: band 0.00-0.10 all give 71% (FLAT). Not it.
  - WRONG guess 2: Layer-2 price-counter over-fires on the lagging 1h line. TEST: counter ON vs
    OFF both 71%. Not it (even though 3 of 8 misses ARE counter-flagged, removing it didn't help
    net — it traded misses for other misses).
  => No single parameter fixes the 1h. Stopped guessing (avoided the reactive patch-hunt trap).

## What the 1h misses ACTUALLY are (8 REAL_DIFFER):
  - 6 of 8: SS360 committed a DIRECTION on very slow LOW-ns 1h drift (ns 0.01-0.19); we stayed
    orange. By CK's OWN principle (5m 03:00/12:00 = SS360 over-grades weak pushes), these low-ns
    1h commits are LIKELY SS360 OVER-COMMITTING on weak drift -> our quieter orange may be the
    MORE HONEST read. NEEDS CK live-1h check.
  - 2 of 8: high-ns (0.56, 1.43) where WE committed and SS360 stayed orange (6/22 14:00, 6/23
    04:00) — possible our edge or 1h over-sensitivity.

## OPEN QUESTION (only CK can answer from live 1h): is SS360's 1h itself noisier/over-committed,
## and is our quieter 1h actually BETTER (like 03:00 on 5m)? Or are we genuinely missing slow 1h
## trends? The 8 bars are in the comparison CSV, tagged, for live review.

## Comparison CSV now has ALL 3 TIMEFRAMES (5m+15m+1h), every ladder bar tagged
## MATCH/TIMING/REAL_DIFFER with ns, OUR color, SS360 event. Total genuine differs to study
## across all TFs/days: ~31. File: SS_v9_vs_SS360_comparison.csv

## STATUS: 5m/15m engine validated & consistent (~88%). 1h weaker (71%), cause is NOT one
## parameter; likely a mix of SS360-over-commit (our edge) + genuine slow-TF slope marginality.
## Defer 1h tuning until more 1h days + CK's live ruling on whether the 6 low-ns commits are
## SS360 mistakes or our misses.

---

# 15m FULL MULTI-DAY (6/21-24) — 2026-06-26: 87% holds across 4 days
CK supplied full v8-logged 15m data 6/21->6/24 (211 bars, SS/ns/rating cols).
RESULT (88 ladder bars, ±1 tol): MATCH 67% + TIMING 20% = 87% same call; 11 REAL_DIFFER.
=> 87% now confirmed across 4 DAYS on 15m (was 89% on 1 day). Matches 5m's 87%. Both TFs
   STABLE multi-day, no longer single-session. Strong validation.

## The 11 REAL_DIFFER split into the SAME TWO consistent patterns seen on every TF:
  P1 (we commit on strong push, SS360 orange): 6/22 00:30 (ns.77), 6/22 10:00 (.77),
     6/23 10:15 (ns 1.22, we GREEN — big push left orange!), 6/24 00:15 (.73). Possible OUR EDGE.
  P2 (SS360 commits on weak drift, we orange): 6/22 12:15 (ns.11), 6/23 13:15 (.05), 6/23 00:00 (.28).
     Same "SS360 over-commit vs our miss?" question as the 1h low-ns bars.

## ★ KEY META-FINDING: our disagreements are SYSTEMATIC, not random. Across 5m/15m/1h they ALL
   fall into P1 (we caught a push they ignored) or P2 (they committed to drift we ignored). This
   consistency = the engine behaves predictably; the open question is uniformly "which of P1/P2
   is edge vs gap" — answerable only by CK's live review. Not noise.

## Engine status (multi-day validated):
  5m  87% (3 days) | 15m 87% (4 days) | 1h 71% (4 days, weaker - low-ns commit gap).
  Comparison CSV updated: 15m now full 6/21-24. All 3 TFs present, tagged.

---

# ★★★ BACKTESTER MILESTONE — 2026-06-26: real trade results on 5 months, market as scorekeeper
# (RESEARCH SANDBOX ONLY — v9 product is FROZEN, ships Sunday, untouched by any of this)

## Data: found CK's 5-month NQ 15m OHLC in uploads (v7 recorder logs, 10k bars Jan20-Jun22,
## full OHLC+SS+rating, integrity verified). Recipe to refresh: run KCR/SS-Line Trinity v7
## recorder on NQ chart -> Pine Logs pane logs OHLC+lines+rating -> export to CSV (10k bar cap
## = ~5mo at 15m / ~7wk at 5m). Other uploads: 5m slice (3d744), 6-ticker bar loggers (ES/CL/GC/RTY/YM/SL).

## Backtest design (CK's specs): entry = SIGNAL BAR CLOSE; test stops (atr1/atr1.5/candle/ssline);
## test exits (fixed 1:1 & 1:2 vs ride-to-opposite-signal); classify WIN/LOSS/GAVEBACK
## (=went +1R our way then reversed to a loss — CK's insight; counts recoverable-by-BE-stop trades).

## RESULT (1798 signals, 5 months, 15m):
## ★ Most configs LOSE (raw signal alone is NOT an edge; trade mgmt does the work). Range
##   -5393 to +7626 pts on the SAME signals by exit/stop choice. Sobering + important.
## ★ WINNERS converge on the CANDLE STOP (stop just beyond signal bar low/high = structural):
##     candle + 1:2 fixed:           +7556 pts, +4.2/trade, 35% win
##     candle + ride-to-opposite:    +7626 pts, +4.24/trade, 33% win
##   Two different exits, same stop, ~same profit -> the STOP is the real lever.
## ★ ROBUSTNESS: candle+1:2 POSITIVE EVERY MONTH (Jan+1154 Feb+243 Mar+1776 Apr+999 May+895
##   Jun+2490). Not one month carrying it. First DURABLE edge measured vs MARKET (not SS360).

## GAVEBACK confirmed real: 34-66/month went +1R then reversed to loss -> CK's break-even-stop
## idea targets a real recoverable population. NEXT TEST.

## HONEST LIMITS (not yet done): 15m only (check 5m next); UNFILTERED (all hours, all ns -> 33-44%
## win has lots of chop; RTH/Globex split + ns filter should raise it); no commissions/slippage
## modeled yet; candle-stop robust but test on 5m + other tickers before any trust.

## NEXT (research, careful, cross-validated each step): (1) BE-stop rule vs gaveback population;
## (2) RTH vs Globex segmentation; (3) ns-strength filter on entries; (4) 5m run; (5) cost model;
## (6) per-ticker. NONE of this touches v9. Nothing goes live until it clears v9-level scrutiny.

---

# ★ CORRECTION — REAL v9 SIGNALS backtest supersedes the approximation — 2026-06-26
CK asked "did you use v9?" -> NO, the +7556 result used a CRUDE direction-flip (1798 signals),
NOT v9's real commit-out-of-orange. Built a FAITHFUL Python port of v9.pine (all 4 layers, real
became_bull/became_bear) and re-ran. (v9 PRODUCT still frozen; this is the research port.)

## Real v9 = 850 signals (vs 1798 crude) — commit-out-of-orange is 2x more selective.

## RESULTS (real v9, candle stop, 5mo 15m):
  candle 1:1:        +366 pts, 48% win
  candle 1:2:        +2760 pts, +3.25/trade, 35% win   <- best, but...
  candle ride-signal:+1746 pts, 22% win
=> profitable but ~1/3 of the approximation's 7556. The extra profit before came from non-v9 signals.

## ★ MONTHLY (real v9, candle 1:2) is LUMPY, NOT positive-every-month:
  Jan +743 | Feb +545 | Mar -158 | Apr +63 | May -4 | Jun +1571
  2 of 6 months flat/negative. Profit concentrated in Jan/Feb/Jun. The "positive every month"
  claim from the approximation was FALSE for real v9. (Exactly why CK asked. Lesson: test the
  REAL engine, never an approximation, before believing a result.)

## ★ BREAK-EVEN STOP test: REJECTED by data. candle 1:2 BE-on = -4460 (vs +2760 BE-off).
  Moving stop to BE at +1R kills gavebacks (186->0) but stops out far more would-be winners before
  they reach 1:2. CK's gaveback population is REAL but BE-stop costs more than it saves. Good
  finding pre-trade. (Other gaveback fixes possible: partial profit at 1R, wider BE trigger — test later.)

## HONEST STATE: real v9 raw signals = net positive 5mo (+2760) but LUMPY -> needs FILTERING
  (RTH/Globex, ns-strength, HTF alignment) to become consistent. That's the next research.
  STILL: validate this Python port against a REAL v9 Pine export when CK can run one (ports can
  differ subtly from shipping code). v9 PRODUCT frozen, ships Sunday regardless.

---

# ★★★ HONEST BACKTEST on REAL v9 + REAL OHLC FILLS — 2026-06-26 (supersedes all prior bt numbers)
CK uploaded via GitHub: nq_pine_logs_combined-v9-6-26-v2.xlsx — 4 sheets, 40k bars:
5m+15m Data Collector (OHLC+lines+ns) AND 5m+15m REAL v9 engine ratings. Downloaded via
raw.githubusercontent (GitHub IS on the bash allow-list — this upload route WORKS).

## PORT VALIDATION (critical): my Python v9 port vs REAL v9 ratings (15m, 10k bars):
  sign match 90% (direction faithful) BUT exact only 62% — my port OVER-GRADES (real says +3
  where mine says +5/+7, ~1960 of 3727 mismatches). Real v9 reaches 5/7 mostly via PERSISTENCE
  ramp at LOW ns (median ns: +3=0.14, +5=0.33, +7=0.28), not my ns-jump thresholds. 
  => MY PORT'S GRADE IS UNRELIABLE. Do not use it. Real v9 grades more CONSERVATIVELY (good).
  => Solution: backtest the REAL v9 ratings directly (CK's engine sheet), no port needed.

## ★ FILL HONESTY MATTERS HUGELY: close-based fills vs real OHLC intrabar fills:
  - Close-based ride-to-signal showed +15,210 pts (LOOKED amazing).
  - REAL OHLC fills (stops trigger on actual low/high): ride = +1,716 with TWO negative months.
  => The +15k was a FILL ARTIFACT (close fills miss intrabar stop-outs). Would have burned live.
     Honest fills REVERSED the "ride is the star" conclusion. ALWAYS use real H/L fills.

## ★ TRUSTWORTHY RESULT (real v9 signals + real OHLC, 15m, 5mo, 918 signals):
  candle stop + 1:2 fixed = +2799 pts, 36% win, +3.05/trade.
  MONTHLY: POSITIVE EVERY MONTH — Jan+780 Feb+202 Mar+218 Apr+986 May+593 Jun+4. 6/6.
  (candle 1:1 = -385 slightly neg; ride = +1716 but 2 neg months, REJECTED as unreliable.)
  => This is the real, honest edge: v9 commit-out-of-orange + candle stop + 1:2 target.
     36% win at 1:2 is profitable (winners 2x losers). Robust across all 5 months.

## STILL TODO (research, v9 frozen/shipped): (1) 5m run (have the data); (2) FILTERS to lift the
  36% — RTH/Globex, ns-strength, HTF align; (3) cost/slippage model; (4) the gaveback population
  (190 trades) — partial-profit-at-1R instead of BE-stop (BE-stop already REJECTED earlier).
  Data route locked: GitHub raw URLs work for uploads going forward.

---

# FILTER TESTS — 5m + 15m, real v9 + real OHLC — 2026-06-26
Applied RTH/Globex, ns-strength, grade filters. candle stop 1:2.

## ★ FILTERS REVERSE SIGN BETWEEN TIMEFRAMES (key structural finding):
  15m: ns>=0.5 = +8.9/trade (vs +3.05 none); grade>=5 = +7.27/trade. High conviction = QUALITY.
  5m:  ns>=0.5 = -10.3/trade; grade>=5 = -9.39/trade. High conviction = LOSS.
  => On 5m a high-ns spike is often EXHAUSTION (end of move), not continuation. On 15m the same
     ns is a sustained push worth joining. Same reading, OPPOSITE meaning by timeframe.
  => grade==3 ("scout") is the ONLY filter profitable on BOTH (5m +2.06/t, 15m +2.4/t).

## ★ v9 is a better 15m engine than 5m engine:
  Unfiltered 15m = +3.05/trade, POSITIVE EVERY MONTH. 5m = +0.55/trade, marginal (7wk sample only).
  Recommend 15m as the primary v9 timeframe.

## MONTHLY robustness of the 15m filters (the honest tiebreaker):
  ns>=0.5: lumpy — Jan+426(6t) Feb+499 Mar-49 Apr+568 May+98 Jun-261. 2 neg months, small samples.
  grade==3: Jan+471 Feb-306 Mar+320 Apr+364 May+746 Jun+356. 5/6 pos, 1 bad month.
  => NO single filter is a clean robust win. They reshape the distribution but none is the magic
     win%-lever. Need more data (esp 5m=7wk only) + likely COMBINATIONS, not single rules.

## HONEST STATE: best trustworthy config remains unfiltered 15m candle-1:2 (+2799, 6/6 months).
  Filters are PROMISING but not yet robust enough to bolt on. v9>15m. 5m needs more history.
  CAVEATS: 5m sample only 7wk; no cost/slippage yet; filters need cross-month + combination work.

## NEXT (research): (1) more 5m history; (2) filter COMBINATIONS not singles; (3) cost model;
  (4) is the 5m-exhaustion effect tradeable as a FADE? (high-ns 5m spike -> fade it). (5) HTF
  alignment (only take 5m signals agreeing with 15m direction).

---

# 5m IDEAS: HTF alignment + spike-FADE — 2026-06-26 (7wk 5m, real v9+OHLC, candle 1:2)
15m win rate for reference: 49% at 1:1, 36% at 1:2 (edge is from R:R not hit-rate).

## IDEA 1 — HTF alignment (take 5m signal only if it agrees with 15m direction): WORKS, modest.
  baseline 5m: +0.55/t | HTF-aligned: +1.04/t (925->328 trades) | HTF-aligned + grade==3: +1.89/t (+532).
  Stacking the two each-helpful filters = best FOLLOW config on 5m. Sensible, more robust direction.

## ★ IDEA 2 — FADE high-ns 5m spikes (trade OPPOSITE the signal): strongest per-trade, LEAST proven.
  FADE ns>=0.5: +2.24/t | FADE ns>=0.7: +4.29/t (best number in test) | FADE grade>=5: +1.86/t.
  CONFIRMS the exhaustion hypothesis: v9 high-ns 5m signals are exhaustion -> fading them profits.
  v9's 5m WEAKNESS (chasing spikes) becomes a STRENGTH inverted.

## ★★ HONEST CAVEATS (critical — do NOT trade these yet):
  - 7 WEEKS of data only. FADE ns>=0.7 = just 56 TRADES. Way too few to trust; could be luck.
  - No monthly robustness possible (7wk too short to split).
  - Fading = trading AGAINST your own engine; mechanically/psychologically harder, needs tight rules.
  => HTF-alignment = safer, more robust, recommend exploring further. SPIKE-FADE = exciting
     HYPOTHESIS, needs MUCH more 5m data before it's anything but that. NOT a validated edge.

## STATE: 15m unfiltered candle-1:2 (+2799, 6/6mo, 36% win) = the trustworthy edge. 5m improvements
  (HTF-align, fade) are PROMISING on tiny sample. Need months of 5m data to validate either.
  v9 PRODUCT frozen/shipped regardless. All this is research.

---

# ★★★ BEST RESULT YET — 15m REVERSE-SIGNAL EXIT + 4xATR STOP — 2026-06-26
CK asked: did I test exit-on-reverse-signal on 15m? Tested properly now. It's the biggest
improvement in the project.

## 15m exit comparison (real v9 + real OHLC, 5mo, 918 signals):
  fixed 1:2 (prior best):        +2799, 36% win, 6.1 bars held
  reverse-signal ONLY (no stop): +8506, 37% win, 12.7 bars  <- but UNBOUNDED risk (worst -538)
  reverse + candle stop:         +1716, 22% win  (too tight, cuts recoveries)
  ★ reverse + 4xATR stop:        +9971, +10.86/trade  <- BEST: rides trend AND caps risk

## ★ The 4xATR stop is the Goldilocks: wider than 95% of trades' adverse excursion (175pt MAE)
   so it doesn't cut recoveries, but caps catastrophe (worst trade -538 -> -306). It BEATS the
   no-stop version (+9971 > +8506) AND is safer. Stop-width plateau 3x-6x all strongly positive
   (robust, not a spike); 2x breaks it (-553, too tight).

## Monthly (reverse + 4xATR): Jan+1761 Feb+1037 Mar+2646 Apr+1129 May-119 Jun+3685. 5/6 strongly
   positive, May barely neg. ~3.5x the fixed-1:2 total, MORE robust.

## ★ VINDICATES CK's long-held instinct: v9's edge is in the HOLDS (ride to reversal), not
   clipping quick targets. The combined B/SC<->S/BC signal is meant to capture whole moves.
   Exiting on the engine's OWN reversal honors the design. This is what the -400 crash hold
   foreshadowed weeks ago.

## HONEST CAVEATS: worst single trade -306 pts (~$6100/NQ contract, ~$600/MNQ) — must be able to
   absorb; the strategy WORKS because it takes those (can't flinch — 2xATR proves cutting early
   breaks it). Holds ~13 bars (~3.25hr on 15m) = patient swing style. No costs modeled yet.

## NEW BEST CONFIG: 15m, v9 commit signals, enter signal close, exit on opposite v9 signal,
   4xATR catastrophe stop. +9971 pts / 5mo / +10.86 per trade / 5-of-6 months positive.
   Still RESEARCH; validate forward + model costs before any trust. v9 product frozen/shipped.

---

# ★ TRADINGVIEW STRATEGY-TESTER CROSS-CHECK (independent of Python) — 2026-06-27
After fixing the strategy (was silently blocked: initial_capital 50k < NQ 1-contract notional
~586k; TV emulator refuses unaffordable entries SILENTLY — signals plotted, 0 trades. Also was
missing //@version=6 -> defaulted to Pine v1. Both fixed.) CK ran the real TV backtest.

## RESULT (TV Strategy Tester, real NQ1!, 11 months Aug2025-Jun2026, 1710 trades):
  Total +$92,630 | +$54/trade | 34% win | avg win $2058 / avg loss -$1016 | worst -$10,755 (-538pt)
  In POINTS: +4632 pts total, +2.71 pts/trade.

## ★ COMPARE to Python backtest (5mo Jan-Jun2026, +10.86 pts/trade):
  TV's 11mo = +2.71/trade vs Python's 5mo = +10.86/trade. GAP IS REAL + INFORMATIVE:
  - TV's longer window includes Aug-Dec2025 which has BAD months Python never saw:
    Oct2025 -1142pt, Nov2025 -1673pt, Dec2025 -203pt (~-$60k combined drawdown stretch).
  - The Jan-Jun2026 months MATCH well (Jan+1204 Feb+1666 Apr+963 Jun+2523) — same strong period.
  => Both engines AGREE on core finding (reverse-exit+4xATR is NET PROFITABLE on NQ) BUT the
     5-month Python window FLATTERED the edge. 11-month truth = profitable but LUMPY with real
     multi-month losing stretches. The honest picture.

## MONTHLY (TV, pts): Aug+529 Sep+395 Oct-1142 Nov-1673 Dec-203 Jan+1204 Feb+1666 Mar+178
  Apr+963 May+192 Jun+2523. => 8 of 11 positive, but 3 consecutive negative (Oct-Dec) = a real
  ~quarter-long losing stretch a live trader must survive. Worst trade -$10,755.

## LESSON (METHODOLOGY): the independent TV cross-check did its job — exposed that the Python
  5mo result was window-flattered. ALWAYS validate across the longest history + a second engine.
  Edge is real but must be traded with full awareness of multi-month drawdowns.

## TV-vs-Python fill differences also contribute to per-trade gap (emulator intrabar assumptions
  vs Python H/L fills); expected, secondary to the window effect.

---

# RTH vs GLOBEX FILTER on real TV strategy data (11mo, 1710 executed trades) — 2026-06-27
Classified by ENTRY time. NQ RTH = 09:30-16:00 ET.

## SPLIT:
  RTH:    460 trades  | 40% win | +$39,780 (1989 pts) | +4.32 pts/trade  <- HIGHER QUALITY
  Globex: 1250 trades | 32% win | +$52,850 (2642 pts) | +2.11 pts/trade  <- more trades, weaker
  => RTH = 2x better per-trade + higher win rate (real liquidity, cleaner trends). Globex thinner/choppier.

## ★ KEY FINDING: the bad Oct-Dec2025 drawdown was MOSTLY GLOBEX.
  Nov2025: RTH +1330 pts but GLOBEX -3002 pts (the single worst block). The overnight session
  is where the big losing stretch came from. RTH was POSITIVE in Nov.
  => RTH-only would DODGE the worst blowups + give up the big Globex winners (Jun glx +2419).

## MONTHLY (neither session uniformly positive):
  RTH:    Aug-23 Sep+575 Oct-942 Nov+1330 Dec-182 Jan-106 Feb+564 Mar-16 Apr+335 May+350 Jun+104
  Globex: Aug+552 Sep-180 Oct-200 Nov-3002 Dec-20 Jan+1309 Feb+1102 Mar+193 Apr+628 May-158 Jun+2419
  => RTH smoother/higher-quality but still has down months (Oct). Globex higher-variance both ways.

## HONEST READ (hypothesis, not deployable rule yet): RTH-only = better win%, better per-trade,
  avoids worst Globex drawdowns -> SMOOTHER. Cost: fewer trades, miss big Globex winners. Does NOT
  manufacture an every-month-positive edge (RTH Oct still -942). A real IMPROVEMENT, not a fix.
  NEXT: test RTH-only equity curve / max drawdown directly; consider RTH + ns/grade filter combo.

---

# ★ 2026-ONLY RE-ANCHOR (CK: drop 2025, focus 2026, prioritize TOTAL profit) — 2026-06-27
CK directive: stop chasing the 2025 crash/whipsaw (different regime). Trade the CURRENT 2026
regime. Have 2 quarters (Q1 Jan-Mar, Q2 Apr-Jun). Train Q1 / test Q2 discipline. Want TOTAL
profit, plainer (not per-trade quality).

## RESULT (2026 only, reverse-exit+4xATR, by quarter, TOTAL pts):
  baseline(plain): Q1 5317 | Q2 4654 | TOTAL 9971  <- WINS, positive+balanced BOTH quarters
  long_bias:       Q1 2196 | Q2 6600 | TOTAL 8796
  htf_align:       Q1 2854 | Q2 4062 | TOTAL 6916
  long_only:       Q1  281 | Q2 5895 | TOTAL 6175

## ★ PLAIN v9 BEATS ALL ADAPTIVE FILTERS on 2026 total profit. Every filter REDUCED total.
  CK's instinct (prioritize total, go plainer) = CORRECT, data-confirmed.
  WHY filters hurt: long_only/long_bias had terrible Q1 (281/2196) — Q1 2026 had tradeable
  SHORT moves they skipped. Only shine in Q2's clean uptrend. Quarter-inconsistent = the
  out-of-sample test EXPOSED them (a long-bias fit to Q2 would FAIL in Q1). htf_align just
  makes less (sits out winners).

## ADAPTIVE STRATEGY VERDICT: the adaptive version (HTF-filter+crashguard) made +15.19/trade
  but LESS TOTAL ($55k vs $92k/11mo; 6916 vs 9971 pts/2026). Crash-guard missed Oct (whipsaw
  not crash, HTF flat). For TOTAL PROFIT on 2026 regime: RUN PLAIN v9. Filters not earning place.

## DISCIPLINED PATH FORWARD: run PLAIN (wins now). Collect Q3/Q4 data. RE-RUN this quarter-by-
  quarter variant test as each quarter completes. If a filter starts beating baseline across NEW
  quarters = real regime shift, adapt THEN (proven forward, not curve-fit). The variant-comparison
  framework is the tool; let DATA pick the variant per quarter. Don't pre-commit to a filter.

## KEEP: SS_v9_ADAPTIVE_strategy.pine exists w/ dashboard+regime detection (useful framework +
  visual), but for live 2026 total-profit = plain SS_v9_PAPER_strategy_v2 is the call.

---

# ★★★ SS360 5m LEDGER ANALYSIS — scoring THEIR signals (not ours) — 2026-06-27/28
CK provided full multi-day ledger (ss360-ledger.xlsx): 5M (3 days 06-23/24/25, 203 events),
15M (6 days), 1HR (5 days). These are SS360's actual COLOR-TRANSITION events. Prices only
filled for 06-23, so scored by MATCHING each ledger signal's timestamp to OUR real 5m NQ OHLC
(v9_merged_5m covers all 3 dates, 276 bars/day). All 203 events matched 100%.

## STEP 1 — 5M ALONE (SS360's 5m signals scored on real NQ OHLC):
  Entry = commit out of orange to color (or B/SC/S/BC); exit = orange-return or opposite signal.
  RESULT: 87 trades | 77% WIN | +3554 pts | +40.85/trade | positive ALL 3 DAYS
  (06-23 +1336/24t, 06-24 +814/29t, 06-25 +1404/34t)
  LONGS 45 @77% +48.3/trade | SHORTS 42 @76% +32.9/trade | best +835 worst -420

## ★ SKEPTIC CHECKS (why +40 when v9-on-5m was only +0.55?):
  - Hold length: median 2 bars, avg 3.1. 51/86 exit within 3 bars = REAL FAST SCALPING.
    (matches 5m structure finding: runs die in ~2 bars). One outlier held 169 bars.
  - ROBUSTNESS: capped hold-time still strong — 3bar cap +27.45/trade@77%, 5bar +35, 10bar +40.
    Stable across ALL caps = REAL EDGE, not long-hold flukes. THIS IS THE KEY VALIDATION.
  - Days had big ranges (792-1006pt) so lots to capture; high vol regime.

## ★★ INTERPRETATION: SS360's OWN 5m signals are GOOD (+27-40/trade, 77% win) where our v9-on-5m
  FAILED (+0.55). The difference = SS360's 5m engine catches the fast 2-bar scalp moves; our v9
  (a 15m trend-follower ported to 5m) doesn't. CONFIRMS CK's hypothesis: SS360 likely uses a
  DIFFERENT (faster/scalp) engine on 5m. We should build a 5m engine around THIS behavior.

## CAVEAT: only 3 days (high-vol regime). 77%/+27 is promising but NOT yet cross-validated across
  regimes. Need more ledger days (esp. calmer days) before trusting as deployable. Same discipline.

## NEXT (CK's sequence): STEP 2 = 5m signals aligned w/ 15m TF. STEP 3 = 5m aligned w/ 15m AND 1hr.

---

# ★★ STEP 2 — 5M signals × 15M alignment (confluence test) — 2026-06-27/28
Built 15m STATE timeline from 15m ledger (152 transitions/6 days), tagged each 5m signal by
whether 15m TF agreed/opposed/neutral at entry time.

## RESULT:
  ALL 5m:                86 trades | 76% | +41.16/trade
  5m ALIGNED w/ 15m:     32 trades | 75% | +51.55/trade  <- CONFLUENCE = best
  5m AGAINST 15m:        11 trades | 72% | +15.91/trade  <- conflict = weak (3x worse)
  5m while 15m NEUTRAL:  43 trades | 79% | +39.88/trade  <- still good

## ★ CONFLUENCE EFFECT IS REAL: aligned +51.55 vs against +15.91 = 3.2x better per trade.
  KEY NUANCE: win% barely moves (75 vs 72) — what changes is SIZE. Aligned trades = bigger/cleaner
  (ride 5m moves the 15m trend pushes); against = small choppy scalps counter to bigger flow.
  Half of all 5m signals (43) fire while 15m NEUTRAL and are still strong (+40) — so DON'T only
  trade when 15m agrees (would skip half the opportunities). The weak bucket to AVOID = AGAINST.

## ACTIONABLE CONFLUENCE RULE (emergent): take 5m when 15m AGREES (+51) or NEUTRAL (+40);
  be cautious/skip when 5m FIGHTS 15m (+16). Trims worst trades w/o killing many (only 11/86 against).

## CAVEAT: 3 days, high-vol, small sub-buckets (11 against). Directionally clear but needs more
  days to confirm the +51 vs +16 split. NEXT: STEP 3 = 5m aligned w/ 15m AND 1hr (triple confluence).

---

# STEP 3 — 5M × triple confluence (15m + 1hr) — 2026-06-27/28 — RESULT: UNDETERMINED
Built 1hr state timeline (40 transitions/5 days), tagged 5m signals by 15m AND 1hr agreement.

## RAW (3 overlap days, n=86 split 5 ways):
  TRIPLE aligned (all 3 agree):  14 | 57% | +34.41/trade
  5m+15m agree,1hr not:          18 | 88% | +64.88/trade
  5m+1hr agree,15m not:          14 | 64% | +28.29/trade
  5m solo (HTFs neutral):        41 | 78% | +33.86/trade
  5m with HTF AGAINST:           31 | 83% | +53.85/trade

## ★ HONEST VERDICT: these sub-buckets are TOO SMALL to trust. Bootstrap: if true win=76%, a
  random 14-trade bucket ranges 50-93% win (95% CI). So triple's "57%" and the "88%" are BOTH
  within noise of the overall 76%. NOT real differences. The clean Step-2 confluence story does
  NOT clearly extend to 3 TFs on this data — but we CANNOT conclude it fails either. UNDETERMINED.

## Plausible (UNPROVEN) mechanism if triple-align really does underperform: when the SLOW 1hr
  finally agrees, the move is MATURE/late = entering near exhaustion (HTF-confirmation lag). Worth
  testing with more data, NOT a claim yet.

## ROBUST STATEMENTS ONLY:
  - 5m signals overall: ~76% win, +41/trade (n=86, solid for 3 days).
  - Step 1 (5m alone) + Step 2 (aligned>against, +51 vs +16) = directionally real but need more days.
  - Step 3 (triple) = can't separate signal from noise at n=14/bucket. NEED MORE LEDGER DAYS.

## THE GATING ISSUE for all 5m work now: only 3 days of priced 5m. Everything points promising but
  NOTHING is cross-validated across regimes. #1 priority before any 5m engine = MORE 5m LEDGER DAYS,
  esp. calmer/non-high-vol days. Then re-run Steps 1-3 with enough n per bucket.

---

# ★★★ 5M ENGINE — EXIT DESIGN VALIDATED (deliberate test of 7 exit variants) — 2026-06-27/28
CK directive: build BETTER 5m product (option 3: keep v9 direction-core, rebuild trigger+exit for
5m structure; use what fits from v9/SS360, match neither slavishly). Tested exits before building.

## EXIT VARIANT BAKE-OFF (SS360 5m entries, n=86, real OHLC):
  natural signal-exit (orange OR opposite): 76% | +41.16/trade  <- WINNER
  orange-return priority:                   76% | +41.16  (identical)
  opposite-commit only:                     76% | +41.16  (identical)
  40% peak-giveback:                        75% | +38.47
  scout-run + fast others:                  75% | +35.71
  half-out +1ATR / half natural:            81% | +28.22  (higher win, less total)
  ATR trail 1.5x:                           63% | +4.79   <- WORST (gives back)
  fixed 3-bar cap:                          77% | +27.45

## ★ KEY FINDING: holding PAST the natural exit gives back profit 72% of the time. SS360's
  return-to-orange/opposite-commit is a WELL-TIMED momentum-death signal. NO trail, NO runner,
  NO scaling improves it. The DISCIPLINED fast signal-exit IS the +41/trade edge.
  (orange-return & opposite-commit nearly always COINCIDE on 5m = move dies and reverses together.)

## CK chose options 2+3 (let scout run + trail); DATA OVERRULED both (trail +4.79 vs fast +41).
  Tested instead of built. CK agreed to test more first; all alternatives lost. Pure fast-exit wins.

## ★★ 5M ENGINE ARCHITECTURE (data-validated, ready to build):
  Direction: v9 sign(dSS)+deadband [KEPT from v9 — works any TF, 99% cross-day]
  Trigger:   FRESH COMMIT out of neutral, act IMMEDIATELY (78%/+42 mechanic; early/scout wins 90%)
             [REBUILT — v9's persistence-ramp is too slow for 5m's 2-bar runs]
  Grade:     early/scout = act now, no persistence wait [INVERTED from v9]
  Exit:      return-to-neutral OR opposite-commit, whichever first [REBUILT — fast, no trail/runner]
  Vol gate:  favor moderate ATR, stand off in EXTREME [NEW — low-ATR won 81% vs high 72%]
  Longs +49/trade, shorts +33 (both strong). Core principle: v9=patient trend-rider;
  5m=disciplined commit-scalper trusting momentum-death exit.

## NEXT: write the 5m engine Pine indicator (//@version=6) implementing the above.

---

# ★★★ v1 5m ENGINE — HEAD-TO-HEAD vs SS360 (CK loaded v1, gave CSV) — 2026-06-28
CK loaded SS_5m_Engine v1 on NQ 5m, exported CSV (10k bars, 3 ledger days included). Scored v1's
ACTUAL engine output on real OHLC, compared to SS360's ledger signals on the same 3 days.

## HEAD-TO-HEAD (same days, same scoring):
  SS360 ledger:  87 trades | 77% win | +40.85/trade | +3554 total
  OUR v1:       100 trades | 44% win | +17.37/trade | +1737 total
  v1 fires 33/day (vs SS360 29/day) — similar activity (my earlier emulation wrongly guessed 65/day).

## ★ DIAGNOSIS (split v1 trades by whether they align w/ an SS360 signal ±10min):
  v1 that MATCH SS360:  30 trades | 53% | +35.76/trade  <- nearly as good as SS360!
  v1 EXTRA (SS360 silent): 70 trades | 40% | +9.49/trade  <- the drag
  => Architecture is SOUND. When v1 fires where SS360 fires, it's a good signal (+35.8). v1's
     problem = too trigger-happy, takes 70 weak EXTRA commits SS360 correctly skips.
  NOTE: matched & extra commits had ~identical ns (0.38 vs 0.40) — so ns-filter is a general
  QUALITY filter, NOT an "SS360-likeness" filter. Honest distinction.

## ★★ FIX (validated, positive ALL 3 days): commit-threshold t_flat 0.15 -> 0.35.
  ns filter sweep: 0.0=44%/+17 | 0.25=46%/+20 | 0.35=53%/+27 | 0.5=59%/+47(only 27 trades).
  0.35 = BEST BALANCE: 58 trades, 53% win, +27.34/trade, +1586 total, positive 3/3 days
  (06-23 +471, 06-24 +759, 06-25 +356). Applied as v2 default.

## STATUS: v1->v2 (t_flat=0.35). Still behind SS360 (+27 vs +41) but much closer, and the gap is
  now understood (v1 takes more marginal commits). v2 = better. NEXT live test: load v2, record
  SS360 5m ladder again, re-compare. The matched-signal quality (+35.8) shows the ceiling is near
  SS360 if we keep tightening toward only the high-conviction commits. DON'T overfit to 3 days.

---

# ★★★ CRITICAL — v2 FULL-RANGE TEST OVERTURNS THE 5m FOLLOW THESIS — 2026-06-28
CK ran v2 (t_flat=0.35) on TV, gave full CSV (May7-Jun26, 10k bars/7wk). Scored across FULL range,
not just 3 ledger days. THE 3 LEDGER DAYS WERE WINDOW-FLATTERED.

## FOLLOW (v2 as built) — FULL 7 weeks:
  -843 pts | 37% win | -1.32/trade | only 2/8 weeks positive.
  3 ledger days (week 26) = +22/trade BUT that was the BEST week. No commit-threshold saves the
  full range (0.35=-1.3, 0.5=-2.0, 0.7=-3.6, 1.0=-9.4). FOLLOW IS STRUCTURALLY WRONG FOR 5m.

## ★ FADE (trade OPPOSITE the commit) — FULL 7 weeks:
  +1.36/trade base | +3.49 @ns>=0.5 | +6.75 @ns>=0.7 | 53-57% win.
  OUT-OF-SAMPLE: positive BOTH halves (1st +1.68, 2nd +5.32 @0.5; +5.27/+8.13 @0.7). 5/8 weeks pos.
  This snaps back to our WEEK-1 finding: high-ns on 5m = EXHAUSTION not continuation. 5m is a
  FADE/mean-reversion market. Following commits loses; fading the exhaustion wins.

## ★★ BUT FADE'S WEAKNESS = the mirror image: week 26 (the 3 trending ledger days) fade LOSES
  -10/trade. Exactly when FOLLOW wins. So:
    - choppy/mean-reverting days (most): FADE wins (wk20-25)
    - strongly trending days (rare): FOLLOW wins (wk26 = the ledger days)
  NEITHER pure direction wins everywhere. 5m edge is REGIME-DEPENDENT (fade chop / follow trend).
  = exactly the "5m choppy, no strong SIMPLE edge" structure we found at the very start.

## HONEST ADMISSION: built v2 as a FOLLOW engine off the 3-day ledger; full data says follow is
  backwards for 5m's normal (choppy) regime. The ledger came from a rare trending stretch. The
  +27/trade was real for those days, meaningless general. CLASSIC window-flatter — 2nd time this
  project (also hit on the 15m Python 5mo vs TV 11mo). The discipline (full-range + OOS) caught it.

## DECISION NEEDED (do NOT just flip v2 to fade — that loses on trend days):
  Options: (a) regime-switch 5m engine (fade in chop, follow in trend — needs a regime detector
  that itself must be validated, risk of overfit); (b) pure fade engine, accept it loses on rare
  trend days (simpler, +5-8/trade base, OOS-positive); (c) pause 5m, the edge is marginal/hard;
  (d) more data first. CK to decide. The fade is the most robust SINGLE finding so far.

---

# ★★★★ BREAKTHROUGH — DECODED SS360's 5m SIGNAL GRAMMAR (reverse-engineered, 3 days) — 2026-06-28
CK directive: stop asking for more data, reverse-engineer the 3 days we HAVE. Understand their
sequence, don't conclude quickly. Target: reproduce those 3 days' signals first.

## ★ THE DECODE — SS360 uses a TWO-STAGE COMMIT (this was the whole missing piece):
  Stage 1: ORANGE → LEMON GREEN = SCOUT/tentative (a "warning, not yet"). NOT a signal.
  Stage 2: LEMON GREEN → GREEN + "PRINTS N" = CONFIRMED commit. THIS is the B/SC signal.
  v2 was firing on STAGE 1 (orange-exit) — taking every tentative poke (84 sigs, 47%).
  SS360 fires on STAGE 2 (the deepening + rating print) — 24-30 confirmed sigs.

## SIGNAL-DEFINITION BAKE-OFF (3 ledger days, signal-to-signal exit):
  orange-exit (what v2 did):     84 sigs | 47% win | +26.73/trade
  DEEPENING commit (stage 2):    24 sigs | 66% win | +72.43/trade  <- 2.7x better per trade

## ★ EXIT FIX: 06-23 looked NEGATIVE with signal-to-signal exit, but at the BAR level its
  deepening signals are 72% win (+6bar). The "loss" was an EXIT ARTIFACT — holding to next
  opposite signal let a few big reversals (13:20 BUY -122) wipe many small wins on a whippy day.
  FIXED SCALP EXIT (target/stop, intrabar) fixes it.

## ★★ FULL DECODED SYSTEM (deepening signal + fixed scalp exit, all 3 days):
  tgt30/stop25: 30 sigs | 66% win | +11.7/trade | per-day 72%/57%/60%
  Big jump from v2's 37%. NEAR SS360 but not yet their 77%.

## STILL OPEN (honest): the last ~10% to 77%. Likely the buy/sell RATING ASYMMETRY (their tags:
  buys fire at rating +3/+5, sells at -5/-7) + possibly a structure/extension filter. Still 3 days.
  Some signals are REVERSALS at extreme (PRINTS -7 AND RED TO GREEN AND S/BC = fade the -7).

## REBUILD PLAN (v3): fire on STAGE-2 deepening (lemon→green confirm), not orange-exit. Add the
  scalp-exit option. Keep v9 colors/alerts/log. This is the evidence-based rebuild; the last 10%
  needs more sequence-decoding (and eventually more days) but don't overfit 3 days to chase it.

---

# ★★★★★ ROOT-CAUSE FOUND — our 5m engine COLLAPSES TO NEUTRAL (CK insight) — 2026-06-28
CK corrected two errors: (1) "not enough 5m data" was WRONG — it's 10,000 bars (= 15m) and 249
ledger events (MORE than 15m's 192). Same TV bar cap. Count RECORDS not days. (2) I had the
problem backwards — chasing "over-firing" / more selectivity (v3), when the real flaw is the
OPPOSITE.

## ★ THE BAR-BY-BAR COMPARISON (enabled by reconstructing SS360's rating from all 249 events):
  Reconstructed SS360's per-bar rating (color->base, PRINTS N->explicit). Compared to our rating.
  - SIGN AGREEMENT: only 19%. Disagreements nearly all "THEIRS ±5 vs OURS 0".
  - ★ OUR ENGINE IS NEUTRAL 86% OF BARS. SS360 is neutral only 19%. We sit at ORANGE far too much.
  - Our rating-hold = median 2 bars then collapses. SS360 HOLDS the rating through the move
    (orange->lemon->green->prints5 = one held position).
  - NOT a timing offset (shifting only 19%->29%). STRUCTURAL.

## ★★ ROOT CAUSE: the counter-neutralize + fast-release logic (built during the FADE hypothesis
  exploration) makes our engine collapse to orange on every counter-bar. SS360 COMMITS AND STAYS
  COMMITTED. We need rating PERSISTENCE, not selectivity. The 641 "commits" weren't over-firing —
  they were our rating flickering 0->±->0 because it can't HOLD state. THAT's why every signal
  definition failed — the underlying rating is unstable.

## ★★★ THE FIX (validated on data): HOLD the rating, neutralize only on a real pause.
  - pure-persistent (hold til confirmed reversal): 0% neutral, 51-57% sign agreement (overshoot).
  - HYBRID (hold rating, go orange when PRICE CROSSES the line): confirm=1 -> 13% neutral
    (~matches their 19%), 53% SIGN AGREEMENT. Up from 19%!
  => agreement 19% -> 53% just by fixing persistence. This is the structural key.

## v4 BLUEPRINT (persistence-first, replaces v3's selectivity-first):
  Direction: sign(line-speed) [keep]. Commit when line+price agree. HOLD the rating through
  counter-bars/wobbles. Neutralize ONLY when price crosses back through the SS line (their orange-
  return). This makes our rating timeline resemble theirs (the prerequisite for matching signals).
  Then re-derive B/SC=commit-from-orange on the PERSISTENT rating.

## CK's process lesson (correct): count records not days; use ALL 249 events not just 30 entries;
  the rating STATE MACHINE is recorded in the events, it was never "invisible" — I was discarding it.

---

# ★★★★★ THE REFRAME THAT WORKED — STOP MATCHING SS360, TRADE THE CHART (CK) — 2026-06-28
CK: "if you were trading this yourself with no SS360 to check, what would you do? We're over-
complicating by matching them instead of making a profitable session. Look at OHLC — a simple
decision: trade or not, when to enter/exit." DROPPED all SS360 reverse-engineering.

## GROUND-LEVEL TRUTH (just looked at the days):
  56% of days (25/44) have a real directional move (+560,-1334,+787,-656 pt moves).
  44% chop/round-trip. THE EDGE = know which day you're in, only press on trend days.

## ★★★ DEAD-SIMPLE OPENING-RANGE-BREAK strategy — FULL 44 days (NOT 3-day flattered):
  Rule: first 30min = opening range. Price breaks + holds (1 close beyond) → take that direction,
  ride to end of day, exit if price falls back inside OR (= day failed).
  RESULT: +1680 pts | +38.2/day | 29% win | avg WIN +240 vs avg LOSS -46.
  ROBUST: profitable without best day (+1130), without best 3 days (+157). Winners SPREAD across
  May 8/15/24/28, Jun 2/7/15/23 — not clustered, not one regime. 
  *** THIS SURVIVES THE FULL DATA — every SS360-matching attempt (follow/fade/deepening/persist)
      died out-of-sample. This works because it's grounded in how markets move, not fit to 3 days. ***

## WHY IT WORKS: low win% (29%) is FINE — it's a trend-catcher. Most days break fails = small -46
  loss. Trend days = +240 win. Wins dwarf losses. Classic asymmetric trend-following.

## THE LESSON (CK was right all along): we overcomplicated 5m for WEEKS trying to reproduce SS360's
  grade/color/signal. The actual tradeable 5m edge is what a trader SEES on the chart: let the day
  declare direction, go with it, ride, cut if it fails. No grade, no color ladder, no matching.

## NEXT: build this as a clean 5m strategy (OR-break + hold + EOD/fail exit). Tune OR length (30min
  best so far) + entry/exit. Then v9-style cross-validation. This is the 5m PRODUCT direction now.
## NOTE: SS360 work not wasted — the v9 DIRECTION engine + persistence finding may COMBINE with the
  OR-break (e.g. only take OR-breaks aligned with the line direction). Test later.

---

# ★★★★★ THE 5m LINE — RMSE-FIT TO SS360 (the 15m method I'd SKIPPED) — 2026-06-28
CK: this project is "build a signal line, then play around it." On 15m we cracked the LINE FIRST
(RMSE-fit smoother to SS360's hand-read line values -> TEMA-12 won @ RMSE 0.081), then everything
followed. I SKIPPED this on 5m — assumed TEMA-12 transfers, jumped to signals/fade/persist/ORB.
Went back, did the line fit properly.

## ★ KEY DISCOVERY: SS360's 5m LEDGER PRICES ARE THEIR LINE VALUES (not close prices!).
  On 06-23 they logged a price at each color event. Checked: median |their_price - our_close|=14.5pt
  (far), but median |their_price - our_SS(TEMA12)|=2.1pt (tracks!). So the ledger embeds their
  actual 5m LINE HEIGHT at 27 timestamps. This is the 5m equivalent of the 15m hand-reads — the
  data needed for the RMSE line-fit was in the ledger all along.

## ★★ RMSE BAKE-OFF (smoother vs their 27 real 5m line values, ±1-bar timing jitter):
  TEMA-13: RMSE 1.70  <- WINNER, clean trough
  TEMA-12: 4.03 | TEMA-14: 2.91 | TEMA-11: 4.78 | TEMA-10: 6.06 | (EMAs far worse, as on 15m)
  => The 5m LINE = TEMA-13 (15m was TEMA-12; 5m wants slightly MORE length). Same TEMA family,
     confirming the line approach transfers — just needs its own length, which I never measured.

## ACTION: updated SS_5m_Engine_v3.pine line: ss_len 12 -> 13. This is now the RMSE-correct 5m line.

## THE LESSON (CK right again): I kept building signals/strategies (fade, deepening, persistence,
  ORB) on an UNFITTED line. On 15m we got the LINE right first (RMSE), THEN built. The 5m line was
  never properly fit until now. With TEMA-13 as the correct line, the signal work (price-vs-line
  behavior) can be redone on a sound foundation — the way 15m worked. NEXT: rebuild the price-around-
  line read (AM/grade) on the TEMA-13 line, validate vs ledger. Don't skip steps this time.

---

# 5m AM REBUILT ON FITTED TEMA-13 LINE — direction solid, grade still the wall — 2026-06-28
After fitting line to TEMA-13, rebuilt the AM (price-around-line read) the 15m way, validated vs
SS360 rating timeline.

## DIRECTION (the foundation) — NOW SOLID:
  Our TEMA-13 line SLOPE (lb=1) vs their rating sign = 73% raw. (price-vs-line only 58%, price-dir
  68% — so LINE SLOPE is the right direction signal, confirming the framework.)
  Clean AM (line-slope + flat band): t_flat=0.05 -> 70% sign / 5% neutral; t_flat=0.15 -> 60% sign /
  17% neutral (matches their 19% neutral but costs direction). TENSION: their neutral isn't a pure
  flat-band on our line — they go orange for a reason (likely price re-crossing line) our slope misses.

## GRADE (3/5/7) — STILL THE WALL: color/grade agreement only ~16% regardless of t_flat. The line-fit
  did NOT fix grade (grade is a separate computation from line-slope). Consistent with earlier finding
  that 6 grade-driver hypotheses (ns, dist, run-length, etc.) all failed. Grade remains unsolved.

## HONEST STATUS of the 5m line+AM rebuild (done the RIGHT way this time, line-first):
  ✓ LINE: TEMA-13, RMSE-fit to their actual 5m line (1.70). SOLID.
  ✓ DIRECTION: ~70% sign agreement on the fitted line. SOLID foundation.
  ✗ GRADE: ~16% color match. NOT solved — same wall as before.
  ✗ NEUTRAL TIMING: their orange-returns need a price-cross rule we haven't pinned.

## NOTE: 15m reached high agreement because BOTH its line AND its grade(structure: stack/LR/V) fit
  SS360. On 5m the grade structure may differ — their 5m grade may not be the 15m stack/LR/V logic.
  That's the next thing to decode IF pursuing exact-match. BUT for TRADING, ~70% direction on a
  correct line is already a usable signal foundation (direction is what drives entries).

---

# ★★ 5m SIGNALS on TEMA-13 structure — HOLD-CONFIRM is the key — 2026-06-28
Built signals on the fitted TEMA-13 line + clean AM. Raw "commit out of neutral" over-fires (54/day,
negative). THE FIX: require the commit to HOLD before firing (like 15m — don't fire on first cross).

## HOLD-CONFIRM SWEEP (t_flat=0.35, scalp 30/20, FULL 7 weeks):
  hold=1: 48/day, 40% win, NEGATIVE
  hold=2: 28/day, 58% win, +8.4/trade
  hold=3: 19/day, 72% win, +15.2/trade  <- looked great BUT...

## ★ SKEPTIC CATCH (honest): the +15/trade ENTERED AT THE COMMIT BAR, but the hold isn't KNOWN
  until 2 bars later = look-ahead bias. Entering REALISTICALLY (at the confirm bar, when you
  actually know it held): +1.4/trade, 43% win, +1150 total. MUCH more modest.
  *** Caught my own look-ahead bias instead of reporting the inflated number. ***

## REALISTIC RESULT (entry at confirm bar, FULL range):
  838 sigs | 43% win | +1.4/trade | +1150 total | 6/8 WEEKS POSITIVE | robust w/o best week (+787).
  First 5m signal that holds up HONESTLY across full data with proper entry timing. Modest but real.

## STATUS: 5m foundation now sound — fitted TEMA-13 line (RMSE 1.70), 70% direction, hold-confirm
  signal discipline (the 15m lesson: confirm before firing). Edge is modest (+1.4/trade) but robust
  and honestly measured. NEXT: improve entry (the look-ahead gap suggests entering FASTER or a
  pullback entry after confirm), and/or add a directional/HTF filter. The base is finally solid.

---

# 5m SIGNALS — applied the 15m signal-fixing levers — 2026-06-28
Recalled the 15m signal history: the key 15m lever was MAGNITUDE HYSTERESIS (sticky structural
booleans + ATR-fraction deadband so +7 holds through pullbacks instead of stepping 7->5->3), plus
counter-poke skip. Tested all on the 5m TEMA-13 structure.

## RESULTS (FULL range, scalp 30/20, realistic confirm-bar entry):
  - 15m-style REGIME-FLIP + hysteresis: ~40% win, breakeven. WORSE than hold-confirm.
  - HOLD-CONFIRM commit (the winner): hold=3 -> +1.4/trade, 43% win, +1150, 6/8 wks, +787 no-best.
  - Refractory period: adds nothing (hold=3 already spaces signals ~19/day).
  - hold=2: negative. hold=3: positive. The 3-bar confirm is THE driver.

## HONEST FINDING: the 15m's magnitude-hysteresis was for DISPLAY cleanliness (long color runs),
  NOT for tradeable-signal edge. On 5m it doesn't lift the number. The robust 5m signal is simply
  the HOLD-CONFIRM commit (require 3-bar hold before firing, enter at confirm bar). Stable at
  +1.4/trade, 6/8 weeks positive, survives dropping best week.

## CONSOLIDATED 5m STATUS (built the right way, line-first):
  ✓ LINE: TEMA-13 (RMSE-fit 1.70 to their actual 5m line)
  ✓ DIRECTION: ~70% sign agreement on fitted line (line-slope is the signal)
  ✓ SIGNAL: hold-confirm commit, t_flat=0.35, hold=3 -> +1.4/trade, 43%, 6/8 wks (HONEST, full range)
  ~ GRADE: ~16% color match (unsolved, but CK accepts grade diff like on 15m)
  The 5m edge is MODEST but REAL and robust — the first thing that survives full-range OOS honestly.

## NEXT OPTIONS (for later): (a) HTF filter — only take 5m signals aligned w/ 15m direction (the
  confluence finding showed aligned +51 vs against +16 — could lift win rate); (b) the realistic-
  entry gap (we give up 2 bars waiting for confirm) — test a pullback-entry after confirm to recover
  some; (c) accept +1.4/trade as the honest 5m base and move to live validation.

---

# 5m ENGINE FINALIZED — signal frequency check + hold-confirm locked — 2026-06-28
CK asked: how many signals/day with current settings? Checked before any HTF filter (HTF too early).

## SIGNAL FREQUENCY (TEMA-13, t_flat=0.35, hold-confirm=3):
  19.0 signals/day avg (median 20.5, range 4-27). SS360 fires ~10/day — we fire ~2x more.

## TIGHTENING TO ~10/DAY MAKES IT WORSE (important): pushing t_flat up / hold up to reach 10/day
  drops win to 38-40% and goes NEGATIVE. Our extra signals beyond their ~10 are NOT all noise —
  the ns-threshold lever cuts GOOD signals too (it selects for fast moves, not quality, and high-ns
  on 5m is exhaustion-prone). So fewer signals != better here. Current 19/day @ +1.4 is near the
  best the simple lever offers. To truly match their 10/day AND keep quality would need their
  internal selection logic (the grade state machine we couldn't decode), not a blunter threshold.

## CK DECISION: 19/day is fine if profitable — MOVE FORWARD. (Don't cripple win rate chasing their
  exact frequency.)

## ENGINE UPDATED: SS_5m_Engine_v3.pine signal block replaced with the VALIDATED hold-confirm:
  fresh commit must HOLD direction 3 bars before firing (signal emits at confirm bar, no look-ahead).
  Defaults: ss_len=13 (TEMA), t_flat=0.35, hold_confirm=3. Selective screen kept as optional OFF
  toggle (didn't beat plain hold-confirm on full range). This is the honest, full-range-validated
  5m signal engine. ~19 sigs/day, +1.4/trade, 43% win, 6/8 weeks positive.

## NEXT (when ready): HTF-aligned filter (confluence showed aligned +51 vs against +16) as a
  WIN-RATE lever — but CK paused this as premature. Live-validate the current engine first.

---

# ★★★ CRITICAL DATA CAVEAT — SS360 LEDGER IS AN UNDERCOUNT (CK) — 2026-06-28
CK flagged (had mentioned before): the 5m ledger does NOT capture ALL their signals. SS360 signals
DISAPPEAR from screen after a while, so CK captured as many as possible but MISSED SOME. The ledger
is a LOWER BOUND on their signal count, not a complete record.

## WHAT THIS INVALIDATES (every "we over-fire vs them" conclusion is now SUSPECT):
  - "v2 matches only 30% of their ledger, 65 EXTRA signals" — the "extra" likely includes REAL
    SS360 signals CK couldn't capture. Our over-firing was OVERSTATED.
  - "SS360 fires ~10/day" — this is a FLOOR (at-least-10 that were caught), NOT their true rate.
    Their actual frequency is higher, unknown.
  - "We fire 19/day = 2x too many" — SUSPECT. We may be CLOSE to their real frequency. Possibly
    not over-firing at all.
  - The "selective filter" / "refractory" work aimed at cutting our count toward ~10/day was
    chasing a FALSE target (their count isn't 10). Good thing plain hold-confirm beat it anyway.

## WHAT STILL HOLDS (did NOT depend on ledger completeness):
  - LINE FIT (TEMA-13): used their logged LINE VALUES (prices at events). RMSE-fit to captured
    points is valid regardless of missing events. Line is still correct.
  - 77% win on their CAPTURED signals: quality of caught signals; missing some doesn't change it.
  - HOLD-CONFIRM validation (+1.4/trade, 6/8 wks): scored on OUR engine vs OUR OHLC, NOT against
    their ledger. Fully independent of ledger completeness.
  - 70% direction agreement: measured at their captured events; a representative sample still
    informs direction even if incomplete.

## NET: the line, direction, signal-quality, and full-range signal validation ALL STAND. Only the
  "frequency comparison / over-firing" narrative is unreliable — and it now makes our 19/day MORE
  defensible (we may be near their true rate, not 2x over). Do NOT tune to hit "10/day" — that
  target was an artifact of incomplete capture.

---

# ★★★ CRITICAL FIX — Pine engine AM did NOT match Python validation — 2026-06-28
CK uploaded the REAL v3 (TEMA-13) TradingView run. Scored the engine's OWN rating column with
hold-confirm=3 on real OHLC: FAILED — 4.4/day, 37% win, -1.27/trade, fragile. DIVERGED hard from
the Python validation (+1.4/trade, 19/day).

## ROOT CAUSE: the Pine engine and my Python validation were DIFFERENT ENGINES.
  - Engine's own rating: NEUTRAL 87% of bars (the "sea of orange" CK saw on chart).
  - My Python AM: neutral 39%. Sign agreement only 52%.
  - The Pine code still had the OLD counter-neutralize ("price opposite line slope -> orange"),
    which on 5m fires nearly every bar (price oscillates across the slow line) -> collapses to orange.
  - I had validated a CLEAN Python AM (line-slope + flat band only) but never ported it to the Pine
    file. The line fix (TEMA-13) landed; the AM rewrite did NOT. Classic validation-vs-deliverable gap.

## FIX (CK chose Option A — make Pine match the validated Python AM):
  Replaced the Pine AM core: removed counter-neutralize + price-side gating + the pers/ns-jump/
  fast-release grade machinery. New AM = exactly the validated logic: dir = TEMA-13 slope sign gated
  by flat band (t_flat=0.35); grade = ns-based (3 / 5@ns≥0.5 / 7@ns≥1.0). Removed 7 orphaned inputs.

## VERIFIED (simulated new AM on the run's real SS line + OHLC):
  neutral 87% -> 39%. hold-confirm=3: 19.1/day, 43% win, +1.35/trade, +1130 total, 6/8 wks, +767
  no-best. NOW MATCHES the validation. Engine and Python are finally the same logic.

## LESSON: always score the REAL engine CSV output before claiming a validation describes the
  deliverable. The Python validation was sound but described a DIFFERENT engine than the Pine file.
  CK caught it by looking at the chart (too much orange). Fixed now — CK to re-run & confirm.

---

# ★★★ ORANGE STILL HIGH — t_flat was on the WRONG ns SCALE — 2026-06-28
After the AM rewrite CK STILL saw lots of orange. Root cause found: the flat band t_flat=0.35 was
tuned on my Python CLOSE-TO-CLOSE ATR proxy, but the real Pine engine uses ta.atr (TRUE RANGE),
which is ~1.6x larger -> real ns runs LOWER (median 0.21 vs my proxy). So t_flat=0.35 sat ABOVE
73% of the real ns values -> 73% orange. Same "different scale" trap as the counter-neutralize, now
in the flat band.

## RE-TUNED t_flat on the REAL engine ns (from the run CSV's own ns column):
  t_flat=0.05 -> 12% orange, +1.07/tr | 0.076 -> 18% (SS360-like) +0.53 | 0.122 -> 29% +0.99
  t_flat=0.15 -> 36% orange, +1.74/tr, 44%, 7/8 wks, +1029 no-best  <- BEST robust
  t_flat=0.20 -> 47% orange, +1.89/tr, 7/8 wks
  Real-ns ~19% orange (SS360 look) needs t_flat≈0.076 but costs edge (+0.53/tr).

## ACTION: set t_flat 0.35 -> 0.15. Fixes orange (73%->36%) AND improves profit (now +1.74/trade,
  7/8 weeks positive vs prior 6/8). Best of both. CK can drop to 0.076 if they want the SS360 look
  over max edge. Tooltip updated explaining the true-range ns scale.

## LESSON (again): validate on the REAL engine's own values, not a Python proxy. The proxy ATR put
  ns on a different scale and mis-set the flat band — exactly parallel to the counter-neutralize gap.
  The run CSV's own ns column is the ground truth; tune against THAT.

---

# ★★★ ORANGE FIX #2 — hold_bars was 1, must be 0 + SS360 chart calibration — 2026-06-28
CK uploaded a NEW run (t_flat=0.15 active) STILL showing lots of orange (56%), and the SS360 6/26
chart for visual comparison.

## DIAGNOSIS: the new run had 56% orange (not my predicted 36%). Found the cause: the engine ran
  hold_bars=1 (the "direction flip confirm" input default), my sim used 0. Replicating the Pine state
  machine: hold_bars=1 matches engine 99%. With the clean slope AM, hold_bars=1 makes a fresh
  direction wait 1 extra bar to confirm — and since slope sign oscillates, that parks the rating at
  orange ~20% MORE often: 56% orange AND negative (-1.05/trade, 2/8 wks). hold_bars=0 -> 36% orange,
  +1.71/trade, 44% win, 7/8 weeks. FIX: set hold_bars default 1 -> 0. (Signal noise-filtering is done
  by hold-confirm=3 on the SIGNAL, not by delaying the rating.)

## ★ SS360 CHART CALIBRATION (6/26, pixel-counted their colored line):
  red 39% / orange 30% / green 20% / blue 10% / pink+lemon ~0%.
  => SS360's real orange is ~30%. Our fixed engine = 36%. VERY CLOSE (was way off at 56%/87%).
  Their intermediate rungs (pink/lemon/magenta) barely appear — they hold trend color (red/green/blue)
  with orange only at turns. Our 3/5/7 split shows more intermediates but the ORANGE level now matches.

## STATUS: with ss_len=13, t_flat=0.15, hold_bars=0, hold_confirm=3 the engine is now:
  36% orange (≈ their 30%), 19.7/day, 44% win, +1.71/trade, 7/8 weeks positive, +1485 total.
  This is the validated, chart-calibrated 5m engine. CK to reload (hold_bars=0) & confirm visually.

## LESSON (3rd time): every input default must be checked against the REAL run, not assumed. The
  line (TEMA-13), the flat band scale (true-range ns), and now hold_bars all had sim-vs-engine gaps.
  The run CSV is ground truth. Score the real output before claiming the result.

---

# ★★★★★ LOOP CLOSED — real engine output MATCHES validation — 2026-06-28
After CK set hold_bars=0 in the TradingView settings panel (the code default of 0 was being
overridden by TV's saved value of 1), the real engine run finally aligns with the Python validation.

## VERIFIED FROM THE REAL ENGINE CSV (not simulation):
  - Orange: 36% (was 56% with hold_bars=1; SS360's real chart = ~30%). CLOSE to theirs.
  - hold_bars=0 confirmed active (99% match to sim).
  - REAL OUTPUT: 19.7 signals/day, 44% win, +1.71 pts/trade, +1485 total, 7/8 WEEKS POSITIVE,
    +1009 without best week. Scored on real OHLC with hold-confirm=3, no look-ahead.

## THE FINALIZED 5m ENGINE (all settings validated against the real run):
  - ss_len = 13 (TEMA, RMSE-fit to SS360's actual 5m line)
  - t_flat = 0.15 (re-tuned to the true-range ns scale; gives ~36% orange ≈ their 30%)
  - hold_bars = 0 (clean slope tracking; 1 over-neutralized to 56% orange + negative)
  - hold_confirm = 3 (signal fires only after a fresh commit holds 3 bars; no look-ahead)
  Engine and Python validation are now the SAME engine. +1.71/trade, 7/8 weeks, full 7wk OOS honest.

## THREE sim-vs-engine gaps were found & fixed this session, ALL caught by scoring the real CSV:
  (1) line type (TEMA-13 vs assumed 12), (2) flat-band ns scale (true-range vs close-to-close proxy),
  (3) hold_bars default (TV saved value). LESSON LOCKED: the real run CSV is ground truth; score it
  before claiming any result. TradingView persists saved input values — code defaults don't override
  them; CK must set them in the panel or remove/re-add the indicator.

## REMAINING (accepted): SS360 leans harder on strong colors (red/green/blue) vs our more frequent
  ±3 intermediates — the grade difference CK already accepted (same as 15m). Orange LEVEL now matches.

## NEXT (when ready): live-session validation; then the HTF-aligned filter (paused as premature) as
  a win-rate lever.

---

# 5m v4 — exit-marker toggle + REVERSE-SIGNAL EXIT analysis (last 3 days vs rest) — 2026-06-28
CK asked: add a separate exit-✕ toggle, and run reverse-signal exit split last-3-days vs rest.

## v4 CHANGE: added `show_exit` input (separate toggle for the orange ✕ return-to-neutral markers).
  Lets CK declutter the chart while keeping B/SC·S/BC entry labels. Exit logic itself UNCHANGED.
  File: SS_5m_Engine_v4.pine. All validated settings intact (ss_len=13, t_flat=0.15, hold_bars=0,
  hold_confirm=3).

## REVERSE-SIGNAL EXIT analysis (hold until OPPOSITE signal fires, real OHLC, run4 CSV):
  Last 3 days (Jun 24/25/26): 69 sigs, 34% win, -7.21/trade, -497, 2/3 pos days.
  Rest (41 days, May7-Jun23): 798 sigs, 38% win, +5.73/trade, +4571, 15/41 pos days.

## vs NEUTRAL/scalp-30-20 exit (current engine) same segments:
  Last 3 days: 37% win, -1.16/trade, -80.
  Rest (41 days): 44% win, +1.96/trade, +1565.

## READ: reverse-signal exit makes ~3x the TOTAL on the bulk 41 days (+4571 vs +1565; avgWin
  +101 vs +28 — lets winners run) BUT is more FRAGILE: lower win% (38 vs 44), bigger avgLoss
  (-54 vs -19), and gets punished harder on bad days (last 3 = -7.21/trade vs -1.16). Same pattern
  as 15m where reverse-signal-exit won total profit. The last 3 days (Jun24 chop + 25/26 reversals)
  were negative under BOTH exits — a rough stretch for this engine regardless of exit.
  NOT baked into the engine — it's a conscious total-vs-consistency tradeoff for CK to decide.

## NOTE: 44 days is a modest sample, and 41-vs-3 split makes the last-3 read low-confidence (just
  69 trades). The bulk-data reverse-signal edge (+5.73/trade over 798 trades) is the more reliable
  signal; the last-3 weakness needs more bad-day samples to confirm it's structural not variance.

---

# ★★★★★ v4 — STRUCTURE GRADE (the 15m lesson, finally carried to 5m) — 2026-06-28
CK observed: on OUR Pine chart the afternoon fall is PINK(-3)+orange; on SS360 it's MAGENTA(-5)
all the way down. CK also rightly noted: had I read the 15m notes fully, we'd have found this
earlier instead of burning cycles on the ns-grade. And: stop relying on Python sim — test on the
real Pine CSV (CK provides each run).

## ROOT CAUSE (confirmed on the REAL Pine rating column, 6/26 fall): our grade was ns-based
  (3 / 5@ns>=0.5 / 7@ns>=1.0). During a steady decline, median ns ~0.24, only 2 bars hit ns>=0.5,
  so the grade was mechanically STUCK at -3 (pink). 17 pink bars, 1 magenta. SS360 holds -5 magenta
  because PRICE IS BELOW LR the whole time — a STRUCTURAL -5, independent of per-bar speed.

## THE 15m NOTES SAID THIS EXPLICITLY (transcript 2026-06-18-05-09-45, lines 1309 & 2207):
  "magnitude is NOT momentum. The 3/5/7 is purely STRUCTURAL — where price sits vs LR and V."
  "You only reach -7 when momentum is strong... a steady grind sits in magenta instead of red.
   SS360's rule: magnitude = STRUCTURE, sign = AM." I built 5m grade on ns anyway. My miss.

## FIX in v4: grade = STRUCTURE (the proven 15m model):
  above_lr = SS > LR  (off the SMOOTH line, not close — avoids flicker, per 15m note)
  ±3 = wrong side of LR | ±5 = right side of LR | ±7 = full stack SS<LR<V or SS>LR>V
  Verified on real CSV (SS/LR/V columns): 6/26 afternoon -> 12 magenta + 6 red (was 1 magenta).
  Matches SS360's "magenta all the way down". Full data: orange drops to ~17%, full color ladder
  used (-7/-5/-3/+3/+5/+7 all populated), vs ns-grade stuck mostly at ±3.

## TRADEOFF (honest): structure grade shifts the SIGNALS (sign agreement 80% vs ns-grade), so the
  old hold-confirm tuning gives +0.71/trade on the new grade (vs +1.71 on ns-grade). The COLOR is
  now right but the signal layer needs RE-VALIDATION on the structure grade. This is expected — the
  grade fix is correct (15m-proven); the signal settings on top of it are not yet re-tuned.

## v4 ALSO ADDS: separate `show_exit` toggle (declutter the ✕ exit marks independently of entries).

## PROCESS CHANGES (CK directives, adopted):
  1. READ THE 15m NOTES FULLY before building 5m equivalents — the answers are often already there.
  2. STOP over-relying on Python sim. We're at live-Pine-test stage. CK provides the run CSV each
     time; verify against THAT (the real rating/LR/V columns), not a Python proxy. Three gaps this
     session (line type, ns scale, hold_bars) all came from trusting sim over the real run.

## NEXT: CK runs v4 (structure grade), sends CSV. Verify: (a) colors match SS360 (magenta down trends),
  (b) re-validate/re-tune the signal layer on the structure grade to recover the edge. Keep v3
  (ns-grade, +1.71/trade) as the known-good fallback until v4's signals are re-validated.

---

# ★★★★★ v4 VERIFIED ON REAL RUN — structure grade works, signals INTACT — 2026-06-28
CK ran v4, sent CSV. Verified against the REAL engine output (not Python sim):

## STRUCTURE GRADE ACTIVE & MATCHES SS360:
  6/26 afternoon fall: 10 MAGENTA + 5 RED (15/36 bars) = "magenta all the way down" like SS360.
  Full data color ladder now rich: BLUE 16%, GREEN 6%, lemon 9%, orange 36%, PINK 9%, MAGENTA 7%,
  RED 13% — full ±3/±5/±7 spread (was stuck mostly at ±3 with ns-grade).

## ★ SIGNALS DID NOT DEGRADE (the key surprise): real v4 = 19.7/day, 44% win, +1.71/trade, +1485,
  7/8 weeks, +1009 no-best — IDENTICAL to v3. My Python sim had predicted +0.71 (a drop), but the
  sim computed LR/V from close-to-close approximations and got different structure boundaries. The
  REAL engine uses its actual LR/V and the signals land in the same place. THE PYTHON SIM WAS WRONG
  AGAIN — the real CSV is the truth. (Exactly the lesson CK drove home this session.)

## RESULT: v4 gives BOTH — SS360-matching colors (magenta down trends, full ladder) AND the full
  +1.71/trade / 7-of-8-week edge intact. The tradeoff I feared (color-vs-profit) was a Python-sim
  artifact; it does not exist on the real engine.

## v4 IS NOW THE PRIMARY 5m ENGINE: ss_len=13 (TEMA), t_flat=0.15, hold_bars=0, hold_confirm=3,
  STRUCTURE grade (±3 wrong side LR / ±5 right side / ±7 full stack, above_lr off SS), show_exit
  toggle. Colors match SS360, edge intact, validated on the real run. v3 retained as ns-grade ref.

## NEXT (when ready): live-session validation; HTF-aligned filter (paused) as a win-rate lever.

---

# v4 — CSV LOGGING REWORK: single source of truth (paint_rating) — 2026-06-28
CK: markets closed, so every chart bar is a CLOSED bar; the CSV must match the chart exactly.
CK saw chart-vs-CSV rating discrepancies. Investigated and fixed.

## ROOT CAUSE: the chart had THREE+ rating references that could visually conflict:
  - line COLOR used paint_rating
  - a PERMANENT number label used raw `rating` (printed on change)
  - a MOVING "current value" label (curlb) used paint_rating, redrawn every bar
  - the CSV logged raw `rating`
  On closed bars these are mathematically equal (barstate.isconfirmed -> paint_rating := rating),
  but the duplicate labels overlapped (looked like doubled/conflicting numbers) and the CSV logged
  a DIFFERENT variable than the line painted, so they were not a guaranteed single source.

## FIX: everything now reads paint_rating (the painted value):
  - CSV logs paint_rating (was `rating`)
  - ONE number label, reads paint_rating (removed the duplicate curlb)
  - exit ✕ marker reads paint_rating
  - line color already used paint_rating
  Chart color + chart number + exit marks + CSV are now the SAME value. Cannot disagree on closed bars.

## NOT TOUCHED: signal (B/SC entry) logic still computes off `rating`. Correct & intentional —
  signals fire only on barstate.isconfirmed (closed bars) where rating==paint_rating exactly, so
  they're already consistent; rewiring risks the validated +1.71/trade edge for zero benefit.

## NOTE: if a discrepancy persists after this, the cause is SAVED CHART INPUTS differing from the
  run (ss_len/v_len/ref_type) — chart computes different SS/LR/V. Fix: re-add indicator fresh.

## NEXT: CK re-runs v4, confirms chart number == CSV rating bar-for-bar.

---

# SESSION WRAP — 2026-06-28 (v4 = current primary 5m engine)
## STATE: v4 is solid and validated. The chart-vs-log discrepancy CK flagged traced back to CK
  looking at 3:45PM vs 1:15PM (not a bug). The log IS correct: internally consistent with its own
  SS/LR/V (0/6308 mismatches), and after the logging rework it matches the chart on closed bars.

## WHAT v4 IS (the validated primary 5m engine):
  ss_type=TEMA, ss_len=13, lr_len=25, v_len=50, ref_type=EMA, t_flat=0.15, hold_bars=0,
  hold_confirm=3. STRUCTURE grade (±3 wrong side LR / ±5 right side / ±7 full stack SS<LR<V),
  above_lr off SS. show_exit toggle. CSV logs paint_rating (single source of truth w/ chart).
  Real run: 19.7/day, 44% win, +1.71/trade, +1485, 7/8 wks, 36% orange, full color ladder.

## OPEN THREADS / READY-TO-DO NEXT SESSION (all tested, parked for CK decision):
  1. ±7 FAST-FIRE: fire signal immediately (1 bar) ONLY on a clean ±7 full-stack commit, keep
     3-bar confirm on ±5/±3. Tested on real run: 21.4/day, 44%, +1.62/trade, +1523, 8/8 WEEKS
     (vs 7/8). Recovers the "late signal lost points" on the strongest moves WITHOUT the fakeout
     bleed that killed broad fast-fire. The one fast-entry variant that survives full-data. Could
     add as a toggle (default on). NOT yet built into v4.
  2. GRADE TIMING (the -7-vs-5 question, RESOLVED): on 6/26's 1:15 fall the log grades -3 (13:15,
     SS still 48pt ABOVE LR) -> -5 (13:45, SS below LR) -> -7 (14:50, full stack LR<V forms). This
     is CORRECT and matches SS360's progressive breakdown. The earlier "premature -7" was a stale-
     display artifact, gone now. If CK reads SS360 as -5 at the very top vs our -3, that's a minor
     -3/-5 top-of-turn boundary calibration — needs their ledger number at that bar.
  3. COLOR PERSISTENCE (flicker): the raw rating flickers to orange on slow single bars mid-trend
     (943 such gaps, 52% are 1-bar). True persistence (hold last color through up to N orange bars,
     release on long stall or sign flip) bridges them and would make logged runs continuous like the
     chart's visual hold. hold_n=1 matches the chart. WOULD CHANGE ~943 bars' rating -> shifts signal
     crosses -> MUST re-validate edge before adopting. Parked.
  4. RTH/Globex split (5m): edge is in GLOBEX (+2.75/trade) not RTH (-0.98/trade). Possible
     Globex-only session filter — not yet built/validated.
  5. Reverse-signal exit: ~3x total on bulk (+4571 vs +1565) but fragile on chop days. Not baked in.
  6. HTF-aligned filter (paused as premature) — win-rate lever, AFTER live validation.

## STANDING: 15m PRODUCT FROZEN. 5m = research sandbox, nothing live until v9-level scrutiny + CK
  decides. Process locked this session: READ 15m NOTES FULLY before building 5m logic; SCORE THE
  REAL ENGINE CSV not a Python proxy (CK provides the run CSV each time).

---

# v4 — THREE CHANGES (immediate fire, bar-close alerts, HTF dashboard) — 2026-06-28
CK requested three changes for live trading.

## 1. IMMEDIATE SIGNAL FIRE (removed 3-bar wait): hold_confirm default 3 -> 1.
  Rationale (CK): in current market conditions the 3-bar wait loses the momentum after a trend/color
  change — the move is gone by the time it fires. hold_confirm=1 fires the moment the color locks at
  bar close (no look-ahead). TRADEOFF SHOWN EXPLICITLY: on the 7-week backtest immediate=48.5/day,
  40% win, -0.46/trade, 3/8 wks (vs hold-3: 19.7/day, 44%, +1.71, 7/8). The backtest scores every
  fire mechanically; CK trades live with discretion (takes the clean early fire, skips obvious chop)
  — a thing the backtest can't capture. Kept as an INPUT (minval 1); 3 still available, and the ±7
  fast-fire option (8/8 wks) is the middle ground if CK wants it later.

## 2. ALERTS BAR-CLOSE ONLY: the three alertcondition() triggers were firing intrabar on every
  transition inside a forming candle (alertcondition fires at whatever freq the user picks in the
  dialog; the condition evaluated intrabar). FIX: gated all three with `barstate.isconfirmed` so they
  can only be true on a CLOSED bar regardless of dialog frequency. The alert() calls already used
  freq_once_per_bar_close (were fine). All alert text now uses paint_rating.

## 3. HTF DASHBOARD (15m on the 5m chart): CK wants the locked 15m rating visible while trading 5m,
  like SS360 prints its higher-TF wave rating. Built a small TABLE (3x3) showing 5m + 15m rating side
  by side, each in its color, with state word (STRONG BULL/BULL/EARLY BULL/NEUTRAL/.../STRONG BEAR).
  Inputs: show_dash, dash_pos (5 positions), htf_tf (15/30/60). The HTF rating is a STATELESS function
  (htf_rating: sign=line slope vs flat band, magnitude=structure ±3/±5/±7) pulled via request.security
  with lookahead=barmerge.lookahead_off (NO repaint/look-ahead, confirmed HTF bars only). NOTE: the
  dashboard shows the HTF RATING/COLOR (the context SS360 prints), NOT the 15m hold-confirm signal
  layer — it's for "trade the 5m WITH the 15m" alignment context. Uses ss_len=12 for 15m per locked v9.

## STATUS: v4 now ~465 lines. All prior v4 features intact (TEMA-13, structure grade, show_exit,
  paint_rating single-source logging). CK to run and confirm: immediate fire prints on color-lock,
  alerts only on close, dashboard shows 5m+15m. Backtest edge of the immediate-fire is negative on
  history — this is a LIVE-DISCRETION config by CK's explicit choice, not a validated-edge change.

---

# v9-15m FROZEN — RE-LOCKED with bar-close-only alerts — 2026-06-28
CK: open the locked v9-15m, update alerts to no-intrabar, re-lock.
FINDING: the frozen v9 had NO alerts at all (display + log only). So this ADDED a clean alert set
rather than fixing existing ones.
ADDED (all gated by barstate.isconfirmed, so no intrabar firing regardless of dialog frequency):
  - alertcondition: signal fired (bsc/sbc), rating/color change
  - alert() freq_once_per_bar_close: signal, rating change — text uses paint_rating
ENGINE LOGIC UNTOUCHED: line/grade/direction/signals/CSV all unchanged. Header notes RE-LOCKED
2026-06-28, alerts-only. v9-15m remains the FROZEN product — engine logic not to be modified.

---

# CORRECTION — alert fix went to the WRONG file, fixed — 2026-06-28
CK clarified: the alert change should go to SS_Trinity_Engine_v9_15m_display (the [15m] version with
the changed color palette + tweaks), NOT SS_Trinity_Engine_v9_FROZEN. CK had said "locked v9" by mistake.

ACTIONS:
  - REVERTED SS_Trinity_Engine_v9_FROZEN.pine to original (removed the alerts + re-lock header I'd
    mistakenly added). Back to 291 lines, no alerts, original header. FROZEN is untouched again.
  - FIXED SS_Trinity_Engine_v9_15m_display.pine (the correct file): it already HAD alerts, but the two
    alertcondition() calls were UNGATED (could fire intrabar). Gated both with barstate.isconfirmed.
    The alert() calls were already gated + freq_once_per_bar_close. Engine logic / color palette / all
    other display tweaks UNTOUCHED — alerts-only change.

FILE ROLES (clarified): SS_Trinity_Engine_v9_FROZEN = the locked product, do not modify.
  SS_Trinity_Engine_v9_15m_display = the [15m] display variant (changed palette + tweaks) — this is
  the one CK runs/iterates on for 15m display.

---

# ★ STANDARD: THREE-SESSION TRADE ANALYSIS MODEL (adopted 2026-06-29)
Going forward, ALL Pine-log trade analysis uses three sessions (ET), per CK directive:
  - SECTION A = EVENING: 6:00PM – 11:59PM
  - SECTION B = OVERNIGHT: 12:00AM – 9:30AM
  - SECTION C = RTH: 9:30AM onwards
Each trade analyzed with entry/exit OHLC, MFE (max favorable excursion), MAE (max adverse), bars
held, and a loss diagnosis (gave-back-gains / wrong-from-start / chop-scratch / fast-failure).
Deliverables: per-trade markdown + session summary table. Files: *_SESSION_trade_analysis_*.md.

# ★ SUNDAY WEEK-OPENING NOTE (CK insight, 2026-06-29)
6/28 evening = SUNDAY week opening (Globex reopen). CK: Sunday openings USUALLY trend one-side for
some time, so evening trades can benefit from a clean directional open. This is REGIME CONTEXT for
Section A on a Sunday — a Sunday evening's profit (e.g. 6/28 v9 +72, v4-5m-on-5m +221) may be a
Sunday-open trend artifact, NOT a general "evening is good" rule. Flag Sunday-evening sessions
separately in future analysis; don't generalize Sunday-open trend behavior to weekday evenings.

# ★ CROSS-TIMEFRAME SESSION FINDINGS (6/28 18:00 – 6/29 17:00):
## 5m TF (v4-5m exit-symbol, 54 trades, +126):
  - A Evening +221 (10tr, 60%) — the profit engine (Sunday-open trend)
  - B Overnight -37 (24tr, 33%) — death by chop, 12 scratch losers, ns 0.15-0.25, no trend
  - C RTH -58 (20tr, 35%) — caught morning trend (10:00-13:25) but WHIPSAWED at the top
    (13:30-16:05 = 9 of 10 losers, fading a stall that wasn't turning) + 9:45 open-spike BUY (MAE-207)
  - #1 LEAK: 20 losers gave back +703 pts of unrealized profit (exit-symbol exits too late on 5m).
## 15m TF (same window):
  - v9-15m: 5 trades, +232. A +72, B -65, C +225 (one held BUY = all the profit). RTH CLEAN, 0 losses.
  - v4-5m-on-15m: 16 trades, +65. A -172 (evening reversal WHIPSAW), B -94 (chop churn), C +332.
    RTH nearly clean (4tr, 1 small loss -22). 
  - 15m give-back leak much smaller than 5m (coarser bars filter micro-noise).
## KEY INSIGHT: persistence (v9) vs structure (v4) on the evening reversal —
  v9's slow grade was an ADVANTAGE on the choppy 6/28 evening (didn't whipsaw, +72) — the MIRROR
  IMAGE of 6/28 afternoon where the same slowness was a LIABILITY on a clean fast reversal. 
  Persistence helps in chop-with-no-trend; hurts on a sharp clean reversal. Structure is the opposite.
## RTH REGIME 6/29: two-phase — sharp drop (29795->29274, 9:45-10:15) then sustained grind up
  (29330->30070). 796pt range, 44% trendiness. Both engines caught the up-grind (the profit).

---

# ★ RTH FIX PROPOSALS (6/29 analysis, 2026-06-29) — NOT yet built, need multi-session validation
Worked on RTH losses for v4-5m (native 5m TF) and v9-15m separately. Both verified non-destructive
(don't clip the profitable trades). Full doc: RTH_FIX_PROPOSALS_v4-5m_and_v9-15m.md.

## 6/29 RTH regime: two phases — 09:30-13:30 TREND (+367, 50% trendiness, both engines earned here),
  13:30-17:00 STALL (-16 net, 17% trendiness, tight 96pt band at highs — ALL the damage here).

## v4-5m fix — CHOP FILTER (gates ENTRIES):
  Problem: 10 of 13 RTH losses entered AFTER 13:30, all chase/fade in upper 92-99% of range, -156pts.
  Fix: skip entry when 8-bar trendiness (|net|/|range|) < 0.40.
  VERIFIED: RTH -58 -> +60. Cuts 6 losers (-117). Skips ZERO winners (all wins had trend8>=0.51).

## v9-15m fix — STALL LOCK-IN (tightens EXIT):
  v9 RTH had ZERO losses (one BUY +225). But peaked +284 (14:30) -> gave back to +225 by 16:45 exit
  in the top chop. v9 held for full opposite signal (-3 at 16:45), no stall lock-in.
  Fix: exit when grade fades to <=+3 (long) AND 8-bar trendiness < 0.35 (stall confirmed).
  VERIFIED: would exit 15:15 @+255 vs +225. +30 better. Triggers at trend8=0.15 (well after trend
  ended), so does NOT cut the run-up short.

## SHARED MECHANISM: 8-bar trendiness detector distinguishes trend (trade/hold) vs stall (skip/lock).
  v4-5m: gates entries. v9-15m: tightens exit. Neither touches genuine-trend behavior.

## CAVEAT: tuned on ONE RTH session (6/29). Thresholds 0.40/0.35 + 8-bar window are STARTING POINTS,
  need multi-RTH-session validation before building into engines. Mechanism sound, numbers may tune.
## NEXT: gather more RTH sessions, confirm the trendiness gate holds, THEN build into engines.

---

# v4-5m CHOP-FILTER validated on REAL run (KCRT chop-filter log) — 6/29 RTH — 2026-06-29
CK ran the new chop-filter version on TV, sent log. Verified: 6/29 RTH ratings IDENTICAL between the
original v4 run and this chop run (0 diffs in window) — confirms the filter does NOT touch the rating,
only gates signals (as designed). The scattered 200 full-history diffs are just EMA warmup (different
data start date), irrelevant.

## RESULT (6/29 RTH, real run, exit-symbol, next-bar-open fills):
  ORIGINAL: 20 trades, -57 pts, 7W
  NEW (chop, 0.40/8-bar): 13 trades, +27 pts, 6W
  Filtered out: 7 trades — 6 LOSERS (-133 pts, intended) + 1 WINNER (+50, collateral: the 09:55 SELL).
  Net effect: -57 -> +27 = +84 RTH improvement.

## The 1 collateral winner: 09:55 SELL +50 (trendiness 0.34, just under the 0.40 gate). This was the
  RTH-open drop entry — a real move that the gate caught because the 8-bar window still included the
  pre-drop chop. A slightly lower threshold (0.30-0.35) would keep it but also let back 1-2 losers.
  Trade-off to tune across more sessions.

## Excel deliverable: v4-5m_RTH_chopfilter_comparison_0629.xlsx — every original RTH trade flagged
  KEPT/FILTERED with trendiness + note. CK to review which trades affected.

## STILL: one session. Mechanism validated on the real run (filter works as designed, ratings intact).
  Numbers (0.40 threshold) need more RTH sessions before locking.

---

# ★ SUNDAY-OPEN EXEMPTION finding + TRADER'S-VERDICT reframe — 2026-06-29
## Sunday-evening exemption (worth noting, NOT building yet — needs more Sundays):
  Chop filter HURT the Sunday evening (cut the +223 19:50 SELL, a gradual-build Sunday-open trend
  whose 8-bar trendiness read 0.10). Range-spike rule does NOT save it (that bar was normal-sized,
  0.99x ATR) — only saves the 9:55 morning drop (2.54x ATR spike). Session-exemption DOES save it.
  RESULT on 6/29-window (exit-symbol, full window):
    Original (no filter): A+182 B-95 C-57 = +30
    Chop all sessions:    A+45  B-25 C+27 = +46
    Chop + exempt SUN 18:00-22:00: A+272 B-25 C+27 = +273  <-- best
    Chop + exempt SUN 18:00-23:59: A+182 B-25 C+27 = +183
  WHY 18-22 is the sweet spot: captures the Sunday-open trend (18-22) but STILL filters the late-
  evening chop (22:00+ had losers -36/-53). Rule would be "exempt 18:00-22:00 ON SUNDAYS ONLY".
  CAVEAT: one Sunday. 22:00 cutoff + Sunday-only scoping need multi-Sunday validation. NOT built.

## ★★ TRADER'S-VERDICT REFRAME (CK insight, important):
  CK: "you worked all RTH and made only ~30 points, wouldn't it be better not to trade at all?"
  CORRECT. ~+30 GROSS over a full RTH is not worth trading: 13-20 round-turns of commission (3-5 pts)
  + slippage (several pts) likely makes it NET flat-to-negative. Sitting out = certain zero, no cost,
  no risk — better than grinding 20 marginal trades for ~+30 gross.
  THE DEEPER GAP: the engine has NO concept of "this session isn't worth trading." It fires
  continuously regardless of whether there's enough opportunity to clear costs. A discretionary
  trader's most valuable skill is NOT trading — recognizing a low-quality session and standing aside.
  The chop filter cuts INDIVIDUAL trades, but the bigger question is "is this session worth engaging
  at all?" 6/29 RTH: 796pt range but 44% trendiness, real money in 2 bursts (10:00 drop, 11-13:00
  grind), rest chop. Verdict: a MARGINAL-TO-SKIP session.
  FUTURE DIRECTION (not built): a session-quality gate — measure session trendiness/opportunity early,
  and if below a bar, the engine signals "low-quality, trade small or stand aside" rather than grinding
  every marginal signal. This is the trader-skill the engine lacks. Reframes filtering from
  per-trade to per-session.

---

# ★★ SESSION-QUALITY GATING RESEARCH (full analyst readout) — 2026-06-29
CK asked for thorough research on session-framing metrics (analyst/PM level). Doc:
SESSION_QUALITY_research_readout.md. Tested 7 early-session (first-hour) metrics vs RTH PnL across
36 v4-5m RTH sessions (mean -33/day, only 13/36 profitable — engine RTH is a net loser, why the
gate matters).

## KEY FINDINGS:
  - NO single metric is a strong predictor. Best correlation: first-hr RATING FLIPS r=-0.22. 
  - Counterintuitive: first-hr TRENDINESS, STRONG-GRADE%, OVERNIGHT RANGE all r≈0 (NO signal). A
    directional first hour does NOT predict a directional day — same early signature precedes both
    good and bad days. This is the fundamental limit: regime isn't determined by 10:30.
  - CHOP has a clearer early signature than TREND. Can't spot good days early; CAN spot whippy days.
  - WORKING RULE (Rule D): skip RTH if first hour has >=3 rating flips OR >30% orange bars.
    Trade-all 36d = -1198. Rule D: trade 18d = +690 (+38/day, 50% win), skip 18 avoiding -1888.
  - OUT-OF-SAMPLE ROBUST: both halves positive (+66 first, +624 second). Simpler flips>=2 was
    OVERFIT (+441 then -498). The specific composite is what holds.

## METRIC TIERS for a session-quality score (first 30-60min):
  Tier 1 (proven): rating-flip count (>=3=chop), orange/neutral fraction (>30%=stalling).
  Tier 2 (weak/confirmation): first-hr ATR vs median, first-hr range (only >550=spike/revert).
  Tier 3 (NO value, drop): trendiness, strong-grade%, overnight range.

## CAVEATS: 36 days small; rule trades only HALF the days (conservative); tuned to v4-5m's chop
  failure mode (v9 needs OPPOSITE metrics - it bleeds on reversals not chop); "first hour" leaks
  2-3 early trades; predicts ENGINE pnl not market (engine-specific regime detection).

## RECOMMENDED NEXT: instrument engine to LOG first-hour metrics (flips, orange%, ATR-vs-median,
  range) every session WITHOUT acting, build to 100+ days, re-test Rule D OOS on larger set. Sound
  mechanism, sample is the gap. Defensive tool (keep engine out of bleed regimes), NOT an alpha source.

---

# ★★ SESSION-QUALITY ON 15m (more data, ~132 days) — 2026-06-29
CK asked to run the session-quality test on 15m (more data). Doc: SESSION_QUALITY_15m_readout.md.

## TEST REDESIGN: v9 barely trades in RTH (68/106 RTH windows had ZERO trades — persistence grade too
  slow for a 26-bar RTH). So used WHOLE-DAY as the unit (where 15m has trade count).
  Whole-day baselines: v9 82d, 3.2 tr/day, 50% profitable, +402. v4-on-15m 132d, 12.8 tr/day, 51%, -350.

## ★ HEADLINE: the two engines have OPPOSITE predictive signatures (confirms failure-mode theory):
  v9 best predictor = first-hr TRENDINESS r=+0.21 (WANTS trend, holds it, bleeds on reversals).
  v4 best predictor = first-hr FLIPS r=-0.22 (HURT by chop, churns — same as 5m).
  => A single universal session gate is WRONG for one of them. Gates must be ENGINE-SPECIFIC.

## VALIDATED (out-of-sample):
  ✅ v9-15m "skip day if first-hr trendiness < 0.30": baseline +440(81d) -> trade 51d +2648 (+52/d,
     58% win), skip 30 avoiding -2208. OOS both halves POSITIVE (+74/d, +23/d). ROBUST. Strongest
     session-quality signal found across EITHER timeframe.
  ❌ v9 trend-gate <0.20: overfit (tighter is fragile, 0.30 is the robust threshold).
  ❌ v4-on-15m chop-gate (flips>=3 OR orange>30%): the 5m rule does NOT transfer to 15m. +31/d first
     half, -24/d second half. Coarser 15m bars already filter micro-chop. DO NOT port 5m rule to 15m.
     v4 session-quality signal lives on its NATIVE 5m TF.

## CAVEAT: v9 gate skips 30/81 days (63% traded); +2648 leans on first half (+74 vs +23); robust claim
  is "both halves positive" not the exact magnitude. 132 days, one regime.

## NEXT: for v9-15m, log first-hr trendiness forward, confirm 0.30 gate holds on new OOS days. For
  v4, keep session-quality on native 5m. Log first, validate forward, gate only after it holds.

---

# ★★ PERSISTENT SESSION-QUALITY MODEL built (consolidates all logs) — 2026-06-29
CK: build a persistent model from all logs collected so far; new data appends without losing prior.
DELIVERED a 3-file system in outputs:
  - SESSION_MODEL_DATA.csv = master store, 1 row per (engine,date), 309 sessions (132 v9_15m, 132
    v4_15m, 45 v4_5m). Holds first-hr metrics + 3-session + whole-day outcomes. SYSTEM OF RECORD.
  - build_session_model.py = parses any log, appends/updates sessions (no data loss).
  - session_model_analyze.py = reads master, prints per-engine baseline + correlations + gates with
    AUTOMATIC OOS robustness labels. Re-runs as data grows.
  - SESSION_MODEL_README.md = how to add data + current state.

## CONSOLIDATED-DATA FINDINGS (more honest than the piecemeal slices):
  ✅ v9_15m trend-gate ROBUST at 0.25 AND 0.30 (OOS both halves +). 0.30: baseline +1472 -> trade
     72d +3464 (+48/d, 55%), skip 37 avoiding -1992. THE ONE VALIDATED GATE. (Earlier I said 0.30
     from the 81-day cut; full 109-day set confirms 0.25-0.30 robust, 0.20/0.35 fragile.)
  ❌ v4_15m: all gates FRAGILE on full data (chop signature weak on coarse 15m bars).
  ❌ v4_5m: chop-gate looked good on 36-day slice but FRAGILE on consolidated set (OOS -9/+183, one
     half drives it). 45 days still thin. Keep collecting.
  CONFIRMED opposite signatures: v9 wants trend (+corr), v4 hurt by chop (-corr). Gates engine-specific.

## WORKFLOW for new logs: drop log -> add to sources list in build_session_model.py -> run build ->
  run analyze. Appends, never loses. Gate becomes trustworthy only when it stays ROBUST as sample grows.
## STATUS: only v9 trend-gate validated. v4 gates need more data. Log first, validate forward, gate
  only after it holds.

---

# ★★ GC TREND-LOCK v1 BUILT (Pine strategy) — 2026-06-30
CK: build v1 for gold; everything stays as v9-15m, only the LOGIC changes; CK will supply pine logs.
DELIVERED: GC_TrendLock_v1.pine (305 lines).

## STRUCTURE:
  - ENGINE = v9 Trinity, INHERITED VERBATIM. Engine core (HELPERS -> `rating = held_sign * mag`)
    is BYTE-IDENTICAL to SS_Trinity_Engine_v9_FROZEN. Nothing in the engine changed.
  - Only NEW code = the trade mechanism appended after the engine: indicator()->strategy().
  - It's a strategy() (backtestable in TV), GC/15m, $100/pt, commission $2.50/order, slippage 2 ticks.

## MECHANISM (the validated Gold Trend-Lock, pinned params):
  - BASE = reverse (strategy.entry auto-reverses on opposite gated signal).
  - GATE = enter/flip ONLY when |rating|>=strong_min(3) AND 8-bar trendiness>=trend_min(0.30).
  - gate-fail opposite signal -> strategy.close (book the trade) but no re-entry until gated signal.
  - VERIFIED: Pine logic faithfully reproduces the validated Python backtest (always-close-on-opposite,
    gated re-entry, gated entry-from-flat). Match confirmed.
  - Params strong_min=3, trend_min=0.30, trend_lb=8 are PINNED (set on train, held on holdout). Exposed
    as inputs but DO NOT re-optimize — stability is the edge.
  - Optional session filter (off by default; backtest traded all sessions).

## VALIDATION BEHIND IT (from GC_MECHANISM_build_result.md): GC 15m, 133 days, 60/40 train/test.
  Holdout (54 unseen days): +94 net (+0.47/tr), BEAT baseline reverse which LOST -87 on same days.
  Both holdout halves positive. Monthly net +5 of 6. Low win-rate (36%), high payoff (2.5:1) trend system.
  STATUS: validated-in-backtest, NOT forward-validated. CK to paper-trade forward + supply logs.

## NEXT: CK supplies forward GC pine logs from running v1 -> assistant validates the holdout result
  holds on brand-new data before any live capital. Size small (low win-rate trend style).

---

# ★★★ GC TREND-LOCK v1 — TV BACKTEST VALIDATED — 2026-06-30
CK ran v1 in TradingView strategy tester on GC (COMEX GC1!) 15m and supplied the export.
Doc: GC_TrendLock_v1_TV_validation.md. 992 trades, Aug 1 2025 - Jun 29 2026 (11 months), real
fills incl commission ($2.50/order) + 2-tick slippage. $100/pt.

## RESULT (the validation we were after):
  Net +$197,272 on 1 lot. 34% win, payoff 2.44, +1.99 pts/trade. Max DD $27,260. Return/DD = 7.24.
  Avg hold 3.0 hrs. Exit mix: 449 flips + 543 gate-fail-closes (the gate-fail-close is doing real
  work pulling out when trend quality drops).

## PASSED EVERY ROBUSTNESS CHECK:
  - TRUE OOS (Aug-Dec 2025, data I NEVER saw — my build was Jan-Jun 2026): +$52,358, 446 trades, same
    34% win. Edge real, not fit to my sample. STRONGEST confirmation.
  - 10/11 months positive (only May 2026 -$5.7k). Strip best 2 months (Jan+Mar $105k) -> other 9 still
    avg +$10.2k/mo. Not lucky-month-dependent.
  - Recent (Apr-Jun 2026): +$24k, still green, not deteriorating.
  - Concentration HEALTHY: top5 = 53%, but WITHOUT top5 still +$92,854 (vs the old fragile GC reverse
    which was NEGATIVE without home-runs). Edge distributed, big trades are upside not life-support.
  - Python holdout predicted +0.47/tr; TV actual +1.99/tr — TV BETTER (2025 had strong gold trends).
  - Engine UNCHANGED (pure v9 Trinity); only trade logic new.

## STATUS: validated in Python holdout AND independent 11-month TV backtest incl true OOS. The
  profitable mechanism the project was searching for — and it's on GOLD, not NQ.
## REMAINING CAVEATS before live: still a backtest (TV fills optimistic on fast moves); one (trending)
  regime; low win-rate needs discipline (take EVERY signal); $27k DD on 1 lot must be survivable.
## NEXT: forward paper-trade v1 on GC 15m, collect forward logs, confirm on brand-new bars. If holds,
  deploy at min size (MGC micro $10/pt) to validate live execution before scaling. DO NOT touch params
  (strong=3, trend=0.30, reverse-base) — held across holdout + 11mo TV + true OOS.

---

# GC v1 INDICATOR fixes + 5m TEST — 2026-06-30
## FIXED in GC_TrendLock_v1_INDICATOR.pine (CK reported logs not generating, price not printing):
  - ROOT CAUSE: when I built the indicator I rebuilt the display/mechanism block fresh and did NOT
    carry over the engine's bar-by-bar CSV logger or the signal-price drawing (they lived at the end
    of frozen v9, past where I cut the engine core). So do_log/price_at_sig were DEAD toggles.
  - FIX 1: added FULL bar-by-bar engine CSV log (same 13-col format as v9 export), DEFAULT ON.
  - FIX 2: added signal-only CSV log (LONG/SHORT/EXIT + price), DEFAULT ON.
  - FIX 3: LONG/SHORT/EXIT markers now use label.new with PRICE printed on them (was plotshape, can't
    show dynamic price text). Toggle "Print entry price on labels" default ON.
  - Removed the dead price_at_sig/do_log inputs. Engine core still byte-identical to frozen v9.
  - View logs: Pine Editor -> run -> bottom "Pine logs" panel.

## ★ 5m TEST (CK asked if logic tested on 5m — it was NOT; now tested):
  Same exact v1 mechanism (reverse-base, strong>=3, trend>=0.30) on GC 5m (46 days, $100/pt):
    799 trades, gross -92, NET -251 (-0.31/tr), 32% win, OOS MIXED (-197/+105), top5 -335%.
  => v1 mechanism is a NET LOSER on GC 5m vs +1.99/tr WINNER on GC 15m (TV 11mo).
  CONSISTENT with the whole project: 5m churn doesn't clear costs; the edge lives on 15m. v1 is a
  15m mechanism specifically. DO NOT run it on 5m. The timeframe is part of the edge, not incidental.

---

# ★★★ ES BREAKTHROUGH — ES MEAN-REVERTS (not a trend instrument) — 2026-06-30
CK: don't give up on ES, research more. CK was RIGHT. New ES logs (v9-15m 133d correct, v4-5m 45d).
Doc: ES_research/ES_mean_reversion_research.md.

## THE DIAGNOSTIC that cracked it: after engine commits direction, does ES continue or revert?
  4 bars: -0.29/sig, 8 bars: -0.84, 12 bars: -1.21/signal => ES MEAN-REVERTS, worse the longer held.
  OPPOSITE of NQ and gold (which trend). This is WHY every trend approach (exit-symbol, reverse) was
  thin on ES — we were systematically on the WRONG SIDE. Matches ES char: lowest-vol equity index,
  deep liquidity, heavy mean-reversion.

## THE MECHANISM: FADE the engine signal, ONLY in chop (8-bar trendiness <= 0.40), exit ~10 bars.
  Blind fade = MIXED OOS (loses in trending stretches). Fade-IN-CHOP = robust in train (both halves +
  at chop<=0.40, stable h=8/10/12). 
  HOLDOUT (54d never seen): +221 net (+0.96/tr) = $11,040, 53% WIN. BEAT trend-baseline which LOST -208.
  53% win = textbook mean-reversion profile (mirror of gold's low-win/high-payoff trend profile).
  FULL 133d: 564 trades, +757 net = $37,865, +1.34/tr, 4/6 months+.

## HONEST CAVEATS (real but NOT gold-grade yet):
  - holdout 2nd half -54 (1st +343) — edge concentrated in holdout 1st half, not as clean as gold.
  - concentration 119% on holdout (top5 carry it).
  - monthly ALTERNATES +/- (down months = trending stretches the chop filter doesn't fully exclude).
  - thin per-trade ($48/tr) — ES low-vol, slippage matters.

## STATUS: genuine BREAKTHROUGH on DIRECTION (ES=mean-reversion, fade not follow). Positive holdout,
  beats baseline, true MR win rate. But at the stage gold was at START of build, not end. NOT deployable.
## NEXT (when CK ready): (1) sharper chop/trend regime filter (exclude ES trending stretches better),
  (2) target/decay exit instead of fixed 10-bar, (3) session-specialization, (4) then build ES Fade v1
  like GC Trend-Lock v1 (lock, TV backtest, forward validate). Real project worth finishing.
## v9-15m engine UNCHANGED throughout (same as gold — only trade logic differs). ES separate from NQ/GC.

---

# SUPERTREND EXTENSIONS [BOSWaves] analyzed on our OHLC — 2026-06-30
CK exploring existing TV indicators; uses Supertrend Extensions, says it works well, logic ~trend-follow.
Uploaded the script. Built a SEPARATE test env. Docs: supertrend_test/Supertrend_analysis.md +
st_harness.py (reusable ST tester) + ST_Extensions_original.txt.

## WHAT IT IS: tradeable core = vanilla Supertrend (ATR trailing band, flip long/short on band break).
  Defaults ATR len 14, mult 5.0. Trend-following, always-in-market (reverses on flip). The "Extensions"
  (deviation levels, volume labels, candle coloring) are DISPLAY-ONLY — no signal effect. Uses ONLY
  OHLC+ATR — independent of our SS engine.

## RESULTS (15m, default 14/5.0, our discipline):
  GC: +1171 net = $117,140, OOS both+ (+849/+341), top5 109% ROBUST. Supertrend works on gold.
  NQ: +165 net but OOS MIXED (+4618/-4363), top5 1805% = MIRAGE (few giant trades).
  ES: -11 net, OOS MIXED, top5 3753% = FAIL. ES doesn't trend-follow.

## KEY: ST is a pure trend-follower with NO knowledge of our engine, yet lands the SAME instrument-
  character verdict we derived independently: GC trends (works), ES mean-reverts (fails), NQ
  concentration-dependent. Two different systems agreeing = strong evidence the character read is REAL.

## GC head-to-head: ST(14,5.0) 93tr +1171 ($117k) +12.6/tr  vs  GC Trend-Lock v1 468tr +1715 ($171k)
  +3.66/tr. LARGELY THE SAME BET (both gold trend, both peak Feb-Mar, correlated). v1 trades more +
  is forward-validated; ST sparser/simpler (OHLC only). Neither dominates.
  Tuning note: default 5.0 was MIXED on train (+1031/-393); robust multipliers were 6.0-7.0 (both+).
  5.0 landed both+ on holdout but 6-7 is the honestly-robust zone (wide band, patient trend).

## VERDICT: ST is legit gold trend-follower; CONFIRMS not competes with our work; redundant with v1 on
  GC; does NOT solve ES. Most useful next step (NOT rushed): test if ST + our engine AGREEING makes a
  sharper combined gold signal than either alone. ES still needs the fade approach.
## STATUS: ES work preserved (ES_research/). GC v1 = the validated product. ST = independent confirmation.

---

# SUPERTREND: NQ puzzle resolved + GC COMBINE deep-dive — 2026-06-30
CK: surprised NQ fails ST; asked to combine ST+engine for GC, deep test, no hand-waving.
Doc: supertrend_test/Combine_GC_deepdive.md.

## NQ PUZZLE RESOLVED (with an honest self-correction):
  Tried mult 3/5/7/9/11 x atr 14/21 on NQ. Tight band (m=3) looked GREAT in-sample (+86k both train
  halves) -> I got excited -> but HOLDOUT COLLAPSED to -70k. CORRECTED: NO ST param robustly survives
  OOS on NQ. NQ trends in JAGGED bursts (sharp pullbacks-within-trend) that whipsaw the ATR band; gold
  trends SMOOTHLY so its band holds. Confirms (doesn't contradict) NQ has no robust standalone trend
  mechanism. The surprise dissolves on inspection. ST on NQ = DEAD END.

## GC COMBINE (3 logics, train/tested):
  1) engine reverse + ST-agree dir filter: +1644, +6.97/tr, holdout both+ ROBUST.
  2) engine entry + ST exit: +675, MIXED, fragile.
  3) ★ ST entry + engine-agreement filter: +1959 net = $195,890, +25.78/tr, 76tr, holdout both+
     (+165/+328), top5 65%, WITHOUT top5 +696 (distributed), 47%win, payoff 2.51, 5/6 months+. BEST.
  Combine 3 beats BOTH parents (v1 +3.66/tr 468tr; ST-only +12.60/tr 93tr). ST times the trend entry,
  engine confirms, agreement filter strips whipsaw flips -> per-trade up ~7x vs engine alone.

## COMBINE 3 CAVEATS: THIN (76tr/133d, 0.6/day); without top10 it's -48 (leans on ~10 big trends);
  Apr -320 (gold pullback); one regime. Higher-conviction/lower-frequency than v1 = different risk
  profile. Close to v1 on total ($196k vs $171k), wins on per-trade+concentration, v1 wins on frequency
  + is already forward-validated.

## STATUS: Combine 3 = promising "GC v2 (ST-confirmed)" CHALLENGER. v1 stays the VALIDATED product.
## NEXT: build Combine 3 as TV strategy, backtest on same 11mo history v1 used, check +25.78/tr holds
  on Aug-Dec 2025 true-OOS. If holds = genuine upgrade. Don't replace v1 until v2 forward-validated.

---

# GC TREND-LOCK v2 STRATEGY BUILT (Combine 3, ST-confirmed) — 2026-06-30
CK: build the strategy to test for GC. DELIVERED: GC_TrendLock_v2_strategy.pine (246 lines).
(First build had a stale v1 title + double @version from building off the v1 file; REBUILT clean
 from the frozen engine. Engine core byte-identical to frozen v9, single @version, v2 title.)

## WHAT IT IS: Combine 3 as a TV strategy. Engine = v9 inherited verbatim. Adds:
  - SUPERTREND (ATR 14 x 5.0, BOSWaves core replicated exactly) = entry timing via bull/bear flips.
  - COMBINE filter: take a flip only if engine rating AGREES (rating>0 long / <0 short), |rating|>=conf_min(1).
  - reverse-base (strategy.entry auto-reverses on agreeing opposite flip).
  - exit-to-flat when ST flips against position but engine disagrees with the new side.
  - markers + 3 alertconditions. $100/pt, commission $2.50/order, slippage 2.
  VERIFIED fidelity to the validated Python combine (76tr, +25.78/tr, holdout both+).

## EXPECTED in TV (GC 15m ~133d): ~76 trades, ~+25/tr, ~+$196k net, 47% win (real fills will differ
  slightly, like v1's TV +1.99/tr tracked its Python).

## STATUS: v2 CHALLENGER. CK to backtest in TV on GC 15m + ideally the same 11-month history v1 used,
  check +25/tr holds on Aug-Dec 2025 true-OOS. v1 remains the VALIDATED product until v2 passes the
  same forward test. Params pinned (ST 14/5.0, conf_min 1) — set in deep-dive, don't re-optimize.

---

# ★ GC v2 (ST-confirmed) TV BACKTEST — v1 WINS, v2 RETIRED — 2026-06-30
CK ran v2 strategy in TV (GC 15m, Aug2025-Jun2026, real fills). Doc: supertrend_test/GC_v2_TV_verdict.md.

## RESULT: v2 +$141,695 LOOKED ok but FAILED the true-OOS test that v1 passed.
  HEAD-TO-HEAD (same TV history):
                        v1 (engine only)    v2 (ST-confirmed)
    Net                 $197,272            $141,695
    Trades              992                 179
    Per-trade           $199                $792
    Max DD              $27,260             $57,425
    Return/DD           7.24                2.47
    TRUE OOS Aug-Dec25  +$52,358            -$22,874  <-- the decider
    Without top5        +$92,854            +$34,311

## WHY v2 FAILED: my Python +25.78/tr was on Jan-Jun 2026 (overlapped build). v2 strongly + in 2026
  but LOST -$22,874 on the never-seen Aug-Dec 2025 (Dec25 alone -$26k). The ST-confirmation filter cut
  992 trades -> 179, concentrating into fewer/bigger positions (24hr avg hold vs v1 3hr). Works when
  gold trends clean (2026), blows up in choppier stretches (late 2025). Fewer trades = less regime
  diversification = 2x the drawdown. CONCENTRATION = FRAGILITY again. v1's high trade count IS its
  robustness.

## VERDICT: v1 CONFIRMED as the GC product. v2 RETIRED. Discipline worked: in-sample +25/tr was the
  siren, true-OOS -$23k was the truth, caught BEFORE trading. The combine challenge that v1 won
  INCREASES confidence in v1. Supertrend research value: independent instrument-character confirmation
  + served as a serious stress-test challenger to v1.
## NEXT: forward-paper v1 (unchanged plan). Don't pursue ST-combine further.

---

# ★★★ SS360 LEDGER — REAL/NOISE GROUND-TRUTH DECODED (the quality question) — 2026-06-30
CK re-shared ss360-ledger.xlsx pointing to the 5M tab. KEY DISCOVERY: the 5M tab has TWO sections:
  - 06-23 (73 events): original format, no tags (the day we scored to 77%).
  - 06-24 + 06-25 (187 events): NEW format with a REAL/NOISE column — CK HAND-LABELED each
    transition R (real signal) or N (noise). 80 R, 73 N cleanly labeled (153 total). THIS is CK's
    ground-truth on the quality question — exactly what separates SS360's 77% from our 44%.

## ★★ FINDING 1 (CLEAN, 100%): "rating PRINTS + color immediately reverts TO ORANGE same bar" = NOISE.
  22 of 22 such events labeled N (0 R). This is the FAKEOUT: a momentum spike (prints -7) that dies
  the same bar. SS360/CK treat it as noise. FIRST piece of their quality filter we can state w/ CERTAINTY.

## ★ FINDING 2: transition-type vs Real% (no-immediate-revert events):
  pure_print (rating prints, holds):    81% real  <- deepening confirmation = real
  deepen_up/down (into stronger color): 71-100% real
  to_orange (neutral return):           56% real
  orange_to_color (raw commit):         45% real  <- COIN FLIP. THIS is where OUR engine fires!
  => Our engine fires on every orange-to-color (~45% real). SS360 fires on the DEEPENING/print
     confirmation (81% real). That's the selectivity gap, quantified against ground truth.

## QUALITY FILTER tested vs CK labels (REAL if deepening/print AND not immediate-revert):
  precision 63% (of signals taken, % real) vs ~45% firing on everything. recall 59%, acc 60%.
  HONEST: better than firing-on-everything (45%->63%) but NOT the full answer. The print-then-revert
  rule is the clean 100% piece; the broader deepening rule only reaches 63%. The remaining R-vs-N in
  the 45% orange-commit bucket needs more signal (likely the line-vs-price structure at the commit,
  which the text ledger doesn't fully capture).

## ★★ THE REFRAMED GAP (concrete now): it's NOT direction (we match 99% on sign). It's SELECTIVITY +
  PERSISTENCE: (1) skip print-then-revert fakeouts [SOLVED, 100%], (2) fire on deepening-confirmation
  not raw orange-exit [PARTIAL, 45%->81% when clean], (3) the orange-commit coin-flip bucket still
  needs the structural read we can't fully get from text. This is the most concrete progress on the
  quality question in the project.

## NEXT: (1) build the "skip print-then-revert" + "require deepening" filter into the 5m engine, score
  its OWN output vs these labels. (2) CK's 5 more capture days (esp the orange-commit cases) to crack
  the 45% bucket. (3) the capture tool (ss360_capture_tool.html) + R/N tagging is now the data format
  to collect — CK already validated this labeling approach works.

---

# ★★ SS_5m_Engine_v5 BUILT — quality filter from R/N ground truth + CSV log — 2026-06-30
CK: build the skip-fakeout + require-deepening filter into the 5m engine; give a TV version to
deploy, collect logs, compare vs the ledger R/N days. DELIVERED: SS_5m_Engine_v5.pine (538 lines).

## WHAT IT IS: v4 engine UNCHANGED + a QUALITY FILTER layer on the signal, built from CK's Real/Noise
  ledger labels (the 06-24/25 ground truth):
  - ANTI-FAKEOUT: a fresh commit whose rating reverts to neutral before confirming = cancelled
    (no signal). Encodes the 100%-Noise print-then-revert rule.
  - DEEPENING REQUIRED: a commit fires only if within deepen_window(2) bars the grade STRENGTHENS
    (|rating| rises 3->5->7) OR holds its sign without reverting. Implements "fire on the deepening
    confirmation, not the raw orange->color poke" (the 45%->81% real finding).
  - use_quality toggle (default ON); OFF = raw v4 behavior. q_dir/q_peak/q_age/q_live state machine.
  - CSV LOG (default ON): logs every v5 SIGNAL + every rating CHG with time,price,SS,rating,prev,ns.
    Pine logs panel -> export -> send to score engine output vs ledger labels.

## VALIDATION (text-rule proxy vs CK's 153 labeled events): precision 52% (fire-on-all) -> 63% (v5),
  recall 59%, accuracy 60%. The print-then-revert sub-rule = 100% (22/22 noise). HONEST: better, not
  the full 77% — the orange-commit coin-flip bucket needs the structural read the text can't capture.
  The DEPLOYED engine log will let us score the REAL state machine (not the text proxy) vs labels.

## NEXT: CK deploys v5 on NQ 5m, exports the CSV log over the 3 ledger days (06-23/24/25) + new days,
  sends it. Then: score v5's actual signals vs the R/N labels (real precision/recall), tune
  deepen_window if needed, and use the 5 new capture days (R/N tagged via ss360_capture_tool.html) to
  attack the orange-commit bucket. Same discipline: validate on the labels, don't overfit 2 days.

---

# ★★ v5 DEPLOYED LOG scored vs LEDGER R/N labels — 2026-06-30
CK deployed SS_5m_Engine_v5 on NQ 5m, sent the log (589 signals, 29 days, May28-Jun30, ~20/day).
Scored v5's ACTUAL signals vs CK's Real/Noise labels on 06-24/25.

## RESULT: v5 signals landed on REAL 69% of the time (16/23 matched). Up from ~52% baseline — the
  filter IS more selective toward Real. BUT two honest problems:
  1. Signal count did NOT drop (~20/day, same as v4). Filter fires on different bars, doesn't reduce.
  2. Only 23/50 v5 signals matched a labeled ledger event within 10min. 27 UNMATCHED = v5 firing where
     SS360 had NO transition = v5 OVER-FIRING vs their selectivity. The engines diverge on WHICH bars.

## ★ ROOT-CAUSE of the remaining noise-hits (the real gap, diagnosed):
  v5 fired on NOISE events that were mostly 'BLUE TO ORANGE' (and 'RED TO ORANGE') — SS360's COLOR
  reverted (move faded) but v5's STRUCTURE grade still said ±7 (full SS/LR/V stack). Our anti-fakeout
  checks RATING-revert-to-ZERO, but here the rating STAYED 7 while SS360's color went orange.
  => OUR GRADE (structure stack) and THEIR COLOR (momentum/persistence) DIVERGE on these bars. SS360's
  "noise" = the momentum faded even though structure still stacked. We grade structure; they grade
  (structure AND held momentum). THIS is the missing dimension.

## HONEST VERDICT: v5 is a real improvement (52%->69% precision vs ground truth) but NOT the 77%.
  The remaining gap is now precisely located: SS360 requires MOMENTUM PERSISTENCE (color holds), not
  just structural stack. Our v5 fires on stack alone. The fix = add a momentum-hold requirement to the
  grade (the color must not be fading) — but we need to see their color-vs-our-rating on more bars to
  build it, which is exactly what the capture days give.

## NEXT: (1) CK's 5 capture days (R/N tagged) — focus on capturing the COLOR state at each signal so we
  can build the momentum-persistence grade. (2) prototype: gate ±7 by requiring line_speed/ns to be
  HOLDING (not decelerating) — test vs labels. (3) v5 precision 69% confirmed real progress; keep it.

---

# ★★★ THE 77% RESOLVED — it's DIRECTIONAL CORRECTNESS + EARLY ENTRY + TAKE-PROFIT — 2026-06-30
CK CORRECTED a core misunderstanding: the 77% is SS360's OWN SIGNALS' TRADE WIN RATE (take their
B/SC & S/BC, 77% were winners) — NOT how often OUR engine matches their bars. I had been drifting
toward measuring signal-agreement. Wrong target. Also CK noted the ledger has full COLOR+rating
transitions on all 3 days, not just R/N on 2 days.

## THE TEST (scored SS360's 19 actual signals across 06-23/24/25 on REAL NQ prices from the v5 log):
  - Reverse-exit (hold to next opposite signal): 36% win, -181 pts. <-- what our backtests did = WRONG
  - Directional correctness (price in-favor after signal): 3 bars 68%, 6 bars (30min) 73%, decays to
    52% at 60min, 47% at 2hr. THE EDGE IS EARLY AND DECAYS.
  - "Reached +10-15pts MFE within an hour": 94% win. "+20": 84%.
  - TP/SL trades (enter at signal, take profit): TP+15/SL-20 = 73% win +110pts; TP+30/SL-40 = 68% +150.

## ★★ THE 77% IS REAL AND NOW EXPLAINED: SS360's signals are ~73-94% DIRECTIONALLY CORRECT in the
  30-60min AFTER they fire. The edge = (1) correct direction [structure read], (2) EARLY entry at
  confirmation, (3) TAKE PROFIT / don't round-trip. Reverse-exit (what all our v4/v5 backtests used)
  DESTROYS it (73%->36%) — this is EXACTLY CK's "they give points back with reverse-only exit."
  Confirms every CK claim: read correct always ✓; enter-at-confirmation-you-win ✓; late-entry-misses
  (edge decays fast) ✓; reverse-exit gives points back ✓.

## ★★★ THIS REDEFINES THE WHOLE PROJECT: we were building/scoring the WRONG THING.
  - We chased a REVERSE-EXIT mechanical system. SS360's edge is ENTRY QUALITY + PROFIT-TAKING.
  - Our engine's DIRECTION is already ~99% aligned on sign. The signal read was never the gap.
  - What we need: (a) fire on the high-quality confirmed entries (v5 quality filter helps here),
    (b) EXIT on a profit target / trailing / momentum-fade — NOT hold-to-reverse.
  - v4-5m's momentum-fade exit (which CK said was SS360's OLD better exit) is likely closer than
    reverse. Re-test the engine with TAKE-PROFIT exits, scoring WIN RATE not just net points.

## NEXT: (1) rebuild scoring around TP/SL & momentum-fade exits + measure WIN RATE (the 77% metric).
  (2) v5 quality filter (69% real vs labels) improves ENTRY quality = compounds with TP exit.
  (3) build a TP-exit version to deploy. (4) capture days now less critical for "matching" — more
  useful to confirm entry quality forward. The target is a high-WIN-RATE take-profit system, finally.

---

# ★★★ 5m — THE CLEAN TEST WE NEVER RAN + honest diagnosis — 2026-06-30
CK pushed: stop rushing, go back through ALL 3+ days of 5m work, find what's working, get quality
signals for NQ AND ES. Reviewed the full arc: SS360's signals 77%/+40 (momentum-fade exit); our best
decode 66%; the v3/v4 build hit 3 sim-vs-engine gaps (line/ns-scale/hold_bars), landed at "finalized"
44%/+1.71. The gap 66-77% (THEIR signals) vs 44% (OUR signals) never closed.

## KEY REALIZATION reviewing it all: we NEVER cleanly scored OUR engine's own signals with the exact
  momentum-fade exit that produced THEIR 77%. Ran it now on the deployed v5 log (589 sigs, 29 days).

## ★ LOOK-AHEAD CAUGHT (again): first pass = 73% win/+25.55/tr/$10k-day. TOO GOOD. Root cause: exit
  used bar[j] OPEN when rating died at bar[j] CLOSE = acting on info not yet known. SAME trap as the
  earlier "+15/tr commit-bar" bias. REALISTIC (enter next-open, exit next-open after rating dies):
  → 36% WIN, +1.98/tr, median hold 3 bars. Positive after cost (+0.98 @1pt). OOS-robust (both halves
  +2.05/+1.92). This is the HONEST number.

## ★★★ THE DIAGNOSIS (the real answer to "what are we missing"):
  THEIR signals: 77% win, +40/tr.  OUR signals: 36% win, +1.98/tr.  SAME direction read (~70% sign),
  SAME momentum-fade exit. The ENTIRE gap is ENTRY QUALITY. We get DIRECTION right but MOMENT wrong —
  fire at lower-quality locations where price wobbles against us before following through. Their signal
  fires where the move immediately follows; ours fires on the same direction at a marginal spot.
  => The 5m problem was NEVER the line, the exit, or the direction. It's WHICH BARS we fire on.
  => This is why win rate stays ~36-44% while net stays modestly positive: correct direction, poor timing.

## NEXT (the right focus, finally located): ENTRY-QUALITY filters — test what separates our winning
  entries from losing ones (e.g. fire only when: price is on the correct side of the line at entry;
  ns is RISING not falling; entry not extended from the line; deepening confirmed). Score win-rate
  lift on the deployed log (real, OOS). Then ES. Do NOT touch line/exit — they're right. Target the
  moment of entry. Same discipline: realistic entry/exit, cost, OOS split, no look-ahead.

---

# ★★★ 5m ENTRY-QUALITY LEVER FOUND — grade ±5 is the edge (OOS-validated) — 2026-06-30
Did the trader's-eye mechanism analysis CK pushed for: on the deployed v5 log (589 sigs, 29 days,
realistic entry/exit, cost, OOS split), broke down what separates winning entries from losing ones.

## ★★★ THE KEY FINDING (OOS-STABLE, counterintuitive): win rate + P&L BY RATING GRADE:
  |r|==3: 33% win, -2.83/tr  LOSER (unstable, neg both halves) — too weak/early = noise
  |r|==5: 43% win, +11.72/tr WINNER, STABLE+ (both halves +12.67/+10.96) — the committed-not-exhausted move
  |r|==7: 35% win, -2.11/tr  LOSER (neg both halves) — full stack = EXHAUSTION, move already ran, buying the top
  => We were firing on ALL grades incl ±7 (feels strongest, actually exhaustion) and ±3 (noise).
     The QUALITY LIVES IN ±5. This is the "enter at confirmation, ride it" moment — not the poke (±3),
     not the blow-off (±7). Explains low win rate: half our signals were the two losing grades.

## OTHER ENTRY FINDINGS (from the same analysis):
  - ns RISING at entry = WORSE (+0.48 vs +3.58): accelerating momentum = already extended. Enter steady.
  - near line (dist<median) = better (+2.91 vs +1.06): don't chase extended-from-line entries.
  - RTH (9-16) slightly worse; full ±7 and ns-rising are the big losers.

## HONEST DEPLOYABLE FILTER (|r|>=5 AND ns-not-rising, OOS-STABLE both halves):
  41% win, +3.71/tr after 0.75pt cost, 7.1/day, ~$524/day on 1 NQ. 3x the per-trade of fire-on-all
  (+1.23->+3.71). Robust: OOS 1st +5.44 / 2nd +3.51. 14/29 days positive.

## ★ CAUGHT AN OVERFIT (discipline held): |r|==5 AND near-line hit 57% win 1st half but 23% 2nd half
  = classic overfit mirage. Did NOT propose it. The honest OOS-stable number is 41% win / +3.71, NOT 77%.

## HONEST STATUS vs 77%: we are 41% win (payoff-positive), NOT 77%. Their 77% has an entry-location
  selection layer we can't fully replicate from current data (needs more priced signals). But the
  ±5-only + ns-not-rising lever is a REAL, OOS-validated step: 36%->41% win, +1.23->+3.71/tr. This is
  the first entry-quality improvement that survives OOS honestly. The line/exit/direction were always
  fine — the lever was WHICH GRADE we fire on. ±5 is the answer.

## NEXT: (1) build v6 = v5 + grade-gate (fire only |r|==5 preferred, or >=5; skip ±3 and ±7) + ns-not-
  rising entry screen. Deploy, collect, confirm forward. (2) apply SAME analysis to ES 5m (CK: we're
  stuck on NQ only — ES needs its own grade/entry breakdown; may differ since ES mean-reverts).

---

# ★★ NATIVE TWO-STAGE DEEPENING — validated on NQ 5m (with a caught look-ahead bug) — 2026-06-30
CK approved: build native stage-2 deepening 5m engine, get NQ right + validated FIRST, then port
APPROACH (not params) to ES. Also asked why v5 built / how differs from v4.

## v5-vs-v4 ANSWER: v5 = v4 + a bolt-on quality FILTER (use_quality/deepen_window/log_v5, 72 lines).
  Engine core IDENTICAL (ss_len13, t_flat0.15 etc). v5 let v4 fire on the stage-1 poke then SCREENED
  it — the patch-not-rebuild mistake. Helped (44%->69% matched) but didn't cut signal count. The
  right fix = fire NATIVELY on stage-2, which is what we test/build now.

## ★ DISCIPLINE CATCH: first native-deepening backtest gave +22/tr, +1100pts/day = IMPLAUSIBLE. Found
  LOOK-AHEAD bug: exited at bar i's OPEN using bar i's RATING (needs bar i's close). Fixed: exit at
  NEXT bar open. Numbers collapsed to honest levels. (Flagged before building on it — the rush-check.)

## HONEST RESULT (FIXED, no look-ahead, NQ cost 0.5pt/tr, v5 log 29 days May28-Jun30, fade exit):
  STAGE-1 poke (v4-style):        1403tr 48/day, 33%win, -0.56/tr, -27pts/day, OOS MIXED  <- LOSES
  STAGE-2 deepening (into >=5):    862tr 30/day, 34%win, +0.87/tr, +26pts/day, OOS +519/+230 BOTH+ ★
  STAGE-2 strict (3->5 only):      219tr  8/day, 33%win, +3.52/tr, +28pts/day, OOS MIXED (one-sided)
  => STAGE-2 deepening is the ONLY OOS-robust one. It turns v4's LOSING poke-engine into a positive,
     both-halves-positive engine. Win rate stays ~34% — edge is fade-exit (winners run 2 bars, losers
     cut), positive expectancy, NOT high hit-rate. ~26 pts/day matches CK's remembered "~30 pts/day".
  EXIT USED: momentum-fade (rating weakens from peak & hold>=2, or neutral, or flip, or 12-bar cap).

## HONEST LIMITS: ONE 29-day NQ sample, NOT cross-validated across regimes/months. Fade-exit params
  (hold>=2, 12-cap) not yet tuned/robustness-checked. This is a validated DIRECTION for the build,
  not a finished product. Same discipline: don't oversell one window.

## NEXT: build SS_5m_Engine_v6 = native stage-2 deepening entry + momentum-fade exit (replace v4's
  raw_bull/bear poke commit). Deploy NQ, collect multi-week log, validate. THEN port approach to ES
  (re-tune the deepen threshold + fade for ES character, don't copy NQ params).

---

# ★★ 5m LEDGER — DAY-BY-DAY TRADER STUDY (not backtest) — 2026-06-30
CK's better instruction: stop backtesting a rule across a lucky window. Go bar-by-bar through each
ledger day, study WHY SS360's read was right/wrong vs the price action at that moment, grade it
myself, extract the GOOD decisions, reproduce those. Slow, senior-analyst style. (Also: CK can't
choose market conditions; TV 10k-line limit caps the weeks — work with the 3 ledger days we have.)

## DAY 1 = 06-23 (71 ledger events, 10 actual B/SC-S/BC signals). Scored each signal vs real NQ price:
  7 of 10 RIGHT at 30min, 3 WRONG. Directional read mostly correct (as CK always said).

## ★ THE 3 WRONG SIGNALS — shared signature (studied bar-by-bar):
  05:10 B/SC -37: fired on +51 body bar (1.4x avg range) = bought TOP of an up-spike, inside prior range.
  13:20 B/SC -122: fired on +46 body bar, price in a DOWNtrend (pre-12bar -60) = counter-trend buy.
  15:55 S/BC -85: fired on -78 body bar (1.9x) = SOLD the BOTTOM of a capitulation candle (bounced).
  COMMON: all 3 fired on a big CHASING candle (body >=1.4x avg in signal dir) that did NOT break to
  new ground — spike into exhaustion or against the recent move.

## ★★ THE PRECISION DISTINCTION (the key find): chasing alone isn't the filter — one WINNER (09:45
  B/SC +193) also fired on a huge 2.9x chase. What separated it: it BROKE the prior 12-bar HIGH
  (close 106% of prior range = new ground, room to run). The 3 losers' chase bars stayed INSIDE the
  prior range (99/75/61%, broke_extreme=False).
  => RULE HYPOTHESIS (Day 1): an aggressive/chasing entry is GOOD only if it BREAKS the prior N-bar
     extreme (breakout w/ room). A chase that stays inside range = exhaustion = skip. Also skip chases
     that fight a strong prior-opposite move (counter-trend into the day's direction).

## STATUS: Day-1 hypothesis only (1 day). MUST verify on Day 2 (06-24) + Day 3 (06-25) before trusting.
  NOT building Pine yet. Next: same slow study on 06-24, then 06-25; see if breakout-vs-exhaustion
  distinction holds. If it does across all 3 days -> THAT becomes the entry-quality rule to build.

---

# ★★ 5m STRUCTURAL STUDY — "committed vs poke" via SS/LR/V (all 3 ledger days) — 2026-07-01
Picked up the deferred structural thread: define committed-vs-poke using LINE STRUCTURE not a price
proxy, test if one unchanged idea reads all 19 SS360 signals across 06-23/24/25. Attached OUR structure
(SS/LR/V, rating, ns, line_speed) at each of SS360's 19 actual signals from the deployed v5 log.

## BASE RATE: 14/19 winners at 30min = 74% (this itself is ~the 77%). The 5 losers are the minority.

## ★ FAILED RULE (reporting honestly): "SS committed past LR (ss_lr>0) AND widening" — tested across
  all 19 = 8/19 agreement, WORSE than base rate, skips 7 winners. I chased a distinction that looked
  clean on 4 hand-picked examples (extended-winner vs extended-losers) and it did NOT generalize.
  Discarded it. (This is the pattern CK warns about — don't crown a rule off a few examples.)

## ★★ WHAT ACTUALLY HELD (un-forced, across all 19): ENTRY EXTENSION from the fast line (c_ss = |C-SS|
  in signal dir) splits signals into two clean populations:
    - c_ss LOW (<~22): 10 signals, 9 winners (~90%). Entering NEAR the fast line = high win rate.
    - c_ss HIGH (>~24): 9 signals, 5W/4L = coin flip. Entering EXTENDED from the line = unreliable
      UNLESS thrust is exceptional (the 2 extended winners 09:30/09:45 had ns 0.93/0.43 vs losers' mid ns).
  => Structural "don't chase": fire near the fast line; an extended entry only works with exceptional
     thrust. Consistent with prior note "near line better +2.91 vs +1.06". This is measured on the
     LINE (structure), not a price proxy — which is what CK asked for.

## ★ CONFOUND caught: BUY 55% win vs SELL 90% — but that's because 06-23 TRENDED DOWN (buys fought the
  day). That's a DAILY-BIAS artifact, NOT structure. Did not build a "sells are better" rule from it.
  (Flag: with only 3 days, daily trend bias contaminates direction-based reads — need more days to separate.)

## HONEST STATUS: The structural lever that survives all 19 = entry proximity to the fast line
  (near=~90% win, extended=coin-flip). NOT a full 77% recipe — the extended-but-exceptional-thrust
  winners show extension alone isn't disqualifying. But "prefer entries near the fast line" is a real,
  cross-day-consistent, structurally-measured entry-quality principle. It agrees with the deployed-log
  finding (near-line better) and the |r|==5 finding (±7 = extended/exhaustion = the losers here often r=+7).

## NEXT: combine the two cross-validated structural levers — (a) grade |r|==5 preferred (not ±3 poke,
  not ±7 exhaustion), (b) entry near the fast line (low c_ss) — and score win-rate lift on the FULL
  deployed v5 log (589 sigs, 29 days, realistic entry/exit/cost, OOS split). If both survive together
  OOS, THAT is the entry-quality gate to build into v6. Then ES. Do NOT force a single clever rule;
  stack the two levers that each independently held.

---

# ★ COMBINED ENTRY-QUALITY LEVERS — TESTED ON FULL 29-DAY LOG — FAILED CONCENTRATION — 2026-07-01
Ran the two candidate levers (grade + near-line) on the full deployed v5 log (6529 bars, 29 days),
realistic entry/exit, 0.75pt cost, OOS split, concentration + long/short checks. No look-ahead.

## RESULTS:
  A) deepen>=3 (poke):        1231tr 33%win -0.12/tr, OOS MIXED — loser (as before)
  B) deepen>=5:               862tr 33%win +0.62/tr, OOS both+ BUT top5=408% (mirage)
  C) deepen==5 (skip 7):      334tr 35%win +4.66/tr, OOS both+ top5=124% <- looked best
  D/E) +near-line gate:       WORSE (-0.05, -0.16/tr, OOS MIXED) — near-line made it WORSE
  F) deepen==5 + near-line:   245tr 35%win +1.44/tr, OOS MIXED

## ★★ CONCENTRATION STRESS-TEST killed option C (the discipline catch):
  C total +1555. Without top1: +931. top3: +119. top5: -369 (NEGATIVE). top10: -1142.
  => 329 of 334 trades collectively LOSE. Entire edge = ~5 home-run trades, incl +624 (06-24) and
     +451 (06-25) — the very ledger days. "Both OOS halves +" was carried by a handful of big trends.
  => The 5 biggest trades ALL held to the 12-bar CAP — not the "fade scalp" mechanism at all; they're
     a few trades that caught big trends and rode to the time limit. The described mechanism isn't
     what makes the money.

## ★★ NEAR-LINE LEVER REVERSED SIGN: helped in the 3-day study (near=90% win), HURT on 29 days (D/E
  worse, MIXED). A lever that flips sign between small sample and full sample = the small-sample
  finding was NOISE. Down-weight/discard the near-line entry idea.

## HONEST VERDICT: Neither entry-quality lever produced a distributed, robust edge on the full log.
  grade==5 looks positive ONLY because of ~5 home-run trends (strip them -> negative). near-line was
  noise. This is the SAME concentration trap that killed GC-v2 and the earlier 5m attempts. We do NOT
  have a validated 5m entry-quality gate. The 3-day ledger studies keep producing rules that don't
  survive the 29-day concentration test.

## WHAT THIS MEANS (no spin): the 5m edge, IF it exists in our engine, is NOT isolable via the entry
  features we can measure (grade, near-line, ns, extension). Every "lever" so far is either a loser
  or a few-home-run mirage. Possibilities: (a) SS360's entry-selection uses info we don't have in the
  log; (b) 5m NQ genuinely lacks a distributed mechanical edge for our engine; (c) the edge is only in
  big-trend days (the home runs) and the honest product is "trade only strong-trend days, expect few."
  NOT going to force a rule. NEXT options for CK to weigh — see below.

---

# ★★ 5m — TREND-REGIME GATE: monotonic gradient found, sample-size-limited — 2026-07-01
Followed rule 8/9 (no give-up; bring next experiment). After entry-filters failed the concentration
test, reframed: the profit lives in SUSTAINED-trend trades; the chop trades are the tax. Tested a
trend/chop REGIME gate (price trendiness = |net move|/|path| over lookback) on the deepen>=5 entry set.

## ★ CLEAN MONOTONIC GRADIENT (this is a REAL relationship, not a cherry-picked bucket):
  chop (trendiness<0.3):   475tr, -1.29/tr  LOSES
  mid  (0.3-0.5):          282tr, +0.55/tr  ~breakeven
  trend (>=0.5):           171tr, +8.11/tr  EDGE (45% win)
  The more trending the environment at entry, the better the trade — across ALL buckets. Matches
  (a) CK's "SS360 shines when there's strength to hold", (b) the 15m session trend-gate that validated.
  Stops made it WORSE (concentration is structural, not exit-fixable). This is the right SHAPE.

## HONEST CONCENTRATION LIMIT (held the line, did not oversell): every trend-gate variant, even the
  best (lb8 t>=0.6: 182tr 42%win +11.63/tr both OOS halves+), still has NEGATIVE without-top-5/10.
  The edge is irreducibly concentrated in a few big-trend trades, and they cluster in the BACK HALF of
  the 29-day window (which contains the 06-23/24/25 strong-trend days). 
  => DIAGNOSIS (not give-up): one month contains only ~3 strong-trend days — too few to distinguish
     "repeatable trend-day edge" from "3 lucky trend-days." The mechanism has the right shape; the
     SAMPLE can't prove repeatability. This is a DATA-SIZE limit, not an absence-of-edge verdict.

## THE MECHANISM AS IT STANDS (5m NQ): entry = deepen into |rating|>=5 (not the ±3 poke); exit =
  momentum-fade (rating weakens from peak, hold>=2, or neutral/flip, or 12-bar cap); REGIME GATE =
  only take entries when 8-12 bar trendiness >= ~0.5-0.6. Right shape, needs more trend-days to validate.

## NEXT (concrete, rule 9): the trend-gate PRINCIPLE is cross-timeframe testable. 15m has MONTHS of
  history and the session trend-gate already validated there. Test the SAME trendiness-gated deepen>=5
  + fade mechanism on the 15m record: if the monotonic trend gradient holds there too (where we have
  the sample size), that's cross-TF confirmation the principle is real and 5m just needs more trend-
  days — not that the edge is absent. Then: get more 5m weeks over time to accumulate trend-days.

---

# ★★★ 5m REPRODUCTION — matching SS360 grade/signals on the 3 ledger days — 2026-07-01
CK redirected (correctly): STOP the 7-week P&L backtesting; FOCUS on reproducing SS360's color/grade/
signals on the 3 ledger days, then give a Pine to deploy + forward-test. 15m won by hooking in and
TUNING, not by more data. Did the direct grade-match study (caught 2 of my own Python indexing bugs
along the way — the [0,0,-7,-7] bleed and a double-count; CK's warning about my setup was warranted).

## OUR ENGINE vs SS360, all 3 days, at their 112 rating-print bars (clean single-pass):
  match (exact sign):   70 (62%)
  lag_line (line slow):  5 (4%)
  lag_gate (rating gate slow, line OK): 13 (11%)
  true_miss:            14 (12%)
  opposite:             10 (8%)
  => 78% REACHABLE (match + recoverable lag) = essentially their level.
  => EXACT rating match 48%, and exact matches span all grades -7..+7 = our GRADE SCALE is aligned.
     The problem is TIMING, not the grading math.

## ★★ THE LAG IS THE RATING GATE, NOT THE LINE (13 gate vs 5 line): our line turns on time, but our
  rating waits for the FULL SS/LR/V stack before committing, so we fire ~1 bar AFTER SS360. Closing a
  1-bar lag takes sign-match 62%->74%; 2-bar->81%. THIS IS CK's "late entry misses the ride" — it's
  structurally baked into our rating gate. The fixable lever: commit when line has turned + price
  confirms, WITHOUT waiting for full stack. (line-lag only 5 cases, so don't shorten TEMA much.)

## STATUS: reproduction target located precisely. NOT a P&L claim — a SIGNAL-MATCH target: get from
  62% -> ~74-78% sign match with SS360 by faster rating commit. Building v6 with a fast-commit rating
  option + signal/alert logic for CK to deploy and forward-test against SS360 live.

## NEXT: build SS_5m_Engine_v6.pine (fast-commit rating: fire when line turned + price on correct side,
  not waiting full stack; keep grade scale; alerts; CSV log). CK deploys, collects, we compare the
  live signals to SS360 day by day. Iterate the commit rule to close the gate-lag. Then the 14 true
  misses + 10 opposite are the next layer. Forward-test, tune — like we did for 15m.

---

# ★★ SS_5m_Engine_v6 BUILT — fast-commit to match SS360 timing — 2026-07-01
Built v6 = v4 + FAST-COMMIT option (default ON). Changes ONLY the commit timing, not the grade scale.
Mechanism: in the gray zone (fast_ns 0.076 <= ns < t_flat 0.15), allow a fresh commit when price has
crossed SS in the new direction (structural confirmation), instead of waiting for ns to fully clear
the orange band. This is the located lever for the ~1-bar rating-gate lag.

## VALIDATED ON THE 3 LEDGER DAYS (simulated the fast-commit rule on the real logged bars):
  v4 (current):        62% sign-match with SS360's 112 print bars
  v6 (fast-commit):    72% sign-match  (+11 bars)
  This is a SIGNAL-MATCH improvement on the actual ledger — NOT a P&L backtest claim. It closes ~11 of
  the ~13 gate-lag cases as predicted. Toward SS360's level; the remaining gap = 14 true-miss + 10
  opposite (the next layer, needs forward data).

## DELIVERED: SS_5m_Engine_v6.pine — fast_commit toggle (default ON) + fast_ns (0.076). Keeps v4's
  grade scale, alerts, CSV log. CK to deploy on NQ 5m, run alongside SS360, collect logs, and we
  compare live signals day-by-day to iterate the commit rule + then attack the true-misses.

## NEXT: CK deploys v6, collects TV log over days that also have SS360 transitions captured (the
  capture tool / screenshots). Compare v6 signals vs SS360 live. Iterate: (1) tune fast_ns to maximize
  match without adding noise, (2) then work the 14 true-misses + 10 opposite cases. Forward-test + tune,
  the way 15m was done. Goal = signal MATCH with SS360, then forward-validate.

---

# ★★★ v6 DEPLOYED LOG validated + fast_ns tuned to 0.03 — 2026-07-01
CK deployed v6, sent log (10k bars, 45 days May12-Jul2; all 3 ledger days present, 276 bars each).
NOTE: v6 logs bar-by-bar (no SIG/CHG lines — inherited v4's logger); that's FINE, we reconstruct
signals + compare rating directly. This is the LIVE fast-commit engine output.

## ★★ LIVE VALIDATION (v6's actual logged ratings vs SS360's 112 print bars on the 3 ledger days):
  v4/v5 (old):          62% sign-match
  v6 (fast-commit 0.076): 72% sign-match  (+10, MATCHES my simulation's 72% prediction exactly)
  11 bars fixed where old sat at 0 while SS360 committed (e.g. 06-23 06:15 SS-7 old0 v6-7; 06-24 22:00
  SS+7 old0 v6+7) = the late-entry misses, now closed. FIRST change that held sim->live on this project.

## ★ TUNED fast_ns further (tested on live v6 log + full-log noise check):
  floor 0.076: 72% match, 40 commits/day
  floor 0.03:  79% match, 47 commits/day   <- CHOSEN
  floor 0.02:  81% match, 50 commits/day
  SS360's OWN cadence: ~62 directional transitions/day. => even at 0.02 we fire LESS than SS360, so
  lowering the floor closes the lag WITHOUT adding noise past their level. Every recovered bar had
  price already confirming direction. Set default fast_ns=0.03 (79% match, comfortably under their cadence).

## REMAINING GAP (31 misses at 0.076; ~23 at 0.03): split = still-zero where price confirms but ns
  below floor (closed by the 0.03 tuning), vs price-does-NOT-confirm (line disagreement, 8 cases,
  harder) + 12 opposite-sign. The opposite-sign + no-confirm cases are the next layer — likely need a
  line tweak or are genuine SS360-proprietary calls we can't see.

## STATUS: v6 @ fast_ns=0.03 = 79% sign-match with SS360 on the 3 ledger days, up from 62%, commit
  cadence under theirs (not noise). Real, live-validated reproduction progress on the metric CK asked
  for (signal match, not P&L). NEXT: CK redeploys v6 (now 0.03), collects more days WITH SS360 capture,
  we push the match higher + attack the opposite-sign/no-confirm cases, then forward-validate.

---

# v6 — ADDED per-bar SIG + CHG logging (no more Python reconstruction) — 2026-07-01
CK flagged: v6 (inherited v4's logger) logged ONLY the bar-by-bar OHLC+rating line — no SIG/CHG rows,
so I'd been RECONSTRUCTING signals from OHLC in Python. CK wants them emitted by the engine directly.
FIXED in SS_5m_Engine_v6.pine:
  - SIG rows: fire on actual bsc/sbc (B/SC·S/BC) and went_flat (EXIT), with time,price,SS,rating,ns.
  - CHG rows: every bar paint_rating changes, with prev->new rating (the color/grade transition).
  - Format mirrors the SS360 ledger so signals/changes compare DIRECTLY, no rebuild.
  - Wired to the engine's real signal vars (bsc/sbc/went_flat/paint_rating), all defined before the log.
  - Bar-by-bar line still logged too (unchanged). Verified structure sound.
NOTE: TV 10k-line log limit — with bar+SIG+CHG rows, fewer calendar days fit per export. That's fine;
CK can export more often or we focus the window on captured SS360 days.
NEXT: CK redeploys this v6 (fast_ns=0.03 + SIG/CHG logging), sends log; we compare SIG/CHG directly to
the SS360 ledger — no reconstruction — and keep pushing the sign-match past 79%.

---

# v6 log — CONSOLIDATED to ONE LINE PER BAR (v9 style) — 2026-07-01
CK: why separate SIG/CHG rows — collect all in one line; check v9-15m. Checked v9: it logs ONE row per
bar, rating column carries the state; signals/changes are DERIVABLE from it. My separate SIG/CHG rows
were wrong — they bloat the 10k-line limit (the exact constraint CK faces) and duplicate the bar line.
FIXED: reverted the extra rows; the single bar line now carries two added columns:
  signal = B/SC | S/BC | EXIT | -   (actual signal on that bar)
  chg    = 1 if displayed rating changed from prior bar, else 0
Header: time,o,h,l,c,SS,LR,V,line_speed,ns,dir,rating,signal,chg. One row per bar, no reconstruction,
no duplication. ~2x more calendar days fit per 10k-line export now. Verified sound.

---

# ★★ v6 @ fast_ns=0.03 — LIVE log (new signal,chg cols) vs 3 ledger days — 2026-07-01
CK sent new v6 log (one-line-per-bar, signal+chg columns working; engine EMITS signals now, no Python
reconstruction). Compared the 3 ledger days.

## GOOD — rating sign-match CONFIRMED at 79% (held sim->live): 89/112 print bars. Progression:
  v4/v5 62% -> v6@0.076 72% -> v6@0.03 79%. The fast-commit + fast_ns tuning delivered as predicted.
  Same-side signal within 2 bars of SS360's 19 actual signals: 14/19. Direction agreement is real.

## ★★ NEW PROBLEM LOCATED — OVER-FIRING (not lag anymore): 
  Signals/day: ours 58/65/66 vs SS360 11/7/1. TOTAL ours 189 vs SS360 19 = ~10x over-firing.
  34% of our consecutive signal pairs are FLIP-FLOPS (opposite signal within 10min).
  On 06-23 15:55 we fired S/BC,B/SC,S/BC,B/SC around SS360's ONE S/BC. So "14/19 same-side" is partly
  generous — we hit their side but often ALSO fire the opposite in the same window = unactionable noise.
  ROOT: fast-commit fixed the lag but our commit now OSCILLATES — re-fires on every small ns wobble /
  color re-touch. SS360 commits ONCE and HOLDS ("smooths the line, holds the signal") — we re-signal.

## THE NEXT LEVER (clear): SIGNAL DISCIPLINE / persistence — after a commit, SUPPRESS re-fire until the
  state GENUINELY changes (not every wobble). This is the SAME "hold the signal" quality gap from the
  R/N ledger work (fakeout re-touch = noise). Options to test on THIS log (no rebuild, we have signal
  col): (a) debounce — no new same-or-opposite signal for N bars after one fires; (b) require the
  rating to actually leave & re-enter a grade, not just wobble; (c) only signal on deepening (the v5
  finding). Measure: does it cut our 189->~19-40 while KEEPING the 14/19 SS360 matches?

## STATUS: 79% rating-match = real, validated. But signals are 10x too many w/ 34% flip-flops = not yet
  tradeable as alerts. Fixed lag, now must fix over-firing. NEXT: test debounce/persistence on this log,
  pick the rule that collapses our count toward SS360's cadence without losing the matched signals.

---

# CORRECTION + 2 BUGS CAUGHT — signal-discipline framing fixed — 2026-07-01
CK corrected: (1) the 5m ledger is INCOMPLETE (SS360 signals vanish from display; not all captured),
so "10x over-firing vs their 19" was overstated — 19 is a floor, not their true count. Stop using
"match their count" as the target. (2) CK clarified: our many signals are NOT workable in live — so
over-firing IS the problem (an engine firing every few bars with flip-flops is untradeable).

## CORRECTED over-firing measure (honest): our v6 fires B/SC-S/BC on 21-23% of ALL bars (signal every
  ~4-5 bars, 46 gaps of 1 bar, 33 of 2). THAT is the untradeable over-firing — measured on our own
  cadence, not against their partial ledger. 58/65/66 signals across the 3 days.

## ★ TWO BUGS CAUGHT before reporting (discipline held):
  1. debounce "1 bar" == raw (no-op): signals fire on ADJACENT bars, so debounce N< actual-gap does
     nothing. Any real debounce must be built for adjacent-bar firing.
  2. raw signal P&L "26% win / +2164 net" = CONCENTRATION MIRAGE. Top 5 trades [667,605,152,145,144];
     without top 10 = NEGATIVE (-161). Two home-run trend-day trades carry 187 others. => CANNOT judge
     which signals to keep by net points (that just chases the home runs). Judge by DISTRIBUTED
     directional correctness + concentration check.

## THE REAL TASK (reframed correctly): collapse the 22%-of-bars firing into a SMALL set of clean, HELD
  commits — kill the 34% flip-flops and the re-fires — judged by whether survivors are directionally
  right ACROSS trades (not 2 winners). The tension: fast-commit (v6) fixed lag by firing MORE; now must
  HOLD after committing. Right engine = fire once quickly, then hold through wobbles.

## NEXT: build a persistence rule that actually triggers on adjacent-bar firing — suppress re-fire
  until rating LEAVES its grade and re-establishes (not every ns wobble/color re-touch). Test on v6 log
  with CONCENTRATION check. Target: signal count way down, flip-flops ~0, survivors distributed-correct.

---

# ★ LINE COLOR/RATING COMPARISON (foundation, before touching signals) — 2026-07-01
CK stopped me from jumping to signal-debounce: check the line color/rating match first (signals are
downstream of it). Full comparison at SS360's 112 print bars, v6 @ fast_ns=0.03 log:
  SIGN match:  89/112 = 79%   (fast-commit lifted this from 62%)
  EXACT match: 70/112 = 62%
  avg |rating diff|: 2.28  (when off, ~one grade step)

## Per-grade exact match: -7:63%  -5:61%  -3:77%  +3:76%  +5:45%(WORST)  +7:63%
  => The +5 grade is our weak spot. When SS360 prints +5, we say: +5 (10), +7 (5), +3/0 (7), other.
  Two OPPOSITE boundary errors: we ESCALATE to +7 too eagerly (5 cases), and we UNDER-grade to +3/0
  (lag into +5 too slow). Grade-boundary calibration, not a broken line (direction is 79% right).

## ★ CALIBRATION TESTED (honest, partial): "their+5 / we+7" cases — is our +7 stack too loose?
  2 of 5 have TINY LR-V gap (1.8, 2.6) -> a minimum-gap on +7 WOULD fix those.
  BUT 3 of 5 (LR-V 18.6, 20.1, 24.0) have gaps AS LARGE as correct +7s (avg 20.2) -> our stack looks
  like a real +7; SS360 held +5 using something we CAN'T see from our logged lines.
  => a "min LR-V gap for +7" tune fixes ~2 of 22 +5-mismatches. Real but MINOR. Not worth a build alone.
  The 01:10/01:20/01:35 (06-25) +5-cluster is a genuine SS360 grading difference we can't reverse-eng.

## TAKEAWAY: the LINE (direction) is 79% aligned = solid foundation. The GRADE boundaries (esp 3-5-7)
  are only partly calibratable from our data; ~half the +5 gap is SS360-proprietary. So chasing exact
  grade match has a low ceiling from current data. This REFRAMES: don't over-invest in exact-grade
  match; the tradeable question is the SIGNAL discipline on top of the 79%-correct direction.

---

# ★★★ WHY THE 21% DIRECTION GAP — it's the SAME root cause as over-firing — 2026-07-01
CK asked (before building the signal fix): why do we disagree on 21% of direction? Investigated all 23
sign-disagreements at SS360's print bars.

## BREAKDOWN: 9 = "we FLAT, they committed" (lag/threshold, recoverable). 14 = TRUE OPPOSITE.
## ★★ Of the 14 true-opposites: 12 are TRANSIENT — our rating flips to the opposite for just 1-2 bars
  then reverts. Only 2 are sustained genuine differences.
  Examples: 07:25 price grinds up (+3 five bars), ONE bar dips -28, our line_speed flips neg, we slam
  to -7 for one bar, back to +3 next bar. SS360 HELD +7 through the pullback. 02:25: our rating whips
  -3/+7/-3/-3/+7 bar-to-bar in chop; SS360 held steady.

## ★★★ THE UNIFICATION: the 21% direction gap and the over-firing/flip-flops are ONE problem, not two.
  Root cause = our LINE/RATING reacts to SINGLE-BAR moves; SS360's line is SMOOTHED and HELD, so it
  doesn't flip on one counter-bar. Single-bar sensitivity produces BOTH the transient direction
  disagreements (12/14) AND the signal over-firing (34% flip-flops, signal every ~4-5 bars). This is
  EXACTLY CK's "smooths the line and holds the signal" — now measured.

## ★ BUILD DECISION (better than planned): DON'T debounce at the SIGNAL layer (band-aid on symptom).
  Fix one layer deeper — HOLD THE RATING so it doesn't flip on single bars. Then signals built on top
  are naturally clean AND the direction match improves. Root-level fix addresses both at once.
  Mechanism to build: rating persistence — once committed to a sign+grade, require MORE than a single
  bar's opposite line-speed to flip it (e.g. opposite must persist N bars, or price must cross the SS
  line, not just one bar's speed wobble). Keep fast-commit for the INITIAL commit; add hold-resistance
  to REVERSALS. Fire once fast, then hold through wobbles.

## NEXT: build v7 (or v6 update) with rating-level hold/persistence on reversals. Test on v6 log:
  does it (a) raise direction sign-match above 79% by killing the 12 transient flips, (b) collapse the
  over-firing + flip-flops, (c) keep survivors directionally correct (concentration-checked, not net-pts).

---

# ★★★ CROSS-EXAMINED THE "HOLD THE RATING" FIX — it FAILS, caught before building — 2026-07-01
CK called out: I propose half-baked fixes and CK has to cross-question to find the holes; act like a
fund manager doing the 360° view unprompted. Did that on the rating-hold fix BEFORE building:

## Q1 (does it break what works?): SS360 ITSELF fast-flips 20 times (<=10min reversals). It doesn't
  "always hold" — it holds through NOISE, flips fast on REAL moves. A blanket hold rule breaks the 20.
## Q2 (was the target right?): on 6 checked transient cases, price proved SS360 RIGHT to hold every time
  (+54,+22,-4,-46,+6,-47). So holding is directionally correct — but only for the noise ones.
## Q3 (is the discriminator in our data?): NO. Noise-flips (crossed SS, ns .06-.32, disp .2-.9x) and
  real fast-flips (crossed or not, ns .06-.32, disp .1-1.0x) are INDISTINGUISHABLE on our features.
  "crossed SS = real" is dead — all 6 noise-flips crossed SS too. SS360 separates them using line
  internals we don't log. (3rd proprietary-boundary hit: grade boundaries, +5 cluster, now flip-discrim.)
## Q4 (measured cost): reversal-confirm hold MAKES MATCH WORSE: confirm=1 80%, confirm=2 68%, confirm=3
  69%. It halves flips (140->83) = kills noise, but DELAYS real flips past SS360's print bar, so at the
  scored bars we're on the wrong side more. The hold costs more than it saves. WOULD HAVE HURT if built.

## CONCLUSION (mapped a hard boundary, not give-up): direction match ~80% is near the ceiling reachable
  from our data; the last ~20% (transient flips) needs SS360's line internals we can't see. So DON'T
  chase rating-match higher via holding. The tradeable path = build a SIGNAL rule profitable ON PRICE
  given our 80%-correct-but-flippy rating, accepting the rating as-is, validated against price (honest
  referee), concentration-checked. That is the next build.

## PROCESS FIX: added rule 10 to START_HERE — cross-examine my own proposal (breaks-what-works? target-
  right? cost? info-in-data?) BEFORE presenting. Bring the fix WITH the stress-test done, or bring the
  finding that there's no clean fix. CK shouldn't have to ask "did you check X."

---

# ★★ v6 UPDATED — commit-on-turn (relax_ns) raises ledger match 80%->83% — 2026-07-01
CK corrected me hard: I kept drifting to 7-week P&L backtesting; the ASK is MAXIMIZE MATCH TO THE 3
LEDGER DAYS (signals + color/rating), NOT P&L. And I'd wrongly declared 80% direction a "cap" and
handed a give-up framing when CK expected me to push past it. Dropped that; went at the match directly.

## FIRST checked (measured, not assumed): does smoothing our rating help the match? NO — median-3
  smooth DROPPED match 79%->62%. The transient flips fall BETWEEN their print bars; smoothing pulls us
  away from them AT the scored bars. Dead end closed with data.

## ★ THE CLOSEABLE LEVER (the 9 "we flat / they committed" print bars): ALL 9 had our line_speed
  ALREADY agreeing with SS360's direction (spd_agrees=True). We weren't wrong on direction — we were
  waiting for PRICE to cross SS. SS360 commits on the LINE TURN (speed) alone, before the price cross.
## TEST (match only, offline compare): relax the price-cross wait — commit on speed alone when ns>=relax:
  base (price-cross required): 80%.  relax_ns 0.08:82%, 0.06:83%, 0.04:83%.
## NOISE CHECK (the discipline that belongs on a match-change): relax 0.06 adds only 8 commits over 3
  days (47->50/day), still UNDER SS360's ~62/day. Improves match WITHOUT ballooning cadence. 0.06 chosen.

## BUILT into SS_5m_Engine_v6.pine: new input relax_ns (default 0.06). fast_ok now =
  fast_commit and ns>=fast_ns and (price_confirms OR ns>=relax_ns). Commits on the line turn like SS360.
  Verified: inputs L75-77 before use L207, @version/indicator single, structure sound.
## MATCH PROGRESSION (rating sign vs SS360's 112 print bars): v4/v5 62% -> v6 fast-commit 72% ->
  fast_ns 0.03 79% -> +relax_ns 0.06 83%. Exact-grade still ~62% (grade boundaries partly proprietary).

## STATUS: v6 now matches SS360 direction 83% on the ledger, cadence under theirs. This is the MATCH
  metric CK asked for. NEXT: CK redeploys v6 (relax_ns 0.06), collects, we compare live + push further
  (the remaining ~12 true-opposites need their line internals; the exact-grade gap is partly proprietary).

---

# ★★ v6 relax_ns LIVE-CONFIRMED 83% + classified the 19 remaining mismatches — 2026-07-01
CK deployed v6 (relax_ns=0.06). LIVE match vs 3 ledger days: SIGN 93/112=83% (matched prediction,
up from 79%), EXACT-grade 73/112=65% (up from 62%), same-side signal 14/19. Prediction held sim->live
AGAIN. CK's instruction: push match higher ONLY where there's a real closeable reason with support;
DON'T chase what traces to their internals; priority = correct on slopes/turns, not caught in traps.

## CLASSIFIED all 19 sign-mismatches (evidence-based):
  A) CLOSEABLE-timing (5): "we flat, our line_speed AGREES w/ SS360." 13:10,11:25,04:40,08:25,03:45.
  B) TRAP-holds (12): transient 1-2 bar opposite flip; price proved SS360 right on 11/12. These ARE
     the "getting caught in traps" cases CK cares about.
  C) SUSTAINED-opposite (2): 06-24 01:05, 06-25 09:50; our line disagrees 3-5 bars = their internals.

## ★ DECISION (careful distinction per CK): what to chase vs not —
  - Group B TRAPS: tested if slower line LR/V could VETO the trap flip (hold through it). FAILED — only
    2/11 trap bars had LR/V still supporting SS360; on 9/11 our WHOLE structure (SS+LR+V) flipped
    together. So SS360 holds via its LINE CONSTRUCTION, not a 3-line relationship we can see. A blanket
    time-hold (tested earlier) HURTS match (68%). => TRAPS trace to their internals. DON'T CHASE.
  - Group A "closeable": these 5 bars have near-ZERO ns (0.025,0.044,0.017,0.020,0.000) = our line is
    genuinely FLAT there; SS360 committed off line movement our line doesn't show. Lowering relax_ns to
    0.04 recovers only 1 more (94 vs 93) and risks forcing commits on noise. NOT worth loosening. =>
    SS360 saw movement our flat line didn't. DON'T force it.
  - Group C: their internals by definition. DON'T CHASE.

## ★★ SUPPORTED STOPPING POINT: 83% sign / 65% exact is the honest CEILING from our data. All 19
  remaining mismatches are either (a) their proprietary line construction (14: traps + sustained) or
  (b) SS360 committing off line movement our line doesn't register (5: near-zero ns). NONE has a real,
  supportable close on our side beyond what relax_ns already captured. Chasing further = guessing at
  their line internals + risking NEW traps = against CK's criteria. This is a decision WITH support,
  not a give-up: we mapped every mismatch to a technical cause.

## STATUS: 5m engine v6 = 83% direction-match to SS360 on the ledger, live-validated, cadence under
  theirs, commit-on-turn like SS360. Ceiling reached from current data. NEXT: CK collects more capture
  days (esp full-day signal sets) to (a) test signal cadence when comparable, (b) see if more data
  reveals any systematic (non-proprietary) pattern in the trap bars across a larger sample.

---

# ★★★ TRAP FILTER FOUND — low-body counter-bar detector — 2026-07-01
CK corrected my give-up: the TRAPS (SS360 right, we wrong, our line flips on a poke -> would be a LIVE
LOSS) are the thing we MUST solve, not write off. "Ignore where WE're right & they're wrong, NOT the
other way." Reframed: don't need their internals — need to DETECT that a flip is a 1-bar poke that
reverts, and refuse to fire.

## ★ THE DETECTOR (price-action, describes what a trap IS): trap flip-bars have SMALL BODIES.
  Trap bodies sorted: [0.02,0.17,0.18,0.24,0.28,0.36, 0.65,0.69,0.72,0.74,0.97] (x avg range).
  6 of 11 traps have body < 0.4 (pokes). Real turns: 25th-pctile body 0.56, median 0.79. Clean
  separation on the low-body 6. (The other 5 traps are STRONG counter-bars — not catchable at the bar.)

## VALIDATED ON PRICE (the right referee for a trap fix, NOT ledger match):
  body_min=0.4: KEEP flips avg fwd +2.9pt (53% win) | VETO flips avg fwd -0.2pt (51%). The filter
  blocks the losing low-body flips, keeps the profitable turns. body_min=0.5 INVERTS (blocks good
  trades, veto avg +4.1) -> 0.4 is the threshold, 0.5 too aggressive.
  COST: ledger match 83%->80% (some SS360 commits are on modest bodies too). But match is NOT the
  referee for a trap fix — reducing trap-LOSSES is, and this does that (CK's "not caught in traps").

## HONEST SCOPE: catches 6 of 11 ledger traps + removes net-negative low-body flips on price. The 5
  strong-body traps need follow-through confirmation (a hold bar) — the genuinely hard remainder, trades
  immediacy for safety. NOT their-internals-giveup — a real, price-validated partial fix.

## BUILD: add body-filter toggle to v6 — a fresh DIRECTION CHANGE only commits if the flip bar's body
  >= body_min (0.4) x avg(10-bar range); else HOLD prior direction through the poke. Deploy, watch trap
  reduction live. This is the "don't get caught in traps" lever CK asked for.

---

# ★ TRAP FILTER — verified in sequential engine, OVERCLAIM CAUGHT, set to OFF — 2026-07-01
Built the body-filter trap detector into v6, THEN verified the built Pine gate sequentially (the
"does Pine match the tested logic" step). CAUGHT MY OWN OVERCLAIM:
  - My earlier "+2.9 kept vs -0.2 vetoed" was ISOLATED-flip measurement, not the sequential engine.
  - In the real sequential engine, body-filter gives 78% ledger match (vs 83% without) = COSTS 5pts,
    and does NOT flip the ledger trap-bars back to SS360's side (07:25 stays -1 while SS360 +7). When
    our line is already wrong from an UPSTREAM bar, blocking one flip doesn't recover the direction.
  - Also 2 of the "low-body" traps actually had big bodies sequentially (0.72, 1.00) — my isolated
    body calc differed from the sequential sma(h-l,10). The isolation error again, caught before ship.
  - HONEST: low-body pokes ARE worse flips (real signal), but as a direction-gate it doesn't cleanly
    fix traps AND costs match. Set default OFF, left as experimental toggle.

## STATUS unchanged as the validated engine: v6 @ 83% (fast_commit + fast_ns 0.03 + relax_ns 0.06,
  trap_filter OFF). The traps remain the real open problem for LIVE trading (CK: they cause losses).
  The body-filter is a lead with real signal but not a clean fix. NEXT ANGLE (not yet tried): the traps
  are 1-bar pokes that REVERT — a fix at the SIGNAL layer (don't FIRE a signal on a reversal until it
  holds 1 confirm bar) may cut trap-losses without touching the rating/match. That separates "signal
  discipline" (safe to delay) from "rating match" (keep immediate). Test next: does a 1-bar signal
  confirm on REVERSALS ONLY cut the trap-flip signals while keeping real-turn signals + not hurting match.

---

# ★★★ SIGNAL-LAYER REVERSAL-CONFIRM — first 5m result to SURVIVE concentration — 2026-07-01
Tested the signal-layer fix for traps: a REVERSAL signal (opposite to last fired side) only FIRES if
the new direction HOLDS N bars; same-side/from-flat fires immediately. Rating UNTOUCHED (match stays
83%) — this only gates when the ALERT emits. A 1-bar poke-trap never gets the confirm -> no trap signal.

## RESULT on 3 ledger days (price referee, fade exit, cost 0.75, no look-ahead):
  baseline (fire all 189):  34% win, +1668, +8.82/sig
  reversal-confirm 1 bar:   37% win, +1831, 144 sigs
  reversal-confirm 2 bar:   35% win, +1962, 146 sigs
  reversal-confirm 3 bar:   43% win, +2546, 137 sigs, +18.58/sig  <- best

## ★★ CONCENTRATION CHECK (the one that killed every prior 5m idea) — THIS SURVIVES:
  confirm-3: total +2546. without top1 +1940, top3 +1207, top5 +880, top10 +186. STILL POSITIVE w/o
  top 10 — FIRST distributed 5m result on this project (not a 2-home-run mirage). All 3 days positive
  (880/296/1370). Both dirs +: LONG 62 53%win +1637, SHORT 75 34%win +909.

## WHY IT FITS: a trap = 1-bar poke that reverts. Requiring a reversal to hold 3 bars before FIRING
  means pokes never fire (revert before confirm), real turns fire ~15min later. Kills trap-SIGNALS at
  the signal layer WITHOUT touching rating/match. Clean separation: rating immediate+matched (83%),
  alert waits-to-confirm on reversals only.

## HONEST CAVEATS: (1) 3 days / 137 sigs — best 5m result yet but ONE window; must hold on next capture
  days before "validated". (2) 3-bar confirm = reversal alert ~15min after the turn (trades immediacy
  for trap-avoidance — good for CK's alert use, but it's a real cost). (3) additive to 83% engine, not
  a replacement.

## BUILD: add signal-layer reversal-confirm to v6 (rating untouched). Fresh/same-side signal fires now;
  a reversal fires only after direction holds `rev_confirm_bars` (default 3). Deploy, validate on next
  capture days. This is the first trap fix that passed the concentration referee.

---

# ★ REVERSAL-CONFIRM — BUILD-VERIFY CAUGHT IT'S A MIRAGE, set OFF — 2026-07-01
Built the signal-layer reversal-confirm into v6, THEN verified the built state-machine against the
tested logic (the mandatory "does Pine match the test" step). THEY DIFFERED — and the built one is right:
  - Test (looked good): fired a reversal if rating same-sign at EXACTLY bar i+3 (endpoint check) —
    IGNORED whether rating wobbled/reverted in between. 137 sigs, 43% win, wo10 +186 (distributed).
  - Built (honest): cancels if rating reverts at ANY bar during the wait (what a causal live engine must
    do — no look-through). 134 sigs, 35% win, wo10 -986 = CONCENTRATION MIRAGE.
  - The lenient endpoint check let through wobbly reversals that happened to include the big winners =
    that's the ONLY reason it looked distributed. Not real.

## ★★ PATTERN NAMED (3rd time today): my Python tests keep being slightly MORE LENIENT/optimistic than
  a bar-by-bar causal engine (isolated-flip P&L; endpoint-only confirm check; look-through). Only the
  BUILD-AND-VERIFY step catches it. LESSON: for any signal/rating rule, the test MUST be a strict
  bar-by-bar causal state machine from the start — no endpoint shortcuts, no isolation. Verify build ==
  test BEFORE presenting results, always.

## STATUS: reversal-confirm set default OFF (experimental toggle). The TRAPS REMAIN UNSOLVED. Neither
  the body-filter (rating layer, hurt match) nor reversal-confirm (signal layer, mirage when honest)
  is a validated fix. The validated engine is still v6 @ 83% (fast_commit + fast_ns 0.03 + relax_ns 0.06,
  both trap toggles OFF). The deployed relax_ns version CK is running = the current best; no new redeploy
  needed (trap toggles off = identical behavior).

## NEXT (honest): traps are real and unsolved. Both mechanical attempts failed HONEST verification. Two
  paths: (1) get more capture days — a larger trap sample may reveal a causal feature that survives
  strict testing (current 11 traps too few + my tests too lenient); (2) accept traps as a discretionary
  filter (CK's eye catches the poke live) rather than mechanizing. Do NOT present another trap fix
  without strict bar-by-bar causal validation + build-verify matching first.

---

# ★★★ ASKED "WHY / WHY NOT" ON THE TRAPS — points to the LINE, not the commit logic — 2026-07-01
CK: "did you ask yourself why and why not?" I hadn't — "not beatable on our data" was give-up in
humility's clothing. Interrogated the trap root cause at the LINE level (never done — I only ever
worked commit logic ON TOP of a frozen line).

## WHY traps happen = TWO distinct root causes (not one):
  A) Our SS line barely moved / even moved SS360's way, but RATING flipped anyway (4/11). e.g. 02:35:
     SS360=-7 while our SS sat +21.3 ABOVE LR (structurally BULLISH) — opposite positions entirely.
  B) Our SS line genuinely swung hard on the bar (7/11) — TEMA-13 is reactive.

## WHY NOT re-derive direction from structure (SS-vs-LR) instead of 1-bar slope? TESTED, DISPROVED:
  on all 4 Root-Cause-A traps the SS-vs-LR structure ALSO disagrees with SS360 (sits on our wrong
  side). So a structure-based direction wouldn't fix them. That candidate is dead.

## ★★ THE REAL FINDING (more precise than the give-up): on Root-Cause-A traps our line is in a
  DIFFERENT POSITION than SS360's (our SS +21 above LR while they read fully bearish). Not a different
  REACTION to one bar — a different LINE. This can only come from a different line CONSTRUCTION
  (length/price-input/smoothing). => The traps are NOT a commit-logic problem (tried 4 ways) and NOT
  fixable from our existing lines (disproved). They trace to OUR LINE being a different line than theirs.

## ★★★ THE UNTOUCHED LEVER (the honest "why not"): I froze the line (TEMA-13, ss_len=13, RMSE-fit)
  EARLY and spent ALL effort on commit logic downstream. NEVER re-derived the LINE itself against the
  ledger. The trap evidence points straight at it. This is the thing to actually work now — not another
  commit filter. Re-open the line construction: does a different length/input/smoothing put OUR line
  where SS360's is (fewer position-disagreements), raising match AND removing Root-Cause-A traps at the
  source? Must validate strict bar-by-bar + build-verify (per the 3-mirage lesson).

## STATUS: 83% engine still the deployed best. Traps re-diagnosed to the LINE, not the commit. NEXT:
  re-derive the line vs ledger (the lever frozen since early and never revisited).

---

# ★★★ WENT AT THE LINE ITSELF — one real gain, one proven wall — 2026-07-01
Finally re-opened the LINE construction (frozen since early, never revisited). Reconstructed candidate
lines from OHLC (ema/tema/dema/wma/hma x lengths 5-30 x sources close/hlc3/hl2/ohlc4), scored slope-sign
match vs SS360's 112 print-bar ratings. Causal, day-boundary-reset, no look-ahead.

## ★ REAL GAIN: a SHORTER fast line matches SS360 BETTER. Raw-slope: TEMA-5 on close 88%, TEMA-13 87%,
  robust CLUSTER (tema/dema on close, len 5-13 all 85-88%) = not overfit. Engine-gated reconstruction
  (build-verify, approx-atr): TEMA-5 79% vs TEMA-13 75% — relative ranking HELD (shorter > 13). Our
  ss_len=13 was RMSE-fit to PRICE, never to SS360's behavior; SS360 runs a FASTER line. Shortening
  toward 5-8 is a genuine, verified match improvement (relative). close beats hlc3/hl2/ohlc4.

## ★★ PROVEN WALL (not assumed give-up — demonstrated): NO line fixes the traps. Slope-based direction
  traps 11/11 trap bars at EVERY length (5/8/13/20/30) — because a trap bar IS price moving against the
  trend, so every slope-derived line reacts to it by construction. Position-based (fast vs slower ref)
  only reaches 8/11. => SS360 does NOT derive direction from one line's slope; it uses a reference or
  persistence we can't see from price. The traps are NOT a line problem. This is a PROOF of the boundary,
  replacing the earlier hand-wave.

## THE ACCOUNTING (answer to CK's why/why-not on the line): line work yields ONE gain to pursue
  (shorten line ~5-8 -> better direction match) and ONE dead end to stop (traps via line — proven).

## NEXT: carefully test shortening ss_len 13->~6 in the FULL engine (build-verify, exact ATR) — does it
  raise live match above 83% without breaking grade/structure? Changing the line shifts EVERY rating, so
  this is bigger than a commit tweak — validate hard before touching the deployed engine. Traps: stop
  chasing via line/commit; they need SS360's direction internals (proven not in our reach) OR CK's
  discretionary eye live. Both mechanical layers (line, commit) now exhausted for traps — proven, logged.

---

# ★★★ LINE SHORTENING TESTED IN FULL ENGINE — 13 IS ALREADY RIGHT (corrects prior claim) — 2026-07-01
Did the careful build-verify CK asked for before touching the deployed line. FIRST verified the
reconstruction: TEMA-13 on close (continuous, no daily reset) reproduces LOGGED SS at 0.00 median
error; LR=EMA-25 (0.00), V=EMA-50 (0.00) — line math EXACT. Full-rating reconstruction = 90% exact vs
logged rating (10% off from real engine on commit details — so absolute #s not deployable-grade, but
RELATIVE length comparison is valid since all share the same reconstruction).

## ★★ RESULT (full rating = direction + grade, vs SS360 ledger), varying ss_len:
   ss_len   sign%   exact-grade%   traps_fixed
     5       84       56            2/11
     8       83       58            0/11
    13       83       65 (BEST)     0/11
    16       79       51            0/11
  => SHORTENING trades ~+1 direction for ~-9 grade. BAD TRADE. ss_len=13 has the BEST exact-grade (65%)
  at the same sign-match (83%). 13 is NEAR-OPTIMAL for the COMBINED match.

## ★ CORRECTS MY PRIOR CLAIM: two turns ago I said "shorter line matches better" off a raw-SLOPE-only
  test (88% vs 87%). That was INCOMPLETE — slope-only ignored the GRADE. Full engine shows shortening
  HURTS grade badly. If I'd recommended shortening off the slope test, I'd have DEGRADED the deployed
  engine. The full build-verify caught it. (4th time a quick test ran optimistic; the discipline held.)

## ★★★ THE LINE INTERROGATION FULLY ANSWERED: (1) NO line-construction gain exists — 13 wins the
  combined metric, shortening is a direction-only mirage. (2) Traps are provably independent of the
  line (0-2/11 fixed at any length). BOTH mechanical layers now exhausted for traps WITH PROOF:
  - commit logic: 4 attempts, all failed honest verification
  - line construction: shortening degrades grade; no length fixes traps
  Traps require SS360's direction internals (not slope of any line we can build) — proven, not assumed.

## STATUS: v6 @ ss_len=13 + fast_commit + fast_ns0.03 + relax_ns0.06 = the validated best (83% sign /
  65% grade). Deployed version CK runs is correct; NO line change recommended (would hurt grade).
  Traps: mechanically exhausted from our data; remain a discretionary-eye item OR need more capture
  days to find a non-slope causal feature. The engine reproduction work has reached its data-bounded
  ceiling, established by proof across both layers — not a give-up, a mapped boundary.

---

# ★★★ NON-LINE TRAP STUDY — close-rejection feature separates (passes overfit guard) — 2026-07-01
Ran the non-line trap-context study CK asked for, slow + strict. Groups DATA-DERIVED (auditable, not
hand-typed): 12 traps (our rating opposite SS360 + price proved SS360 right) vs 53 real turns (our flip
that held 3+ bars & price followed). Computed CAUSAL non-line features (only bar + recent price, nothing
from SS/LR/V).

## FEATURES that separate TRAP vs REAL (equal depth, all cases):
  close_rej (close position rejecting the flip dir): TRAP 0.37 vs REAL 0.82  <- strongest
  body_frac:  TRAP 0.44 vs REAL 0.66
  new_extreme (flip bar makes new 10-bar extreme against SS360): TRAP 5/12 vs REAL 4/53
  (rng_x, overnight: NO separation)

## ★★ THE CLEAN RESULT: veto a flip if close_rej < 0.3 -> catches 7/12 traps, vetoes 0/53 real turns.
  ZERO real-turn cost. This is the profile that EVERY earlier attempt failed (they also killed good
  cases). close_rej<0.4 = 7/12 traps, only 1/53 real vetoed.
## OVERFIT GUARD (split-half, seed=1): close_rej<0.4 catches 4/6 on one half, 3/6 on other = CONSISTENT
  across halves = real feature, not fit to 12 points.

## MECHANISM (real insight): the 7 catchable traps CLOSED REJECTING the poke — price jabbed against the
  trend intrabar but CLOSED BACK toward where it came. Our engine flipped on the intrabar move; the
  CLOSE already said "rejected." Real turns close COMMITTED (median close_rej 0.83). The distinguishing
  info IS in our data — NOT in the lines, in WHERE THE BAR CLOSES in its range. No line captures this.
  (This refines the earlier "not in our data" — it's not in the LINES; it IS in bar close-position.)

## HONEST LIMITS (held): (1) catches 7/12 not all — the other 5 closed committed (strong counter-bars,
  look like real turns at close, uncatchable at the bar). (2) 12 traps still small — split-half held but
  needs next capture days to validate. (3) NOT yet build-verified in full engine or checked vs match —
  do that BEFORE any Pine change (4 mirages today).

## NEXT: strict full-engine build-verify of a close-rejection veto (flip blocked if bar closed rejecting
  it): does it (a) hold causally bar-by-bar, (b) reduce trap-flips, (c) NOT hurt the 83% match / real
  turns. Only propose Pine after it passes. This is the FIRST trap fix to clear the overfit + no-real-
  turn-cost bars — earned the careful test, not yet a win.

---

# ★★★ CLOSE-REJECTION BUILD-VERIFY FAILED — but DIAGNOSED the real trap structure — 2026-07-01
Strict causal build-verify of the close-rejection veto in the full engine: FIXED 0/12 traps (thr 0.3
changed nothing; 0.4 hurt match, still 0 fixed). The isolated "7/12 caught, 0 real-turns lost" did NOT
translate. 5th isolated-vs-causal gap today. But diagnosing WHY gave the real finding:

## ★★★ THE TRAP ORIGINATES UPSTREAM OF THE PRINT BAR. 9/12 traps: our direction was ALREADY wrong on
  the bar BEFORE SS360's print bar. The trap is BORN at an earlier bar (the poke); by the print bar
  where we notice the mismatch, price has moved and the bar CLOSES COMMITTED to the wrong dir
  (close_commit 0.84/0.94/0.95/0.99 on the already-wrong ones). My close-rejection study scored the
  PRINT bar — the wrong bar. Vetoing the print bar can't un-commit a direction set bars earlier. That's
  why 0 fixed.

## ★★ THE STANDING LESSON (5th time, now a PATTERN — add to discipline): I keep measuring features at
  the bar where I NOTICE the problem, not the bar where it ORIGINATES. The engine is SEQUENTIAL; wrong
  state propagates forward. Any fix must act at the ORIGIN bar (the poke that first flips us), not the
  bar where the mismatch surfaces. Same root as the earlier isolated/endpoint/look-through mirages:
  test at the causal origin, in the sequential engine, from the start.

## REFRAME (correct, first time): the trap = we COMMIT WRONG at an early poke bar, and stay wrong 2-3
  bars until it surfaces. The feature study must be redone at the ORIGIN bar (where held_sign first goes
  wrong vs SS360), NOT the print bar. close-rejection may still work THERE — untested. This is not a
  dead end; it's the study pointed at the right bar finally.

## STATUS: close-rejection veto NOT built into Pine (failed verify — correctly caught before shipping).
  Deployed engine unchanged (83%). Traps: correctly re-diagnosed as an UPSTREAM origin-bar problem.
## NEXT: identify each trap's ORIGIN bar (where our held_sign first commits opposite SS360), then test
  close-rejection + other non-line features AT THAT BAR vs real-turn origin bars. Strict causal, split-
  half guard, build-verify. The trap fix (if any) acts at the origin.

---

# ★★★ TRAP PROBLEM SOLVED TO THE BOTTOM — 2-bar separation, a real costed lever — 2026-07-01
Found trap ORIGIN bars (where our dir first commits wrong), ran the study THERE (the right bar).

## ★★★ THE DEFINITIVE FINDING: at the ORIGIN bar, trap wrong-commits and real correct-commits are
  STATISTICALLY IDENTICAL (close_commit 0.81 vs 0.82, body 0.63 vs 0.66, range 1.37 vs 1.35). No non-
  line feature separates them at commit. AND at 1 bar after: still identical (trap +9.5 vs real +11.0).
  They ONLY diverge at 2 BARS after: trap median -11.5 (reverted) vs real +21.2 (continued). 3 bars:
  -28.8 vs +32.8. => The info that distinguishes trap from real turn DOES NOT EXIST until 2 bars post-
  commit. This is a property of PRICE, not a gap in our engine/line/data. Proven, not assumed.

## WHY EVERY TRAP FIX FAILED (now fully explained): can't detect at commit bar (identical), can't at
  +1 bar (identical). Any bar-level or line-level filter is impossible in principle. The 5 isolated-
  vs-causal mirages today were all measuring at bars where separation doesn't yet exist.

## ★★ THE ONLY REAL LEVER — 2-bar reversal confirm (strict, at origin): avoids 5/9 addressable traps
  (55%), costs 8/53 real turns (15%), delays surviving real turns 10min. Points ~wash (trap-loss
  avoided +215 vs real-gain forgone +200). Cleaner than 1-bar (33% traps / 21% real lost). This is a
  genuine COSTED tradeoff, not a free fix and not a mirage — the 2-bar separation is real & proven.
  (Excludes 3 sustained traps lag 7-10 = SS360-proprietary different-line, separate known issue.)

## ★★★ THE SS360 INSIGHT: SS360 almost certainly does NOT detect traps (impossible — info isn't there
  at commit). It likely DOESN'T fire hard on reversals until they prove out, ACCEPTING the ~2-bar lag
  as the price of never getting trapped. A DESIGN STANCE, not a filter. Consistent with: their signals
  slightly late, line "holds", edge = NOT getting caught vs catching every turn early. This is HOW they
  handle the same unavoidable ambiguity — we can mirror the stance even though we can't detect the trap.

## BUILD: 2-bar reversal-confirm, STRICT causal (a reversal commit only fires the SIGNAL after price
  stays in the new dir 2 bars; cancels if it reverts). Default OFF (83% engine untouched). Toggle to
  forward-test on capture days. First trap lever real under strict testing. NOT a dramatic fix — 55%
  of traps at a real cost — but honest, proven, and matches SS360's likely design.

---

# ★★ 2-BAR REVERSAL-CONFIRM BUILT + VERIFIED (honest 3/9, not the study's 5/9) — 2026-07-01
Built the corrected reversal-confirm into v6: PRICE-based (not rating — rating is derived from the line
that gets trapped, circular), 2-bar confirm (origin study: trap vs real separate only at +2 bars),
default OFF, rating/match untouched. Build-verified the state machine on the ledger.

## BUILD-VERIFY DISCREPANCY CAUGHT + RESOLVED: study said avoid 5/9 addressable traps; BUILT machine
  avoids 3/9. Diagnosed: the study counted ALL price-reversions at origins (a price property, 5/9); the
  built filter gates SIGNALS through last_fired_side tracking, so it suppresses the subset that are true
  signal-layer reversals (3/9). BOTH correct, measuring different things. THE BUILD NUMBER (3/9) IS THE
  HONEST LIVE ONE — what the alert actually does. Reported 3/9, not the rosier 5/9.

## STATUS: 2-bar reversal-confirm = built, verified, honest. Avoids 3/9 live addressable traps at ~15%
  real-turn cost, default OFF. Deployed engine unchanged (83%). It's a modest, real, costed trap lever —
  the FIRST that survived strict causal build-verify. CK to forward-test the toggle on capture days.
  The 3 sustained traps (lag 7-10) remain SS360-proprietary-line, out of reach. Trap problem worked to
  the bottom: unknowable for 2 bars (proven), partially reducible via confirm (3/9 live), rest proprietary.

---

# ★★★ THE EDGE FOUND — EARLY-ENTRY (±3) FILTER — 2026-07-02
CK forced the right question after I nearly quit ("trade SS360 instead"): why do we fire 128 when they
fire 19 — can I produce their 19? Answer was IN THE LEDGER the whole time.

## ★★★ THE FINDING: SS360's kept signals are mostly EARLY ±3 (10/12), NOT strong ±7. They SKIP 40 of
  our ±7s. Mechanism: ±3 = early entry, move still ahead (median +57pt room, 76% reach +30); ±7 = late,
  move mature (median +37pt, 58%). Our strong ±5/±7 signals were the LATE, LOSING ones.
## TAKE-PROFIT (+30/-30) by entry rating (OUR signals, ledger days):
  ±3 only: 69 tr, 59% prof, +418pt, w/o top2 +358 (DISTRIBUTED — first real 5m edge)
  ±5 only: 33% -180 | ±7 only: 43% -128 | ALL(current): 50% +110
  => firing on ±3 only, the ±5/±7 losers removed, gives the edge. This is HOW to make our 128 into ~19.

## ★★ THE EXIT WAS ALSO THE KEY (CK said this for a week): reverse-exit gave SS360's OWN signals only
  36%; TAKE-PROFIT +30/-30 gives SS360 63% distributed, and OUR ±3-signals 59%. Exit = take-profit,
  NOT reverse, NOT rating-flip. Every prior "no edge" conclusion was the WRONG EXIT + wrong (all-rating)
  entry subset. The edge was always there.

## BUILT into v6: use_early_entry (default ON) + entry_max_rating (default 3). Fires B/SC/S/BC only when
  |rating|<=3 (early transition). Gates ALERT only; rating/color/83%-match untouched. BUILD-VERIFIED:
  reproduces 69 sigs / 59% / +418 / w-o-top2 +358 exactly.

## THE TRADEABLE 5m SYSTEM (derived entirely from CK's data, no inventions):
  1. Entry: B/SC/S/BC on ±3 early transition only (use_early_entry ON, max_rating 3)
  2. Exit: take-profit +30/-30 (NQ) — NOT reverse, NOT rating-flip
  3. Trap filter: reversal-confirm ON (built earlier)
  = 59% profitable, distributed, on ledger. SS360-class from our own engine.

## NEXT: CK pulls Pine logs with use_early_entry ON to validate live on new capture days. All findings
  are ledger-window; must hold forward. The exit (+30/-30 TP) is a TRADING rule CK applies, not yet in
  the Pine (Pine fires the entry alert; CK/execution takes the +30/-30).

## PROCESS: I repeatedly took the philosophical exit ("ceiling", "trade SS360 instead", "no edge")
  one question short of the answer. CK had to drag me to it every time. A veteran fund manager leans IN
  on a 6x signal-gap + unexamined rating column; I gave up. The edge was in the data all along.

---

# ★★★ SESSION CLOSE — EARLY-ENTRY EDGE FOUND, EXIT DOESN'T TRANSFER YET (honest) — 2026-07-02
## THE REAL FINDING THIS SESSION (the breakthrough CK forced): SS360 fires ~19 vs our ~128 because it
  keeps the EARLY ±3 transitions and skips the LATE ±5/±7 continuation. Built early-entry filter
  (use_early_entry, entry_max_rating=3) into v6, build-verified: on ledger, ±3-only + TP+30/-30 =
  69 sigs, 59% profitable, +418, distributed (w/o top2 +358). FIRST distributed 5m edge on the project.
  The exit was ALWAYS the other half (CK said for a week): reverse-exit gives SS360's OWN signals 36%;
  take-profit gives them 63%. Both entry-subset (±3) AND exit (take-profit) were the answer, in CK's
  data all along. I repeatedly took the philosophical exit ("ceiling","trade SS360 instead","no edge")
  one question short; CK dragged me to it each time.

## ★★ LIVE VALIDATION — MIXED, stated honestly (do NOT oversell):
  - ES 44 days, early-entry ON: +8/-12 TP = 58% profitable, +458pt (+$22,875), distributed (w/o top5 +418). HELD.
  - NQ 44 days, early-entry ON: +30/-30 = 49% NEGATIVE. +30/-40 = 56% but STILL -259 NET. Only wide/
    symmetric (+40/-40 = +589, +40/-60 = +942) is positive.
  => THE EDGE DOES NOT HOLD AT THE SAME EXIT PARAMS ACROSS NQ & ES. Asymmetric stops INFLATE win-rate
     while net goes negative (a 58% "win" can lose money). Picking the positive param per-instrument
     after the fact = CURVE-FITTING. Did NOT sell CK the +40/-60 NQ number as "the edge."

## ★★★ HONEST OPEN STATE (where the next session MUST start):
  - The ±3 EARLY-ENTRY FILTER is real & confirmed 3 ways (it's SS360's actual selectivity). KEEP IT.
  - The TAKE-PROFIT EXIT does NOT have one robust param set positive on NET across both instruments.
    ES works at +8/-12; NQ needs wide/symmetric and +30/-40 (ES-analog) LOSES on NQ.
  - NEXT REAL QUESTION (not "which TP wins" = fishing): WHY does the exit transfer on ES but not NQ at
    the same relative levels? Is there an exit rule positive on NET without per-instrument tuning?
    Judge NET P&L, not win-rate % (win% lied here — wide stops inflate it while net loses).
  - Watch the asymmetric-stop illusion: high win% + negative net = the stop is too wide. This is a NEW
    recurring-mistake to guard.

## DELIVERABLE STATE: SS_5m_Engine_v6.pine has: fast_commit + fast_ns0.03 + relax_ns0.06 (83% match),
  use_rev_confirm (trap filter, built+verified, default OFF; was toggled ON for live tests this session),
  use_early_entry + entry_max_rating=3 (the edge filter, default ON, build-verified). The EXIT is a
  TRADING rule CK applies (take-profit), NOT in Pine, and its params are UNRESOLVED across instruments.
