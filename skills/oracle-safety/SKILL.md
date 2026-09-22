---
name: oracle-safety
description: Consume external data (prices, off-chain metrics, aggregates) without trusting a single manipulable reading — staleness checks, bounds, TWAP over spot, and report/use separation with a dispute window. Use whenever on-chain logic depends on a value from outside the contract.
---

# Oracle safety

## The principle

**Never trust a single spot reading you don't control.** Any value that enters from outside — a
price feed, an off-chain metric, a reported aggregate — is an attack surface. The question is not
"is this oracle honest" but "what happens when this reading is wrong, stale, or manipulated in one
block."

## Defenses, by threat

### Manipulable-in-one-block (AMM spot prices)

A spot price from a DEX pool can be moved with a flash loan inside a single transaction, read by your
contract, and moved back. Don't read spot.

- Use a **TWAP** (time-weighted average) — Uniswap v3's cumulative-tick observations average the
  price over a window, so a one-block manipulation is diluted by the window length.
- Or use a robust external feed (e.g. Chainlink) with the checks below.

### Stale data

A feed that stopped updating returns a plausible-but-old value. Always check freshness.

```solidity
(, int256 answer, , uint256 updatedAt, ) = feed.latestRoundData();
require(answer > 0, "bad price");
require(block.timestamp - updatedAt <= MAX_STALENESS, "stale price");   // heartbeat
```

### Trusted-but-unverified reporter (off-chain computation pushed on-chain)

When a value can only be computed off-chain (an aggregate, a scan of event logs, a KPI), you can't
verify it on-chain — so **separate the report from its use with a challenge period**. This is the
optimistic model:

1. A permissioned reporter *submits* a value; it is stored with a `deadline = now + disputeWindow`.
2. During the window, a governor (or anyone, in a bonded scheme) can *dispute* and void it.
3. Only after the window closes unchallenged can the value be *applied*.

```solidity
function submit(bytes32 id, uint256 value) external onlyReporter {
    reports[id] = Report(value, uint64(block.timestamp + disputeWindow), false, false);
}
function apply_(bytes32 id) external {                       // permissionless after the window
    Report storage r = reports[id];
    require(!r.disputed && !r.applied, "void/done");
    require(block.timestamp >= r.deadline, "window open");   // report/use separation
    r.applied = true;
    _use(r.value);
}
```

Bind the report id to the reporter and a per-reporter sequence so identical claims don't collide,
and enforce that the target is real (e.g. a registry membership check) before storing.

## Cross-cutting rules

- **Bound the move.** Reject a new value that jumps more than a sane delta from the last — catches
  both fat-fingers and manipulation.
- **Validate the domain.** Positive price, expected decimals, sane range.
- **Fail closed.** On a missing/garbage reading, revert or fall back to a safe default — never
  proceed on `0` as if it were a real value.
- **Front-running.** Applying a matured report is often permissionless; make sure caller *ordering*
  between two pending reports can't misattribute or double-count (monotonic cumulative counters help).

## Checklist

- [ ] No raw AMM spot price for anything valuable; TWAP or a vetted feed instead.
- [ ] Staleness/heartbeat check on every external feed read.
- [ ] Trusted off-chain values go through report → dispute window → apply, not straight to use.
- [ ] Values domain-validated (sign, decimals, range) and bounded against the previous reading.
- [ ] Fail-closed on bad/missing data; `0` never treated as valid.

## References

- Uniswap v3 `contracts/libraries/Oracle.sol` (TWAP observations)
- Aave v2 price oracle; Chainlink `latestRoundData` patterns
- Mastering Ethereum, ch. 11 "Oracles"; ch. 9 — "Price Manipulation"
