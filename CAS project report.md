# Indian CAS Synthetic-Close Transition — Project Report

**Subject:** How to predict and bridge synthetic close volume when NSE/BSE switch to the Closing Auction Session (CAS).
**Go-live:** 3 August 2026, first phase = cash-segment names with active derivative contracts.
**Status:** Consolidated working report. Supersedes earlier draft `cas_synthetic_volume_scaleback.md`, which contained an early (parallel-session) prediction now corrected.

-----

## 0. Executive summary

- Old close = VWAP of the last 30 min (3:00–3:30 IST). CAS replaces this for in-scope names with a single auction cross.
- Per the **official SEBI session table**, **3:15 PM IST is the transition from the continuous trading session to CAS**. The surviving continuous tail is **3:00–3:15**; the **3:15–3:30** slice no longer has a continuous venue and is handed to the auction.
- Model the old 30-min close pool as split: `alpha` stays in continuous (3:00–3:15), `(1−alpha)` goes to the auction.
- **Measured anchor (your data):** median over 60 days of vol(3:00–3:15) / vol(3:00–3:30) ≈ **0.40** for in-scope names. Under static reallocation this *is* alpha. So **alpha ≈ 0.40, auction share ≈ 0.60** is the central, data-grounded estimate.
- **First-3-day adjustment:** day-1 `alpha` will sit **above** 0.40 (auction share **below** 0.60). The decisive evidence is **Shanghai** — the only major *truncating* analogue (it abolished the last 3 min of continuous trading in 2018, as India deletes 3:15–3:30) — where volume shifted **into** the surviving pre-auction continuous minutes. Day-1 seed: **alpha ≈ 0.44, auction share ≈ 0.56** (range 0.43–0.45 / 0.55–0.57). The excess over 0.40 is *small* because India’s in-scope universe is liquid-only, which mutes pull-forward and amplifies the offsetting close-seeking inflow.
- **Long-run:** `alpha` drifts **below** 0.40 (auction share above 0.60) as close-seeking passive flow ramps. The measured 40% is the long-run gravitational center; day-1 is above it, steady-state below it.
- **W (total close-window volume): assume constant for the first 3 *normal* days only**, and use it only to fix the split (ratio), not to pin absolute auction volume. Void on rebalance/expiry days.

-----

## 1. Confirmed CAS design (from official SEBI documents)

Source: SEBI consultation papers (Dec 2024, Aug 2025) and circular dated 16 Jan 2026; BSE/NSE reiterations. Use the **final** column; some press coverage quotes the stale Dec-2024 draft.

|Parameter          |Stale draft (ignore)         |**Final (use this)**                                            |
|-------------------|-----------------------------|----------------------------------------------------------------|
|Session            |15 min, after hours 3:30–3:45|**20 min, 3:15–3:35 PM IST**                                    |
|Phase 1            |—                            |**Ref-price calc / transition from CTS to CAS: 3:15, 5 min**    |
|Phase 2            |—                            |Order entry, limit+market: 3:20, 5 min                          |
|Phase 3            |—                            |Order entry, limit only + random close (last 2 min): 3:25, 5 min|
|Phase 4            |—                            |Order matching / cross: 3:30, 5 min                             |
|Price band         |±5%                          |**±3%** of reference price (fixed)                              |
|Reference price    |VWAP 3:00–3:30               |**VWAP 3:00–3:15**; fallback LTP, then prior close              |
|Order priority     |limit > market               |**market > limit** (reversed)                                   |
|Carried-over orders|all                          |**unexecuted limit only; cannot modify, only cancel**           |
|Order types        |—                            |limit + market only; **no stop-loss, no iceberg**               |
|Closing price      |—                            |**auction equilibrium cross** (max-matched volume)              |
|Post-close session |—                            |cash post-close ~3:50–4:00 at the closing price                 |
|Scope (phase 1)    |Nifty50/Sensex30             |**all derivative-segment (F&O) names**; others keep VWAP        |

**Key reading of the 3:15 transition:** continuous trading feeds the reference window up to 3:15; at 3:15 the book transitions into CAS. So 3:00–3:15 is the surviving continuous tail and 3:15–3:30 is replaced by the auction process. This is the truncation structure the alpha model assumes — confirmed, not assumed.

