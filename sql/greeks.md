# Option Greeks — coverage, provenance, and the computed approximation

`tcg_instruments.fact_option_greeks` holds one greeks row per `(option instrument_id, trade_date)`. **Not every option-bar has one**, and the greeks that exist do **not** all come from the same source. This page explains what is present, what is missing, why the missing ones cannot be exactly reconstructed, and how TCG fills the gaps with a clearly-marked computed approximation.

> **TL;DR.** The loaded greeks are the **IVOLATILITY vendor's** own numbers (a third-party feed), on an undocumented `r ≠ 0` convention. We can reproduce their **delta and implied vol**, but **not** their gamma/vega/theta (that needs their exact rate + day-count, which we don't have). VIX, ETH, and part of BTC arrived with **no greeks at all**. For those we **compute an approximation** (Black-76 on the matching future, fixed rate), store it tagged `greek_source = 'computed'`, and treat it as an **estimate, not a reconstruction** of the vendor.

---

## 1. Coverage — who has vendor greeks, who doesn't

| Option family (`root_symbol`) | Vendor greeks? | Notes |
|---|---|---|
| `IND_SP_500`, `IND_NASDAQ_100`, `GOLD`, `T_NOTE_10_Y`, `T_BOND`, `EURUSD`, `JPYUSD` | **Full** | IVOLATILITY feed |
| `BTC` | **Partial** | greeks on some days, missing on others |
| `IND_VIX` | **None** | quotes-only — 0 greek rows |
| `ETH` | **None** | quotes-only (Deribit feed was never wired for greeks) |

Quick check of what a given root actually has:

```sql
SELECT d.root_symbol,
       count(*) FILTER (WHERE g.instrument_id IS NOT NULL) AS greek_rows
FROM tcg_instruments.dim_instrument d
JOIN tcg_instruments.fact_price_eod p   ON p.instrument_id = d.instrument_id
LEFT JOIN tcg_instruments.fact_option_greeks g
       ON g.instrument_id = p.instrument_id AND g.trade_date = p.trade_date
WHERE d.asset_class = 'option' AND d.root_symbol = :root
GROUP BY 1;
```

A `v_option_chain` row with quotes but NULL `delta…theta` is an option-bar with **no greeks** — expected for the families above, not a data error.

---

## 2. Provenance — the loaded greeks are a third party's

The greeks in the table were **not computed by TCG**. They were read verbatim from the **IVOLATILITY** end-of-day feed (the legacy Java `integration.parent/ivolatility` module ingests `iv, delta, gamma, vega, theta` as feed fields) and loaded as-is. IVOLATILITY computes them with a **non-zero risk-free rate and its own day-count convention**, neither of which is documented anywhere we hold.

(The legacy Java `compute.parent/greeks.core` module is a *portfolio/strategy* delta aggregator — it does **not** produce these per-contract greeks. The only TCG-authored per-contract option pricer is the current `tcg.engine.options.pricing` engine, which uses `r = 0` — see §4.)

---

## 3. Why the vendor greeks cannot be exactly reproduced

We validated a correct Black-76 pricer against the vendor rows on `GOLD` and `IND_SP_500` (which have full vendor greeks). Result:

- **Delta and implied vol — reproducible.** They barely depend on the rate; delta matched within tolerance.
- **Gamma, vega, theta — not reproducible to the vendor's numbers.** Reproducing them needs IVOLATILITY's exact `r` *and* theta day-count. The divergence is irreducible: even when the pricer is fed the vendor's *own* implied forward, theta still differs by ~130% — that residual is purely the rate/day-count recipe, not a pricing bug (the kernel reproduces TCG's engine to 1e-9).

Reverse-engineering those constants by fitting to the existing numbers is possible in principle but is guesswork against an undocumented third-party pipeline, and the result would then disagree with TCG's own engine. **We do not attempt it.** It also doesn't help the gap families — VIX/ETH have no vendor numbers to fit against.

---

## 4. The computed approximation (`greek_source = 'computed'`)

Where greeks are missing, TCG computes an **estimate** so the chains are not empty, and marks it so it is never confused with vendor data.

- **Model:** Black-76 (option on a forward), **European**, calendar-day TTM/365, IV inverted from the option's mid price. This reuses the production `tcg.engine.options.pricing` kernel.
- **Rate:** a **fixed rate** (the engine's `r = 0` convention by default; a non-zero fixed constant may be used per family where it materially improves the theta estimate — recorded with the data).
- **Underlying / forward:**
  - **VIX** → the matching-expiry `FUT_VIX` future close (VIX options are settled on the VIX **future**, not the index). VIX **weeklies** with no matching monthly future are **left as a documented gap** in the first pass.
  - **Crypto (ETH, BTC gaps)** → the front `FTS_{BTC,ETH}_USD` future close (spot `BTC_USD`/`ETH_USD` as fallback). Deribit premiums are quoted **in the coin** — they are **dollarized** (× coin/USD spot) before inversion so premium and forward share units.

**These are estimates, validated by construction, not reconstructions of the vendor.** The pricer is proven mathematically correct (golden-tested against the engine); for the gap families there is no vendor reference, so correctness is established by sanity properties — ATM delta ≈ ±0.5, gamma/vega > 0, put/call sign, positive finite IV — rather than by matching a vendor number. Delta and IV are comparable to vendor conventions; **gamma/vega/theta are on the house `r`=fixed convention and will not be on the same basis as the IVOLATILITY rows for the vendor-greeked families.**

---

## 5. Telling them apart — and a caution

Every greeks row carries `greek_source`: `'vendor'` (IVOLATILITY, loaded) or `'computed'` (TCG estimate). Filter on it whenever the distinction matters:

```sql
-- only vendor greeks:
SELECT * FROM tcg_instruments.fact_option_greeks WHERE greek_source = 'vendor';
-- only TCG-computed estimates:
SELECT * FROM tcg_instruments.fact_option_greeks WHERE greek_source = 'computed';
```

**Caution — do not mix conventions for higher-order, cross-family analysis.** A theta-harvesting or gamma/vega study that spans, say, GOLD (vendor, `r ≠ 0`) and VIX (computed, fixed `r`) is comparing two conventions. Delta/IV are safe to mix; gamma/vega/theta are not. Slice by `greek_source` (and ideally by family) for those.

---

## 6. Status

| Family | Plan |
|---|---|
| `IND_VIX` monthlies | computed approximation (Black-76 on `FUT_VIX`, fixed rate) |
| `IND_VIX` weeklies | documented gap (no matching monthly future) — deferred |
| `ETH`, `BTC` gaps | computed approximation (front crypto future, dollarized premium) |
| vendor-greeked families | unchanged (`greek_source = 'vendor'`) |

Coverage of the computed fill is reflected by the `greek_source = 'computed'` rows; re-run the §1 query to see current state.
