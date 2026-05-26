
# Outlier Weekly — Methodology

## Three formulas, one weighting, applied to rare-event prediction markets.

This page is the open methodology behind every issue of Outlier Weekly. The newsletter applies a synthesized three-formula system — Poisson distributions, Shannon entropy / KL divergence, and Taleb fat-tail asymmetry, weighted 40 / 35 / 25 — to live Polymarket and Kalshi markets, with every issue showing the inputs, the math, and the forward-tracked outcome.

The methodology is open by design. The integration — the 40 / 35 / 25 weighting and the override gates around it — is the contribution. Each component formula is well-established in the academic literature and is linked to its primary source. If you read the math here and decide it is not for you, that is a fair conclusion to reach before subscribing.

**Forward-track (live ledger):** Every issue’s call — including SKIPs — is logged on the [forward-track page](/outlier-weekly-methodology/forward-track/) on publish day. As of Issue 2 (2026-05-26): two consecutive skip-discipline entries (Anthropic–Pentagon regulatory binary; Iran airspace geopolitical binary).

**Publish discipline (updated Issue 2):**

- Re-run the full synthesis if YES moves **≥3 percentage points** between draft and publish lock.
- Log **SKIP** when weighted output is **< ~0.5% of bankroll** even if directionally non-zero (noise-floor / actionable-minimum rule).
- Show both draft and publish-day prices on the forward-track when they differ.

---

### Why a three-formula synthesis instead of one indicator?

Most prediction-market analysis uses one indicator at a time: volume, raw historical frequency, simple Bayesian updates. That works on coin-flip-shaped markets where the price is already close to fair. It misses systematically on rare events — the long-tail outcomes where mispricing is largest.

Three independent frameworks, each capturing what the others miss, combined in a weighted ensemble that degrades gracefully when one input is missing:

- **Poisson distributions** — frequency-driven base rates. The right tool when a market asks *"will rare event X occur ≥N times before date Y?"* and the per-period rate is approximately memoryless and independent.
- **Shannon entropy and Kullback–Leibler (KL) divergence** — information geometry. KL between your subjective probability and the market's implied probability is the edge expressed in bits, with a threshold table that prevents over-trading on noise.
- **Taleb fat-tail asymmetry** — barbell-portfolio asymmetric sizing. A 1.5x tail-bucket multiplier captures the structural asymmetry of fat-tailed markets where single headlines re-price overnight.

Sized inside each component via quarter-Kelly, the operational default that captures roughly half of full-Kelly's long-run growth at roughly a quarter of its volatility.

---

### Poisson edge — Formula 1 of 3

The Poisson distribution models the probability of at least one rare event occurring in a fixed window, given a historical rate. The formula:

```
P(X ≥ 1) = 1 − exp(−λT)
```

Where `λ` is the historical rate per period and `T` is the window length. Implementation:

```python
import math

def polymarket_poisson_edge(
    event_name: str,
    historical_rate: float,      # λ — events per period
    market_price_cents: int,     # Polymarket YES price (0–100)
    time_periods: int,           # T — window length
) -> dict:
    poisson_prob = 1 - math.exp(-historical_rate * time_periods)
    fair_value_cents = poisson_prob * 100
    edge_cents = fair_value_cents - market_price_cents
    full_kelly = abs(edge_cents) / 100
    qk = full_kelly / 4

    if edge_cents > 1:
        direction = "BUY_YES"
    elif edge_cents < -1:
        direction = "BUY_NO"
    else:
        direction = "SKIP"

    return {
        "event": event_name,
        "poisson_probability": poisson_prob,
        "fair_value_cents": fair_value_cents,
        "edge_cents": edge_cents,
        "quarter_kelly_fraction": qk,
        "direction": direction,
    }
```

**Sanity-check rules** (the operational guard layer):

1. `λ` must come from genuine historical data, not back-of-envelope. The single largest failure mode is fabricated rate estimates.
2. `T` must match the market resolution window, not a convenient round number.
3. If `|edge| < 1¢`, SKIP — within bid/ask noise and execution slippage.
4. If `|edge| > 30¢` ("too much edge"), do not trust the calculation — either `λ` is wrong or the market is pricing information the model does not see.

**Worked example.** Market: "Will there be ≥1 magnitude-7+ earthquake in country X in next 30 days?" Historical rate 1 such event per 90 days → `λ = 1/90` per day. Window 30 days → `T = 30`. Market YES price 25¢.

Poisson probability: `1 − exp(−30/90) ≈ 0.283` → fair value 28.3¢. Edge: 28.3 − 25 = +3.3¢ (BUY_YES). Quarter-Kelly: `(3.3/100) / 4 ≈ 0.0083` → 0.83% of bankroll on YES.

