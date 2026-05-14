---
title: Forward-Track — Outlier Weekly
description: Public forward-track of every Outlier Weekly call. Every issue's call is logged here on publish day with entry price, sized fraction, and resolution oracle. No retroactive edits.
permalink: /forward-track/
---

# Outlier Weekly — Forward-Track

Every call made in Outlier Weekly is logged on this page the day it ships, with the entry price, the sized fraction, the resolution date, and the eventual outcome. Track-record discipline begins with Issue 1.

**Rules of this sheet:**

- No retroactive edits. If a call resolves wrong, it stays on the sheet.
- SKIPs are first-class entries. The methodology refuses to trade when market and model converge inside the noise floor, and those refusals are logged the same as positions taken.
- Each row links back to the issue that produced the call, so any reader can audit the math from inputs to weighted output.
- Resolution rows are appended as outcomes settle; the post-mortem appears in whichever issue ships within 72 hours of resolution.

---

## Forward-track ledger

| Issue | Ship date | Market | Outcome window | Direction | Entry (YES¢) | Sized fraction | Resolution date | Status | Resolution | Issue link |
|------:|-----------|--------|----------------|-----------|--------------|----------------|-----------------|--------|------------|------------|
| 1 | 2026-05-14 | Will Anthropic make a deal with the Pentagon by [date]? | June 30, 2026 | **SKIP** | 32¢ (draft) → 27¢ (publish-day lock) | 0% of bankroll | 2026-06-30 | OPEN | pending | [Issue 1](https://outlierweekly.substack.com/p/issue-1-one-market-three-formulas) |

---

## Issue 1 — call detail

**Market:** Will Anthropic make a deal with the Pentagon by June 30, 2026?
**Direction:** SKIP — model–market convergence inside the noise floor.

**Model output (draft, p_market = 32¢):**

- Poisson edge: −4.2¢ → BUY_NO at the candidate level
- Shannon / KL divergence: 0.006 bits → SKIP (below the 0.01-bit floor)
- Taleb tail-bucket multiplier: 1.5x
- Weighted synthesis: 0.81% NO

**Model output (publish-day, p_market = 27¢):**

- Poisson edge: +0.8¢ → near-zero edge
- Shannon / KL divergence: <0.01 bits → SKIP
- Taleb tail-bucket multiplier: 1.5x
- Weighted synthesis: ≈ 0% (effective SKIP)

**Why the change between draft and publish:** the market moved 5pp between the analytical work and the publish-day lock, into the immediate neighborhood of the Poisson fair value. The directional signal that read BUY_NO at 32¢ flattens to a near-zero edge at 27¢. Under the synthesis's skip-discipline rules, this collapses the weighted output to an effective non-position. The methodology refuses the trade when market and model converge inside the noise floor — Issue 1's forward-track row is the first published instance of that refusal.

**Resolution oracle:** Polymarket UMA / public announcement of qualifying agreement, evaluated against the market's rules text as of 2026-05-14.

**Re-watch trigger:** If the June-30 price moves outside a 22–32¢ band before resolution, the synthesis is re-run and any resulting call is logged in the next issue.

---

## Ledger schema (for forward-readers)

Future entries will follow this column ordering. Definitions:

- **Issue** — newsletter issue number (1, 2, 3, …)
- **Ship date** — date the issue published (ISO YYYY-MM-DD)
- **Market** — short title of the underlying Polymarket / Kalshi market
- **Outcome window** — the specific outcome / resolution date addressed by the call
- **Direction** — BUY_YES, BUY_NO, or SKIP. SKIPs include the reason (e.g., "noise-floor convergence", "Shannon-collapse without news", "liquidity-failure")
- **Entry (YES¢)** — price at the moment of publication; for SKIPs, both draft and publish-day prices are shown
- **Sized fraction** — quarter-Kelly position size expressed as % of bankroll. SKIPs are 0%
- **Resolution date** — date the market is scheduled to resolve
- **Status** — OPEN until resolved; then WON, LOST, or SKIP-CORRECT / SKIP-WRONG
- **Resolution** — pending while OPEN; final outcome once resolved
- **Issue link** — link to the issue that produced the call

When SKIPs resolve, they are evaluated as either SKIP-CORRECT (the position the methodology would have taken would have lost, so the refusal was right) or SKIP-WRONG (the position would have won, so the refusal cost the portfolio). Track-record discipline is measured against both kinds of calls.

---

## Related

- [Methodology — three formulas, one weighting](/outlier-weekly-methodology/)
- [Subscribe to Outlier Weekly](https://outlierweekly.substack.com/)
- [Issue 1 — One Market, Three Formulas](https://outlierweekly.substack.com/p/issue-1-one-market-three-formulas)
