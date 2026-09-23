---
name: dos-unbounded-operations
description: Keep functions available under adversarial input — no unbounded loops over attacker-growable data, and one recipient can never block everyone. Pull over push, bound the work, isolate external-call failure. Use for distributions, batch payouts, queues, refunds, or any loop whose length a user controls.
---
# DoS via unbounded operations

## The problem

A function that is correct on small inputs can become **permanently uncallable** when an attacker
inflates its cost past the block gas limit (~30M gas), or makes one iteration revert so the whole
loop reverts. Two recurring shapes:

- **Unbounded loops over attacker-growable data.** Looping over an array of entrants/holders that
  anyone can add to. The `PuppyRaffle` refund loop and its `O(n²)` duplicate check are the classic
  examples — once the array is large enough, the function always runs out of gas.
- **Push payments to a list.** Sending funds in a loop to many recipients. If one is a contract that
  reverts, consumes all forwarded gas, or is now blacklisted by the token (e.g. USDC freeze),
  **every** payout in the batch reverts. `King of the Ether` bricked exactly this way: a contract
  king whose fallback reverts blocks the next king forever.

## The rule

1. **No unbounded loop** over data whose length an attacker can grow.
2. **Pull over push:** credit an internal balance and let each account withdraw itself, so one bad
   recipient can't block the others. (See [[escrow-accounting]] for the accounting side.)
3. **Isolate external-call failure** (per-recipient claim, `try/catch`, or a bounded gas stipend)
   and **cap batch sizes** for anything that must iterate.

## Pattern: pull payment instead of a push loop

```solidity
contract Distributor {
    error NothingToClaim();
    mapping(address => uint256) public owed;   // credited, not pushed

    // O(1) per call, and no external call in the accounting step.
    function allocate(address to, uint256 amount) internal {
        owed[to] += amount;
    }

    // Each recipient pulls their own funds; one reverting recipient affects only itself.
    function claim() external {
        uint256 amount = owed[msg.sender];
        if (amount == 0) revert NothingToClaim();
        owed[msg.sender] = 0;                                   // effects before interaction
        (bool ok, ) = msg.sender.call{value: amount}("");
        require(ok, "transfer failed");
    }
}
```

## Design notes

- **`O(n²)` scans** (e.g. "is this address already in the array?") are a DoS on their own — replace
  with a `mapping(address => bool)` membership check, turning the loop `O(1)`.
- When a loop is unavoidable, **paginate**: process a caller-supplied `[start, end)` window with a
  bounded size, so the work per transaction is capped regardless of total size.
- For push ETH sends you can't avoid, forward a **fixed gas stipend** so a griefing fallback can't
  burn the whole block — Solady's `SafeTransferLib` exposes `GAS_STIPEND_NO_GRIEF` (100000) and
  `forceSafeTransferETH`. (See [[safe-erc20-transfers]].)
- Don't gate exits on external success you don't control: a pausable/blacklisting token or a
  reverting recipient must not be able to trap other users' funds (see [[pausable-circuit-breaker]]).

## Checklist

- [ ] No loop iterates over an array an attacker can grow without bound.
- [ ] Distributions use pull (per-account `claim`) rather than a push loop.
- [ ] Membership/dedup checks use a mapping, not a nested loop.
- [ ] Any required iteration is paginated with a caller-bounded window.
- [ ] A single reverting/blacklisted/gas-guzzling recipient cannot block others.

## References

- OpenZeppelin `contracts/security/PullPayment.sol`, `utils/escrow/Escrow.sol`
- Solady `src/utils/SafeTransferLib.sol` (`GAS_STIPEND_NO_GRIEF`, `forceSafeTransferETH`)
- *Mastering Ethereum*, ch. 9: "Denial of Service (DoS)"
- Cyfrin security course, §4: DoS (motivating case: Puppy Raffle)