Primary source: the Poisson distribution itself dates to Poisson (1837) and was famously calibrated against rare-event data by Bortkiewicz (1898). The Cemini methodology adapts these classical foundations to prediction-market sizing via the quarter-Kelly criterion.

---

### Shannon entropy + KL divergence — Formula 2 of 3

Claude Shannon's 1948 paper defined the entropy function on probability distributions: *"The fundamental problem of communication is that of reproducing at one point either exactly or approximately a message selected at another point."* [Source: Shannon, 1948, *A Mathematical Theory of Communication*, Bell System Technical Journal §1, p. 379]. KL divergence (Kullback and Leibler, 1951) extends this to the *distance* between two probability distributions in bits of information.

For prediction-market edge:

```
KL(p_mine || p_market) = p_mine · log₂(p_mine / p_market)
                       + (1 − p_mine) · log₂((1 − p_mine) / (1 − p_market))
```

Operator threshold table:

| KL bits | Action | Rationale |
|---------|--------|-----------|
| < 0.05 | SKIP | Edge is in the noise; transaction costs eat it |
| 0.05 – 0.10 | WEAK | Small position only |
| 0.10 – 0.20 | TRADE | Normal position size |
| 0.20 – 0.30 | STRONG TRADE | Above-average size |
| > 0.30 | RECHECK | Suspiciously large edge — your input is probably wrong |

The >0.30 RECHECK gate is the operational humility rule. A quarter-bit-of-information edge over a liquid market is enormous. The most-likely explanation when the model claims it is a model bug; the second-most-likely is the market is pricing news the model has not seen.

**Worked example.** Market: "Will Fed cut rates in September 2026?" Your model: 60% (0.60). Market price: 45¢ (0.45). KL(0.60 || 0.45) ≈ 0.065 bits → WEAK bucket → small position on YES.

KL is *sizing-only*. Entry price comes from the market's bid/ask plus slippage budget; KL tells you how much to put on.

---

### Taleb fat-tail asymmetry — Formula 3 of 3

Nassim Taleb's *Antifragile* (2012) and *The Black Swan* (2007) argue that when returns are fat-tailed, the mean is dominated by rare extreme outcomes. The optimal portfolio is *not* mid-conviction across the universe — it is a barbell: a large allocation to very safe positions plus a small allocation to very asymmetric positions. The middle is where capital dies on a fat-tailed distribution.

Default split: **85% safe, 15% tail**.

What makes a market "tail" versus "safe":

| Bucket | Polymarket examples |
|--------|---------------------|
| Safe | Liquid binary markets near 50/50 with clear resolution; markets near expiry with locked-in outcome |
| Tail | Long-tail markets with months-out resolution; markets with possible-but-rare extreme outcomes; markets near 1% or 99% (the cheap-contracts regime); regulatory binaries with single-headline re-pricing capacity |

Inside each bucket, individual positions are sized via quarter-Kelly. The barbell handles asset allocation; the Kelly criterion handles position sizing within the allocation.

Quantitative classifier: markets with excess kurtosis > 3 (total kurtosis > 6) or Hill-estimator tail-index α below threshold go into the tail bucket. In practice, regulatory binaries and crypto-extreme markets default to tail; near-resolution coin-flip binaries default to safe.

**Outlier Weekly uses a 1.5x multiplier for tail-bucket markets and 1.0x for safe-bucket markets.** This is a size factor, not a directional signal — it scales the position produced by the Poisson and Shannon components.

---

### Quarter-Kelly sizing — the universal primitive

Every position size in this methodology is fractional-Kelly at the *quarter* level. Why not full-Kelly:

1. **Model error** — `p_mine` is a guess. If true `p` is 5 percentage points lower, full-Kelly is over-betting.
2. **Fat tails** — Polymarket has resolution disputes, oracle delays, on-chain settlement quirks. Full-Kelly under fat tails has worse drawdowns than log-utility math predicts.
3. **Regime change** — parameters change. A `p` calibrated on six months may not hold for the next six.
4. **Concurrent positions** — Kelly is single-bet. With many simultaneous positions, the sum of full-Kelly fractions overshoots true portfolio-Kelly.

Empirical observation across decades of fractional-Kelly research: half-Kelly captures about 75% of full-Kelly's growth rate with about 50% of the volatility; quarter-Kelly captures about 50% of the growth with about 25% of the volatility. Quarter is the operational default.

A hard per-position cap (default 5% of bankroll) sits on top of every Kelly output as the final safety layer.

---

### The synthesis — 40 / 35 / 25 weighting