-----

## 2. The model

Let the old close window **W = 3:00–3:30**, split at 3:15 into tail **T = 3:00–3:15** and replaced slice **M = 3:15–3:30**.

- `alpha` = share of W that remains in continuous trading = vol_new(T) / vol_new(W)
- `(1−alpha)` = share that prints in the CAS auction
- Pattern before 3:00 assumed unchanged.

**Validity:** under the confirmed 3:15 transition this is structurally exact (T and M are disjoint, M loses its continuous venue), unlike a parallel-session reading. Two standing caveats: alpha is heterogeneous across stocks; rebalance/expiry days are a separate regime.

-----

## 3. The anchor: why alpha ≈ 0.40 (your measurement)

Measured: median[ vol(T)/vol(W) ] ≈ 0.40 over 60 days for in-scope names ⇒ back half M ≈ 0.60. This is back-loading (the last 15 min is heavier), quantified.

Under **static reallocation** (T prints exactly as today; all of M migrates to the auction; nothing added or lost), the identity gives **alpha = 0.40 exactly**. So 0.40 is not a guess — it is the measured split, and it is the correct central anchor.

-----

## 4. Why day-1 alpha will EXCEED 0.40 (formal, with the correct analogue)

Decompose the day-1 continuous tail:

`vol_new(T) = vol_today(T) + [pull-forward from M] + [stop-loss/iceberg re-routed from M] − [close-seeking flow leaving T for the auction]`

The first three terms are positive; the last is the offsetting downward force. The net sign is an empirical question — and it must be settled by the analogue with the **same structure as India**, not just any closing-auction market.

**The structural fork — additive vs truncating — determines the sign, so pick the right analogue:**

- **Additive design (e.g. Hong Kong 2016):** continuous trading was left intact to 16:00 and the auction was *bolted on after*. The auction had to *win* volume by attraction, so volume *leaked out* of the (still-existing) continuous tail → continuous-tail share **fell** (alpha down). **This is NOT India’s structure** and must not be used to sign India’s alpha.
- **Truncating design (Shanghai 2018, and India 2026):** the final continuous minutes are *abolished* and replaced by the cross. The deleted flow is forced somewhere, and a chunk pulls forward into the *surviving* pre-auction continuous minutes → pre-auction continuous volume **rises** (alpha up).

**Shanghai is the decisive evidence (the matching truncating case).**
On 20 Aug 2018 SSE changed the last three minutes of continuous trading to a call auction — structurally identical to India deleting 3:15–3:30. Peer-reviewed difference-in-difference studies (2017–2019 A-share sample) find:

- a *significant shift of trading volume from the closing call into the preceding continuous trading*;
- specifically, closing volume *shifts toward the 3-minute interval immediately before the call* — i.e. into the surviving continuous tail;
- accompanied by *increased volatility at pre-closing* — the classic impatience/pull-forward signature (traders crowding the surviving minutes to secure fills before the single cross).

This confirms the sign of the India argument: under truncation, vol_new(T) > vol_today(T), so day-1 alpha > 0.40.

**The four forces, now correctly weighted:**

1. **Impatience / pull-forward (up) — confirmed by Shanghai.** The cross has execution uncertainty (±3% band rejection, pro-rata fills, random close) and no track record at launch; the surviving tail offers guaranteed immediacy. Fill-seeking flow pulls forward into ~3:10–3:15 (inside T). Strongest at launch, decays as trust builds.
1. **Barred order types (up) — India-specific, structural.** CAS bars stop-loss and iceberg; that flow can’t be expressed in the auction and re-routes into T or disperses. Present from minute one.
1. **Close-seeking inflow (down) — the brake.** Net-new passive/index volume executing at the official close. HK shows this force can be *fast* in passive-heavy markets (HK’s auction matched the last-15-min continuous volume within ~a year). India is passive-heavy too (SEBI: ~29% of FPI / ~27.5% of domestic-MF equity AUM), so this brake is real and limits how far above 0.40 alpha goes.
1. **Universe composition (caps the excess).** Shanghai’s pull-forward effect was *strongest in small-caps*. India’s in-scope set is the opposite — liquid derivative names only — exactly where close-seeking inflow (force 3) is strongest and impatient pull-forward (force 1) is weakest. So India’s *net* upward push is **smaller** than Shanghai’s all-share average.

