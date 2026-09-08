# choco

A financing primitive: a **financier co‑purchases an asset with a client**, the
client economically owns all of it from day one, and the financed portion is held
as collateral that is **released progressively** as the client repays principal.

A first‑loss buffer — funded by the client's own down payment — absorbs adverse
price moves, so the financier is not exposed to day‑to‑day volatility and never
has to force a sale. There are no price oracles after origination, no margin
calls, and no automatic liquidation.

> Buy more of the asset today. Repay over a fixed term. Every principal payment
> releases more of it to you.

## Read the spec

**[SPEC.md](SPEC.md)** — the formal, implementation‑independent specification:

- the position model and its invariant (`owned = locked + available + reserved`)
- the two‑phase loan lifecycle (request → fund → active → settled; reject; default)
- tranching at origination (immediate / equity / financed)
- progressive collateral release, proportional to principal repaid
- interest (simple, ACT/365, from funding; reads never mutate state)
- level‑payment schedule and interest‑first payment application
- delinquency (computed) vs. default (declared)
- the append‑only, balanced double‑entry ledger model

Concrete numbers in the spec are **illustrative only** and are not part of the
protocol.

## Status

Draft, version 0. This repository is the public specification only.
