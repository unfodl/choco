# Progressive‑Release Asset Financing — Protocol Specification

**Status:** Draft · **Version:** 0 · **Audience:** public

This document describes, formally and implementation‑independently, a financing
primitive in which a financier co‑purchases an asset with a client, the client
economically owns the whole asset immediately, and the financed portion is held
as collateral that is *released progressively* as the client repays principal.

It is a conceptual specification. Concrete parameters (amounts, rates, terms)
are given only as **illustrative examples** and are not part of the protocol.

---

## 1. Motivation

A client wants more of an asset now than their cash allows. A financier advances
the difference. Two properties are desirable and usually in tension:

1. **The client owns the upside from day one** — all of the asset's future
   value accrues to the client, not the financier.
2. **The financier's advance is secured** — if the client stops paying, the
   financier can be made whole from collateral without recourse to the client.

Progressive‑release financing reconciles them: the asset is bought in full at
origination, the client owns all of it in economic terms, but the part
corresponding to the outstanding advance is **locked**. Each principal repayment
**unlocks** a proportional slice. A first‑loss buffer, funded by the client's own
down payment, absorbs adverse price moves so the financier is not exposed to
day‑to‑day volatility and never needs to force a sale.

> Buy more of the asset today. Repay over a fixed term. Every principal payment
> releases more of it to you.

---

## 2. Roles

| Role | Description |
|------|-------------|
| **Client** | Borrows against a cash down payment to acquire more of the asset. Owns 100% of the economic exposure from origination. |
| **Financier** | Advances the financed principal, buys the asset, custodies the locked portion, and services the loan. |
| **Price source** | Provides a single executable acquisition price at origination. Not consulted again. |

The protocol assumes omnibus custody with internal accounting. Per‑client wallets,
on‑chain settlement, and automated treasury movement are out of scope (§13).

---

## 3. The position

Every client has exactly one **position** in the asset, denominated in the
asset's smallest indivisible unit (integers only; no fractional or
floating‑point quantities anywhere).

```
owned = locked + available + reserved            (invariant — always true)
```

| Field | Meaning |
|-------|---------|
| `owned` | Total units the client economically owns. Derived; never written directly. |
| `locked` | Units securing outstanding financing. Not transferable. |
| `available` | Units the client may withdraw or transfer out. |
| `reserved` | Formerly‑available units held against an in‑flight outbound transfer. |

The invariant is checked after every state transition and re‑asserted at the
storage layer. Any violation is a fault, not a client‑visible error.

Monetary amounts are integer minor currency units (e.g. cents). Sub‑unit
quantities that arise during interest accrual are carried at higher fixed‑point
precision (§8) and only rounded when a payment is actually due or made.

---

## 4. Loan lifecycle

A loan is **two‑phase**: the terms are recorded first; nothing financial happens
until the financier funds it.

```
                 ┌─────────────────┐  reject   ┌──────────┐
   request  ───▶ │ PENDING_FUNDING │ ────────▶ │ REJECTED │
                 └─────────────────┘           └──────────┘
                         │ fund
                         ▼
                    ┌────────┐   repay in full   ┌─────────┐
                    │ ACTIVE │ ────────────────▶ │ SETTLED │
                    └────────┘                   └─────────┘
                         │ declare default
                         ▼
                   ┌────────────┐
                   │ DEFAULTED  │
                   └────────────┘
```

- **request** — records the fixed product terms for a client. Buys nothing,
  locks nothing, writes no ledger entry. Idempotent per client: at most one
  loan may be `PENDING_FUNDING` or `ACTIVE` at a time.
- **fund** — the only moment value moves at origination (§5). Financier‑initiated.
- **reject** — a pure status change on a `PENDING_FUNDING` loan.
- **repay** — client payments (§10) reduce principal and release collateral (§7)
  until the outstanding principal reaches zero, at which point the loan is
  `SETTLED` and all remaining locked units are released.
- **declare default** — a financier‑initiated status change on an `ACTIVE` loan
  (§11). It moves no value.

---

## 5. Origination (funding)

At funding the financier:

1. Obtains **one** executable acquisition price and buys the asset for the total
   consideration `C = D + P`, where `D` is the client down payment and `P` the
   financed principal.
2. Records the exact quantity acquired, the exact price, an execution
   reference, and the funding timestamp `t₀`. **Ownership is never re‑marked
   from any later price.**
3. Partitions the acquired quantity `Q` into three tranches (§6).
4. Generates the repayment schedule (§9) with due dates measured from `t₀`.
5. Posts the origination entries to the ledger (§12) and activates the loan.

Interest accrues from `t₀` — the moment the asset is bought and principal is
outstanding — not from the earlier request or approval time.

---

## 6. Tranching