```python
def complete_rare_event_system(
    event_name: str,
    historical_rate: float,
    operator_estimate: float,      # your subjective probability
    market_price_cents: int,
    time_periods: int,
    safe_or_tail_bucket: str,      # "safe" | "tail"
    bankroll: float,
) -> dict:
    poisson_result = polymarket_poisson_edge(
        event_name, historical_rate, market_price_cents, time_periods
    )
    estimate_kl_bits = kl_divergence(operator_estimate, market_price_cents / 100)
    estimate_size = kl_bucket_to_size(estimate_kl_bits)
    tail_multiplier = fat_tail_size_multiplier(safe_or_tail_bucket)

    weighted_fraction = (
        0.40 * poisson_result["quarter_kelly_fraction"]
        + 0.35 * estimate_size
        + 0.25 * tail_multiplier
    )
    final_fraction = min(weighted_fraction, 0.25)

    return {
        "event": event_name,
        "final_fraction": final_fraction,
        "position_usd": bankroll * final_fraction,
        "components": {
            "poisson": poisson_result,
            "kl_bits": estimate_kl_bits,
            "bucket": safe_or_tail_bucket,
        },
    }
```

**Why these weights:**

- **40% Poisson** — highest weight because Poisson is grounded in historical data, not operator opinion. When the Poisson signal is strong, it is load-bearing.
- **35% operator estimate (KL bits)** — second-highest. The operator's number sized in information units, not raw probability differences.
- **25% fat-tail adjustment** — smallest because it is a multiplier on the asset-allocation decision (safe vs tail bucket), not a directional signal. It modifies size, not direction.

The 40 / 35 / 25 split is the starting point, not a fitted constant. As forward-tracked data accumulates, the weights are reviewed — not in the hot loop, but as documented operator-config changes.

**Override gates (applied after weighted output):**

1. KL bits > 0.30 anywhere → RECHECK; override → 0 position.
2. |Poisson edge| > 30¢ → RECHECK; override → 0 position.
3. Insider entropy collapse with no news → STAY OUT; override → 0 position.
4. Per-position hard cap (default 5%) — applied after all weighted math.

---

### Why combine instead of pick-the-best?

A pick-the-best approach (use whichever single formula has the highest signal) over-trusts whatever is loudest. Combining:

- **Down-weights single-source signals** — if only Poisson fires, the weighted fraction stays small.
- **Up-weights agreement** — when Poisson and KL fire on the same side, weights compound.
- **Allows graceful degradation** — if one input is unavailable (e.g. no kurtosis data → fat-tail term contributes zero), the system still produces a sized position from the remaining components.

The synthesis is intentionally conservative. When only one signal fires, the weighted output stays small. That is the methodology working as designed, not a bug.

---

### Sources and further reading

- Shannon, C. E. (1948). *A Mathematical Theory of Communication*. Bell System Technical Journal, 27(3), pp. 379–423.
- Kullback, S., and Leibler, R. A. (1951). On Information and Sufficiency. *Annals of Mathematical Statistics*, 22(1), pp. 79–86.
- Taleb, N. N. (2007). *The Black Swan: The Impact of the Highly Improbable*. Random House.
- Taleb, N. N. (2012). *Antifragile: Things That Gain from Disorder*. Random House.
- Kelly, J. L. (1956). A New Interpretation of Information Rate. *Bell System Technical Journal*, 35(4), pp. 917–926.
- Bortkiewicz, L. von (1898). *Das Gesetz der kleinen Zahlen*. B. G. Teubner.
- Thorp, E. O. (2006). The Kelly Criterion in Blackjack, Sports Betting, and the Stock Market. In *Handbook of Asset and Liability Management*.

Each issue of Outlier Weekly re-cites the relevant primary source inline so the math is verifiable end-to-end without trusting this page.

**Published issues (methodology applied):**

| Issue | Date | Market (short) | Call | Link |
|------:|------|----------------|------|------|
| 1 | 2026-05-14 | Anthropic–Pentagon by June 30 | SKIP | [Substack](https://outlierweekly.substack.com/p/issue-1-one-market-three-formulas) |
| 2 | 2026-05-26 | Iran airspace by June 30 | SKIP | [Substack](https://outlierweekly.substack.com/p/issue-2-iran-airspace-three-formulas) |

---

### Subscribe

If this is the methodology you want applied to live markets every Tuesday with the math shown and the call forward-tracked, the founders' tier is $19 / month, capped at the first 100 subscribers, locked at that price for as long as you stay subscribed.

[Subscribe to Outlier Weekly →]

*Research, not financial advice. Outlier Weekly publishes methodology applied to live markets; readers are responsible for their own due diligence, position sizing, and risk management. No fiduciary relationship is created by subscription.*

---

