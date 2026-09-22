---
name: fixed-point-rounding
description: Do fixed-point arithmetic without overflow or precision loss, and always round in the protocol's favor. Use when computing shares, proportional payouts, interest, prices, or any a*b/c where inputs are large or user-controlled.
---

# Fixed-point math & rounding

## Two failures to avoid

1. **Overflow on the intermediate product.** `a * b / c` computes `a * b` first. Even in Solidity
   0.8+ (which reverts on overflow rather than wrapping), a legitimate large `a * b` reverts before
   the divide brings it back into range — a denial of service on valid inputs.
2. **Rounding that leaks value.** Integer division truncates. If you round the *wrong* way, dust
   accumulates against the protocol and, over many operations, drains the pool or lets the last
   claimant find nothing left.

## Rule 1 — full-precision mulDiv

Use a `mulDiv` that carries the product at 512-bit intermediate precision, so `a * b` never
overflows before the division.

```solidity
import {FixedPointMathLib} from "solady/utils/FixedPointMathLib.sol";

uint256 shares = FixedPointMathLib.fullMulDiv(assets, totalShares, totalAssets); // a*b/c, no overflow
uint256 up     = FixedPointMathLib.fullMulDivUp(assets, totalShares, totalAssets); // rounds up
```

(OpenZeppelin's `Math.mulDiv` provides the same full-precision guarantee.)

## Rule 2 — round in the protocol's favor

The invariant: **the protocol must never round in a way that lets a user extract more than they put
in.** Concretely:

- **Minting shares for a deposit → round DOWN.** The user gets no more shares than earned.
- **Burning shares for a withdrawal → round DOWN** the assets returned. The user withdraws no more
  than backed.
- **Computing what a user must PAY (repay, mint cost) → round UP.** They pay at least what's owed.
- **Computing what a user RECEIVES → round DOWN.** They receive at most what's earned.

Mnemonic: *round against the party pulling value out of the system.* This is why vaults keep a `mulDivUp`
alongside `mulDiv` and choose per call site. A single flipped rounding direction is a real,
audited-for bug class (the ERC-4626 "inflation attack" is rounding + `balanceOf`-accounting combined).

## Rule 3 — order operations to preserve precision

Multiply before you divide (`a * b / c`, not `(a / c) * b`) so you don't truncate an intermediate to
zero. With a full-precision `mulDiv` this is automatic; if you must hand-roll, keep the divide last.

## Notes

- **Decimals.** Normalize tokens to a common scale before comparing/summing; an 6-decimal USDC amount
  and an 18-decimal DAI amount are not directly addable.
- **No floating point.** The EVM has none. All "decimals" are fixed-point integers; document the
  scale (e.g. WAD = 1e18) and apply it consistently.
- **Zero denominators.** Guard `totalAssets == 0` / `totalSupply == 0` bootstrap cases explicitly.

## Checklist

- [ ] All `a*b/c` go through a full-precision `mulDiv` (Solady/OZ), not raw `*` then `/`.
- [ ] Every rounding site chooses direction against the value-extracting party; up-variants used where owed.
- [ ] Multiply-before-divide; divide is the last operation.
- [ ] Token amounts normalized to a documented fixed-point scale before arithmetic.
- [ ] Zero-supply / zero-asset bootstrap paths handled.

## References

- Solady `src/utils/FixedPointMathLib.sol`
- OpenZeppelin `contracts/utils/math/Math.sol` (`mulDiv`)
- Solmate `src/utils/FixedPointMathLib.sol`
- Mastering Ethereum, ch. 9 — "Floating Point and Precision"
