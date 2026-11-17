# Strategy — GTA VI launch-day Twitch viewers forecast

## What the program does

`bot.py` (packed into `main.py` by the kit's `makefun compress`) reads one
case (`/input/<id>.in`) and writes `/output/<id>.out`: for hours 2–24 of
the release day, five quantiles (5/25/50/75/95%) of that hour's Twitch
concurrent viewers in the game's category. Pure standard library, no
network, deterministic, ~0.2 s per case.

## The model

For each past big launch in the case's `comparables`, take its release-day
hourly Twitch counts and form the log-ratio of each hour h = 2..24 to that
launch's hour-1 print. Per hour, regress that log-ratio on the log of the
launch's hour-1 print:

    log(viewers_h / viewers_1) = a_h + b_h * (log(viewers_1) - log(100000))

The forecast median is the fitted value at the case's own hour-1 print;
the five quantiles are the median plus the empirical 5/25/50/75/95%
quantiles of that hour's regression residuals across the comparables.
The fit is done at run time from the case file itself, so the final case
(with all 42 comparables, including the four practice launches) is
calibrated on its own data. The log-size axis is clamped to the
calibration range (hour-1 6,400–875,000) so a bigger launch is scored as
the biggest calibrated one, never extrapolated linearly.

## Why this shape

A launch-day curve is not flat. Across the comparables, hour 1 is a
midnight-UTC snapshot: only the regions already unlocked (New Zealand,
Australia, Asia, Europe, UK) are in. Viewership dips to ~0.7x in hours
3–4 (dead of night in the Americas), steps up at hours 5–6 (US East
unlock, 04:00–05:00Z), sags through the morning, then rises to an
evening peak (US daytime/evening) before easing at hours 23–24. The
slope b_h is near 0 early and about −0.3 to −0.5 late: a launch already
huge in hour 1 grows little afterwards, while a small hour-1 launch
grows several-fold. The house baseline (flat at hour 1, spread widening
as 0.15·sqrt(h−1)) misses all of this.

## Measured edge (practice cases, answers from SullyGnome.com)

Practice answers are the historical source's hourly counts for the four
practice launches (Marvel's Wolverine, Red Dead Redemption 2, God of War
Ragnarök, Marvel's Spider-Man 2), fetched once from SullyGnome's public
chart API; each case's hour-1 value matches its `.in` file exactly.
Pinball loss on the log scale, mean over 23 hours × 5 quantiles:

| case | model | flat baseline |
|------|-------|---------------|
| p00 Wolverine (h1 23,357) | 0.2475 | 0.7437 |
| p01 RDR2 (h1 116,550) | 0.1676 | 0.2868 |
| p02 GoW Ragnarök (h1 269,517) | 0.1172 | 0.2146 |
| p03 Spider-Man 2 (h1 119,087) | 0.1858 | 0.3033 |
| **pooled** | **0.1795** | **0.3871** |

54% below the baseline (the rules quote the baseline at "about 0.387";
this replay reproduces 0.3871, which validates the scoring code).
90% coverage on practice: 70–96% per case (mean ~85%); typical median
miss 18–47%. Run `python3 replay.py` to reproduce; `python3
tests/test_bot.py` runs the unit tests (6/6 pass).

## Entry / exit rules (what the code does, per case)

- Entry: hour-1 print p1 from `known`; the calibration set is selected
  from the case's comparables (see below); per-hour OLS on log-ratios.
- Exit (output): 23 rows, hours 2–24, quantiles non-decreasing,
  clamped to (0, 1e9], rounded to 2 decimals.
- Never crash a case: any failure in the calibrated forecast falls back
  to the house baseline for that case (a valid forecast always beats an
  invalid one, which scores as flat and takes no prize).

## Comparable selection (the filters matter more than the regression)

1. Skip the target's own pre-release category history (tier
   `target-history`) and the case's own launch.
2. Keep only launches already materially live at 00:00 UTC: hour-1 print
   ≥ 5% of that day's maximum. This drops fixed-time global unlocks
   (10:00 UTC+8, 14:00Z, ...) whose hour 1 is near-zero and whose
   ratios are meaningless. 8 of 37 dropped.
3. Drop early-access-contaminated launches (their "release day" is days
   after the real launch, so the curve is post-launch decay, not a
   launch day). 9 dropped; pooled loss 0.2155 → 0.1861.
4. Drop weak-tier launches (free-to-play, slow-burn). Pooled 0.1861 →
   0.1842 (normal spread) / 0.1795 (empirical spread).

19 comparables survive. The final case holds 42 (the 38 + the four
practice launches); the same filters select 22 there, and the p00 loss
moves only 0.2475 → 0.2496, so the fit is stable.

## Tried and dropped

- Pooled median curve (no size conditioning): pooled 0.2852. The
  size regression is the main edge.
- Normal (Gaussian) residual spread instead of empirical quantiles:
  pooled 0.1842 vs 0.1795. The residuals are left-skewed (bigger
  downside), so empirical quantiles win.
- Spread ×1.15: pooled 0.1814 (empirical) / 0.1933 (normal). No gain.
- Fitting only on big launches (h1 ≥ 100k, n=16): pooled 0.2610.
  The small launches carry the slope information.
- Blending the median toward the six biggest launches' median curve
  (for mega-launch extrapolation): pooled 0.3081 at 50% weight — much
  worse on practice. The regression with the x-clamp is kept instead.
- Theil–Sen robust fit: pooled 0.1810 vs OLS 0.1795. No gain.
- Dropping the two fixed-time outliers that survive the filters (Halo
  Infinite, campaign unlock 18:00Z; Resident Evil Requiem, PC unlock
  later): pooled 0.1934 vs 0.1795 — worse, because they help the
  smallest practice case. Kept.
- Console-only rolling subset (n=7): pooled 0.1970, and p00 blows out
  to 0.3798. Too small to trust.
- Conditioning on hour-1 channels, on the pre-launch baseline level,
  or on release weekday: no signal in the comparables (n too small,
  no stable pattern). Not used.
- `streams` / `categories_hourly`: null in every practice case, so
  nothing to calibrate against. Ignored by design.

## Expected outcomes

- Practice pooled pinball ≈ 0.18 vs house baseline 0.39 and a house
  model forecast of unknown strength. A paid place needs a final loss
  strictly below the better house entry on the single final case.
- The final is one case (GTA VI, hour-1 print likely far above the
  calibration range). The x-clamp keeps the median at the biggest
  calibrated launch's behaviour (evening peak ≈ 0.85–1.05× hour 1);
  the empirical spread (90% band roughly ×0.4 to ×2.4 at late hours)
  is the honest uncertainty. If hour 1 comes in near the top of the
  calibration range the forecast is on familiar ground; if it is far
  above (e.g. 2M+), the median is a deliberate, conservative
  no-extrapolation choice.
- Main risk: the final's true shape differs from every past launch
  (GTA VI is the largest launch ever; Twitch may behave differently
  at that scale, e.g. a bigger US-unlock step or a flatter evening).
  The wide empirical spread is the hedge; the median is the best
  calibrated guess.