Let `Q` be the total quantity acquired for consideration `C = D + P`, split by
value into three notional sub‑amounts that sum to `C`:

| Tranche | Value | Initial state | Released… |
|---------|-------|---------------|-----------|
| **Immediate** `R₀` | a small fraction of `C` | `available` | at funding |
| **Equity** `E` | ≈ the down payment `D`, less `R₀` | `locked` | in full, only when the loan is `SETTLED` |
| **Financed** `F` | ≈ the financed principal `P` | `locked` | progressively, with principal repayment (§7) |

Quantities are derived by proportional split of `Q`:

```
q(R₀) = floor( Q · R₀ / C )
q(E)  = floor( Q · E  / C )
q(F)  = Q − q(R₀) − q(E)          // financed tranche absorbs the rounding remainder
```

so `q(R₀) + q(E) + q(F) = Q` exactly. The **equity** tranche is the client's
first‑loss buffer: it is the client's own money, it stays locked for the life of
the loan, and it is what a non‑recourse financier looks to before any shortfall
could reach them.

---

## 7. Progressive release

Only **principal** repayment releases collateral. Interest never does.

Let `q(F)` be the financed‑tranche quantity fixed at funding, `P` the original
financed principal, and `ρ` the cumulative principal repaid so far. The
cumulative financed quantity that has been released is:

```
released(ρ) = floor( q(F) · ρ / P )          bounded above by q(F)
```

A repayment that advances `ρ` moves `released(ρ_after) − released(ρ_before)`
units from `locked` to `available`, clamped so the loan never releases more than
it currently has locked.

The payment that drives outstanding principal to **zero** releases **every**
remaining locked unit — the financed remainder *and the entire equity tranche* —
so integer‑rounding dust and the first‑loss buffer are both returned on payoff.

No asset price is consulted at any point in this calculation.

*Illustrative:* with `q(F) = 200,000` units and `P = 20,000` minor units, repaying
`1,891` minor units of principal releases `floor(200,000 · 1,891 / 20,000) =
18,910` units.

---

## 8. Interest

- **Basis:** simple interest on the *outstanding principal*, actual days over a
  365‑day year, whole days only.
- **Accrual start:** the funding timestamp `t₀`.
- **Precision:** carried in a fixed‑point unit finer than the minor currency
  unit (e.g. millionths of a major unit), so daily accrual on small balances
  does not round to zero. Converted to minor units only when a payment is due
  or made.
- **Rate:** a fixed annual rate for the life of the loan.

```
interest_over(d days) = outstanding_principal · APR · d / 365
```

**Reads never mutate.** A query that reports accrued interest computes a
*preview* up to the current time and persists nothing. Accrual is written to
storage only when a repayment is applied; at that point the "accrued‑through"
marker advances by whole days and any partial day is carried forward.

There is no penalty interest and no compounding in this specification.

---

## 9. Repayment schedule

At funding, `n` equal‑interval due dates are generated from `t₀`. The scheduled
installment is the level payment that amortizes `P` over `n` periods at the
periodic rate `i = APR / periods_per_year`:

```
M = P · i / ( 1 − (1 + i)^(−n) )
```

rounded to the minor unit, with the final installment absorbing residual
rounding so the installments sum to principal plus total scheduled interest. The
schedule is what the client sees and what delinquency is measured against;
interest actually charged is always the day‑count accrual on the real
outstanding balance at payment time (§8).

*Illustrative:* `P = 20,000` minor units, `APR = 24%`, `n = 12` monthly periods
⇒ `M ≈ 1,891` minor units.

---

## 10. Payment application

A payment carries a single amount. The servicer allocates it:

1. **to accrued interest** first, up to the amount owed;
2. **to principal** with the remainder, up to the outstanding principal.

Rules:

- **No overpayment.** Any amount exceeding `outstanding_principal + accrued_interest`
  is rejected; the client is told the exact payoff figure.
- **At most one installment per payment.** Any amount above the current scheduled
  installment is *excess principal*: it reduces the balance and releases
  collateral immediately, but it does **not** pre‑pay a future installment. The
  next due date does not move — extra payment shortens the term.
- **Prepayment is free.** Paying the full outstanding principal plus interest
  accrued to that instant settles the loan with no unearned/future interest and
  no penalty, and releases all collateral (§7).
- **Idempotency.** A payment carries a client‑supplied key; a repeat of the same
  key is a no‑op that returns the already‑applied result.

---

## 11. Delinquency and default

**Delinquency is computed, never stored.** Let `days_late` be the number of days
between today and the earliest unsatisfied scheduled due date:

| `days_late` | State |
|-------------|-------|
| ≤ 0 | `CURRENT` |
| 1 … `g` | `PAST_DUE` |
| > `g` | `DELINQUENT` |

where `g` is a fixed grace period. `PAST_DUE` and `DELINQUENT` are derived on
every read from the schedule.

