# choco

<p align="center">
  <img src="assets/choco-logo.png" alt="Choco logo" width="160">
</p>

Choco is a high-level specification for a Bitcoin financing platform.

The platform lets a client acquire more Bitcoin than their cash down payment
would otherwise allow. Choco funds the remaining principal, the client receives
economic ownership of the full position at funding, and part of the Bitcoin is
held as collateral until repayment milestones are reached.

> Buy more Bitcoin today. Repay over time. Unlock more of your Bitcoin as you
> pay.

## Read The Spec

**[SPEC.md](SPEC.md)** describes the platform model:

- fixed starter financing products;
- admin-controlled approval and funding;
- client positions split into `available`, `locked`, and `reserved` Bitcoin;
- progressive unlocks tied to principal repayment;
- early payoff with full unlock;
- authorized agent repayment recording;
- Lightning or Spark-style withdrawal rails;
- operational wallet controls;
- reserve visibility and liabilities snapshots.

The spec is intentionally high-level. It explains what the platform allows and
what must remain true, without binding the product to a specific codebase,
payment provider, custodian, or deployment architecture.

## Status

Draft, version 0. This repository is the public platform specification.