**Conclusion:** day-1 **alpha ≈ 0.43–0.45 (seed 0.44)**, auction share ≈ 0.55–0.57. Direction (above 0.40) is robust — anchored on Shanghai, the correct truncating analogue. Magnitude of the excess is small and capped by the liquid-only universe; downside toward 0.40 is live only if close-seeking adoption is unusually fast on day one (HK’s caution).

**Note on an earlier error:** a prior draft of this report leaned on US/Singapore “auctions start thin” evidence and then over-corrected on Hong Kong. Both are *additive*-design markets and the wrong template for signing India’s alpha. Shanghai (truncating) is the correct analogue and restores the day-1 alpha > 0.40 conclusion, with HK retained only as (a) the manipulation cautionary case and (b) evidence that the downward close-seeking force is fast in passive-heavy markets.

-----

## 5. Time path: U-shaped around the measured 40%

- **Day 1–3:** alpha above 0.40 (forces 1–2 maximal, force 3 minimal).
- **Weeks–months:** alpha falls back through 0.40 as the auction earns trust (force 1 decays), desks re-tool order types (force 2 decays), and close-seeking flow ramps (force 3 grows).
- **Steady state:** alpha plausibly **0.30–0.40**, auction share 0.60–0.70+, with a higher ceiling than a naive “small passive base” guess because Indian passive penetration is already material (SEBI: ~29% of FPI equity AUM, ~27.5% of domestic MF equity AUM; Indian index weights 17–30% in FTSE/MSCI).

The measured 40% is the **gravitational center**: day-1 sits above it on the alpha axis, steady-state below it.

-----

## 6. Is total close-window volume W the same?

Not exactly. W = relocated volume (conserved by construction; where the 40% lives) + pull-forward in (regime 2, launch-heavy) + close-seeking inflow (regime 3, ramps over years) − leakage (e.g. barred stop-loss flow dispersing earlier).

- **Long run W grows** (close-seeking inflow), which is also why alpha drifts below 40%. A fixed-W model under-counts auction volume as adoption matures.
- **Assume W constant for the first 3 NORMAL days only.** Justification: at launch both growth terms are near their minimum, so the relocation term dominates and W ≈ today’s W. The approximation is least wrong exactly in the bootstrap window where it’s used.
- **Use W-conservation only to fix the split (alpha / ratio), not to pin absolute auction volume.** The ratio is robust to small W errors (numerator and denominator move together); the absolute level inherits the full W uncertainty.
- **Void on rebalance/expiry days** — close-seeking inflow spikes by design (SEBI’s overnight-borrowing provision exists for rebalance days). Check whether any of the first 3 sessions from Aug 3 is a monthly expiry or index-reconstitution date and exclude it from the conserved-W treatment.

-----

## 7. Synthetic-close bridge (the original project question)

1. Store real CAS price + volume **unmodified**, tagged `source=CAS_REAL`, from Aug 3. Never scale the real print.
1. Tag the Aug-3 regime break for in-scope names: `close_method: SYNTH_VWAP_30M → CAS_AUCTION`.
1. **Confirm (blocking):** does any downstream consumer splice synthetic-then-real into one continuous lookback series? If no → no scale-back; rely on the regime flag.
1. If yes → bridge the **legacy synthetic** series only. Continuity should be assessed on the **end-of-day total** (T continuous tail + auction cross), not the auction print alone, or you manufacture an artificial downward step.
1. Per-stock, not one global constant: use each name’s own measured T/W ratio as its alpha baseline, plus a uniform **launch premium** (seed alpha = own-ratio + ~0.04) that decays over the first weeks toward the bare ratio.
1. Day-type regimes: normal vs index-rebalance vs expiry; never average across them.
1. Define the close-volume field explicitly: auction cross only, or auction + post-close session — they are different quantities.
1. Mock sessions (NSE’s 2–3 pre-launch) test plumbing, not liquidity — infer no volume prior from them.
1. Retire W-conservation and the launch premium after the bootstrap window; let W grow and refit per-stock from real normal-day prints.