**Default is declared, never inferred.** `DEFAULTED` is a stored status set only
by an explicit financier action. It is a status change only:

- locked units stay locked; no further release occurs;
- already‑available units remain the client's;
- **no automatic sale, no liquidation, no margin call.**

There are **no asset‑price triggers anywhere** in this specification. Volatility
is absorbed by the equity tranche (§6), not managed by forced action. Recovery
of a defaulted position is an out‑of‑band, deliberate process outside this spec.

---

## 12. Ledger model

State is an **append‑only, double‑entry ledger**. A materialized position (§3)
is a cache derived from it, with the invariant re‑checked at the storage layer.

- **Transactions** group **entries**; each transaction is *balanced*: for every
  unit (asset units, minor currency units) the signed entry deltas sum to zero.
  Value is only ever moved between accounts, never created or destroyed inside
  the system's books.
- Posted transactions are **immutable**. Corrections are new reversing
  transactions.
- Accounts include the three materialized position sub‑balances (`locked`,
  `available`, `reserved`), a counter‑account for units entering or leaving
  custody, and non‑materialized audit accounts for the cash legs of origination
  and repayment.
- Event kinds cover at least: asset purchase, loan origination, principal
  repayment, interest payment, collateral release, and outbound‑transfer
  reserve / settle / release.
- Every mutating operation runs in a single serialized unit of work with the
  client's position row locked for update, so concurrent operations on one
  position cannot double‑spend. Idempotency keys on transactions and a
  processed‑event table make external callbacks replay‑safe.

---

## 13. Custody, price, and reserves

Three concerns are kept strictly separate:

1. **Client ownership ledger** — who owns what (§12).
2. **Operational liquidity** — the working balance the financier uses to service
   outbound client transfers; its own append‑only ledger; may be toggled off.
3. **The financier's actual asset reserves** — backing all client claims.

A single **execution price at origination** is the only price the protocol
needs. After funding, the unit quantities are fixed forever; no ongoing price
feed participates in release, interest, delinquency, or settlement. A
production deployment obtains that price from a real execution venue; a
development deployment may use a fixed configured price — a fixed price is a
hard blocker before real client funds are involved.

A deterministic, canonically‑encoded snapshot of `(available, locked, reserved)`
per client SHOULD be producible so that a solvency proof
`Σ client claims ≤ financier reserves` can be verified without revealing
individual balances.

---

## 14. Illustrative instantiation

Purely as an example of how the symbolic parameters bind. **Not part of the
protocol.**

| Parameter | Symbol | Example value |
|-----------|--------|---------------|
| Down payment | `D` | 50 major units |
| Financed principal | `P` | 200 major units |
| Total consideration | `C = D + P` | 250 major units |
| Immediate release | `R₀` | 10 major units |
| Equity (first‑loss) | `E` | 40 major units |
| Financed tranche | `F` | 200 major units |
| Term | `n` | 12 monthly periods |
| Annual rate | `APR` | 24% |
| Grace period | `g` | 3 days |
| Level installment | `M` | ≈ 18.91 major units |

At an example acquisition price where `C` buys `250,000` units: immediate
`10,000` available, equity `40,000` locked to payoff, financed `200,000` locked
and releasing with principal. Repaying the `≈ 18.91` installment on day 0
(no interest yet) releases `18,910` units; final payoff sweeps all remaining
`locked` units including the full `40,000` equity tranche.

---

## 15. Non‑goals

Out of scope for this specification: automated on‑chain settlement, per‑client
wallets or nodes, margin calls, liquidation, dynamic loan‑to‑value, price
oracles after origination, credit scoring, client‑to‑client transfers, a
separate cash wallet, an order book, automated treasury management, secondary
markets, and multi‑product loan books. One live loan per client. A single
financier‑operator role. Optimized for correctness, auditability, and
simplicity over feature breadth.

---

## 16. Glossary

| Term | Definition |
|------|------------|
| **Position** | A client's holding of the asset, split into `owned / locked / available / reserved` with `owned = locked + available + reserved`. |
| **Tranche** | One of the three notional partitions of the asset bought at funding: immediate, equity, financed. |
| **Equity tranche** | The client‑funded first‑loss buffer; locked for the life of the loan, released only at payoff. |
| **Progressive release** | Movement of units from `locked` to `available` in proportion to cumulative principal repaid. |
| **Accrued‑through marker** | The timestamp up to which interest has been *persisted*; advanced only by an applied repayment. |
| **Delinquency** | A computed lateness state (`CURRENT` / `PAST_DUE` / `DELINQUENT`) derived from the schedule. |
| **Default** | A stored status set only by explicit financier action; moves no value. |
| **Execution price** | The single acquisition price captured at origination; never re‑queried. |
