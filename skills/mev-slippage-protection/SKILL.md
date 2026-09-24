---
name: mev-slippage-protection
description: Protect user trades from front-running and sandwich attacks — require a caller-supplied minimum output (or maximum input) and a real deadline; never default them to 0 / block.timestamp. Use for swaps, liquidity add/remove, mints priced off a pool, or any value-moving op exposed to the public mempool.
---
# MEV & slippage protection

## The problem

Pending transactions sit in a **public mempool** where searchers can reorder, insert, and bundle
them (MEV). A trade with no output floor is **sandwiched**: the attacker buys ahead of you (pushing
the price up), your trade executes at the inflated price, and they sell right after — the spread is
your loss. The two canonical bugs:

- **No minimum output / maximum input.** `amountOutMin = 0` means "accept any price," so the whole
  slippage the attacker manufactures is legal.
- **Fake deadline.** Setting `deadline = block.timestamp` inside the function is *no protection at
  all* — it's whatever block includes the tx, so it always passes. A validator can hold the tx for
  blocks and execute it whenever the sandwich is most profitable.

## The rule

1. Every price-taking operation accepts a **caller-supplied `minAmountOut`** (or `maxAmountIn`) and
   reverts if the realized amount is worse.
2. Accept a **real `deadline`** passed in by the caller and enforce `block.timestamp <= deadline` —
   don't synthesize it on-chain.
3. **Don't compute the slippage bound on-chain from the pool you're trading against** — that reads
   the very price being manipulated, so it protects nothing. The bound must originate off-chain.

## Pattern: floor the output, honor a real deadline

```solidity
contract Swapper {
    error Expired();
    error TooLittleReceived(uint256 got, uint256 min);

    function swap(uint256 amountIn, uint256 minAmountOut, uint256 deadline)
        external
        returns (uint256 amountOut)
    {
        if (block.timestamp > deadline) revert Expired();   // deadline comes from the caller

        amountOut = _executeSwap(amountIn);                 // touches the pool / AMM

        // Realized price must clear the caller's floor, computed off-chain from a fair quote.
        if (amountOut < minAmountOut) revert TooLittleReceived(amountOut, minAmountOut);
    }
}
```

This mirrors the Uniswap router interface (`amountOutMin`, `amountInMax`, `deadline`): the contract
guarantees "no worse than this," and the frontend derives the bound from a recent quote and a user
slippage tolerance (e.g. 0.5%).

## Design notes

- **Tolerance lives at the edge.** The contract enforces the floor; the *value* of the floor is a
  UX/off-chain decision. On-chain, only reject — never invent — the bound.
- **Deadlines bound proposer withholding.** A tight deadline limits how long a validator can sit on
  the tx waiting for a profitable sandwich.
- **Defense in depth:** private orderflow (e.g. Flashbots Protect) keeps the tx out of the public
  mempool; commit–reveal ordering hides intent until execution. Use these on top of, not instead of,
  the min-out/deadline check.
- Oracle-priced (rather than pool-priced) operations still need slippage bounds if the oracle can
  lag or be nudged — see [[oracle-safety]].

## Checklist

- [ ] Every swap / liquidity op takes `minAmountOut` (or `maxAmountIn`) and reverts on shortfall.
- [ ] `deadline` is a caller argument enforced against `block.timestamp`, not `block.timestamp` itself.
- [ ] The slippage bound is computed off-chain, not from the pool being traded against.
- [ ] Sensitive flows consider private orderflow or commit–reveal as additional cover.

## References

- Uniswap v2 `UniswapV2Router02` / v3 `SwapRouter` (`amountOutMin`, `amountInMaximum`, `deadline`)
- Flashbots Protect (private transaction relay)
- Cyfrin security course, §8: MEV & slippage protection (Vault Guardians)