-----

## 8. Numbers to seed the system

|Quantity                         |Day-1 seed    |Range    |Steady-state      |
|---------------------------------|--------------|---------|------------------|
|alpha (continuous share of old W)|**0.44**      |0.43–0.45|0.30–0.40         |
|auction share (1−alpha)          |**0.56**      |0.55–0.57|0.60–0.70+        |
|W (total close-window volume)    |assume = today|±small   |grows (do not fix)|

Per-stock: alpha_i(day1) = (own 60-day median T/W) + ~0.04 launch premium, decaying to own ratio over ~2–4 weeks. Liquid index heavyweights: lower baseline T/W, smaller launch premium (close-seeking inflow dominates there), and faster decay below baseline.

-----

## 9. Open items

- **Blocking:** downstream splicing usage (drives whether any scale-back is needed at all).
- Close-volume field definition (cross only vs +post-close).
- First-3-day calendar check (expiry/rebalance exclusions).
- Per-stock T/W ratios extracted from the existing 60-day history (you have this data).

-----

## 10. Cross-region comparison

SEBI’s consultation paper cites **NYSE, LSE, Euronext, HKEX, ASX** as the global benchmarks it is aligning with (Paris is not named separately — it sits inside Euronext). Notably SEBI does **not** cite **Shanghai/Shenzhen**, even though China is the closest *structural* match to India’s design. The table below compares on the axes that actually drive day-1 alpha, not on headline “has an auction / doesn’t.”

### 10.1 Comparison table

|Market                    |Launch (relaunch)                                |**Continuous tail: truncated or kept?**                        |Auction window         |Reference / price-control                       |Mature auction share (of daily vol)        |Day-1 / early volume behaviour                                                                     |Passive-heavy?            |
|--------------------------|-------------------------------------------------|---------------------------------------------------------------|-----------------------|------------------------------------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------|--------------------------|
|**India (CAS)**           |**3 Aug 2026**                                   |**Truncated** — CTS→CAS transition at 3:15; 3:15–3:30 abolished|3:15–3:35 (cross ~3:30)|VWAP 3:00–3:15; **fixed ±3%** band              |unknown (target)                           |unknown — **this report predicts it**                                                              |Yes (~27–29% AUM)         |
|**Shanghai (SSE)**        |20 Aug 2018                                      |**Truncated** — last 3 min CTS → call                          |14:57–15:00            |max-volume cross                                |small/modest                               |volume shifted **into** pre-auction CTS minutes; vol-up pre-close; strongest in small-caps         |Moderate, growing         |
|**Shenzhen (SZSE)**       |2006                                             |**Truncated** — last 3 min                                     |last 3 min             |max-volume cross                                |small/modest                               |(long-established; same family as SSE)                                                             |Moderate                  |
|**Hong Kong (HKEX)**      |2008 → **relaunch 2016**                         |**Kept** — CTS intact to 16:00; auction **bolted on after**    |16:00–16:08/10         |±5% band, reference-price controls, random close|auction ≈ last-15-min CTS volume by Q2 2017|volume **leaked out of** CTS tail into auction (alpha down); 2008 version scrapped for manipulation|Yes (index/ETF-heavy)     |
|**Euronext (incl. Paris)**|Paris closing auction 1996/1998                  |**Kept** — auction after continuous close                      |post-close call        |call-auction, indicative price dissemination    |~20%+ of day; CAC 40 up to ~41%            |grew over **years**; voluntary migration                                                           |Yes                       |
|**LSE**                   |long-established                                 |**Kept** — auction after close                                 |post-close call        |call-auction                                    |high; benchmark for funds                  |grew over years                                                                                    |Yes                       |
|**NYSE / Nasdaq**         |closing auction 2004 (Nasdaq); NYSE long-standing|**Kept** — auction at/after 16:00                              |at close               |D-orders, imbalance feeds                       |~10% (S&P 500), reached over ~a decade     |3%→10% over ~8 yrs; indexing-driven                                                                |Yes (largest passive base)|
|**Australia (ASX)**       |1997 (reformed 2002)                             |**Kept** — auction after close                                 |post-close call        |call-auction (reformed for manipulation 2002)   |modest                                     |gradual drift into auction                                                                         |Moderate                  |

### 10.2 Similarities and dissimilarities vs India

**On market microstructure:**

- *Similar (passive/index pull):* NYSE, LSE, Euronext, HKEX, ASX and India are all institutionally/passive-heavy, so they share the structural driver that grows auction volume over time (funds wanting execution at the official close). This is the axis SEBI emphasises and the basis for the *long-run* alpha drift below 40%.
- *Dissimilar (universe at launch):* India’s phase-1 universe is **liquid derivative names only**. Shanghai’s effect was strongest in *small-caps* — the opposite end. So India will see a *weaker* pull-forward and a *stronger* close-seeking-inflow brake than Shanghai’s all-share averages. This is why India’s day-1 excess over 40% is small.
- *Dissimilar (price control):* India’s **fixed ±3%** band is tighter and non-flexing vs HKEX’s ±5% and Euronext/NYSE dynamic mechanisms. Tighter band ⇒ more day-1 order rejection ⇒ auction capture capped early.

**On CAS dynamics (the decisive axis — truncated vs additive):**

- *Similar (truncating):* **Only Shanghai and Shenzhen** match India — the final continuous minutes are *abolished* and replaced by the cross. This forces the deleted flow somewhere and produces the pull-forward-into-surviving-CTS signature.
- *Dissimilar (additive):* **NYSE, LSE, Euronext, HKEX, ASX — every market SEBI actually cites — are additive.** The continuous session is kept and the auction is added on top. There, the auction must *win* volume by attraction, so the continuous tail *empties* (alpha down) rather than the pre-auction minutes filling (alpha up). Using any of these to sign India’s day-1 alpha gives the wrong direction.

### 10.3 The closest analogue, and why

**Shanghai (SSE 2018) is the closest analogue to Indian CAS.** Reasoning, in priority order:

1. **Identical structural mechanic (decisive).** SSE truncated the last 3 minutes of continuous trading and replaced them with a single call cross — exactly what India does by moving the CTS→CAS transition to 3:15 and abolishing 3:15–3:30. This is the *only* axis that flips the *sign* of day-1 alpha, and Shanghai is the only liquid, cited-or-not major market that shares it. Every market SEBI names (NYSE/LSE/Euronext/HKEX/ASX) is additive and therefore signs alpha the wrong way for India.
1. **Same alpha direction, empirically measured.** Shanghai’s peer-reviewed DiD studies found volume shifting *into* the surviving pre-auction continuous minutes with elevated pre-close volatility — the precise behaviour the India model predicts. No additive market shows this; they show the reverse.
1. **Comparable emerging-market microstructure.** Retail-heavy participation, evolving regulation, and a still-maturing passive base resemble India more than the developed additive markets do.
1. **Same regulatory motive and safeguards.** Manipulation-resistant design (single cross, no time/price priority gaming, random-close family) aimed at curbing closing-price manipulation — the same rationale SEBI gives.

**Why not the markets SEBI cites:** SEBI’s list (NYSE/LSE/Euronext/HKEX/ASX) is the right reference for the *policy goal* — “align with global practice, reduce tracking error, serve passive funds.” It is the **wrong** reference for *predicting day-1 volume mechanics*, because all five are additive-design auctions. For the alpha question specifically, Shanghai/Shenzhen — which SEBI does not cite — are the correct technical analogue. Use SEBI’s list to justify *why CAS exists*; use Shanghai to predict *how the volume splits on day one*.

**Caveat on Shanghai’s transferability:** the match is structural, not universe-matched. Shanghai’s documented pull-forward was strongest in small-caps, whereas India launches on liquid names only — so apply Shanghai’s *sign* (alpha > pre-period split) but *discount its magnitude*, which is exactly what the day-1 seed of 0.44 (just above the measured 0.40) does.