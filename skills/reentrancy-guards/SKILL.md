---
name: reentrancy-guards
description: Prevent reentrancy on any function that makes an external call — token transfers, ETH sends, calls into unknown contracts. Use checks-effects-interactions first, a guard second.
---

# Reentrancy guards

## The problem

Any external call hands control to code you don't own. If that code calls back into your contract
before your state has settled, it observes stale state and can act on it repeatedly — the classic
drain is a `withdraw` that sends ETH *before* zeroing the balance, letting the recipient's fallback
re-enter and withdraw again. This is the vulnerability behind The DAO.

External calls that hand over control include: ETH sends (`.call`, `.transfer`), ERC-20 transfers
of tokens with transfer hooks (ERC-777, ERC-1363), ERC-721/1155 `safeTransfer` callbacks, and any
call into an address supplied by a user.

## The rule (in priority order)

1. **Checks-Effects-Interactions.** Validate inputs, then update *all* state, then make the external
   call *last*. This alone defeats most reentrancy. The guard is belt-and-braces, not the primary fix.
2. **Prefer pull over push.** Let the payee withdraw their own funds rather than pushing to them in
   the middle of another operation.
3. **Add a `nonReentrant` guard** on every function that makes an external call and mutates state.

## Pattern — CEI + guard

```solidity
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract Vault is ReentrancyGuard {
    mapping(address => uint256) private _balance;

    function withdraw(uint256 amount) external nonReentrant {
        uint256 bal = _balance[msg.sender];      // CHECK
        require(amount <= bal, "insufficient");
        _balance[msg.sender] = bal - amount;     // EFFECT (before the call)
        (bool ok, ) = msg.sender.call{value: amount}("");   // INTERACTION (last)
        require(ok, "transfer failed");
    }
}
```

The state write happens *before* the call, so a re-entrant call sees the already-debited balance
and fails the check — even without the guard.

## Gas: use transient storage where available

OpenZeppelin's default `ReentrancyGuard` is a storage-slot mutex (~2900 gas warm SSTORE per guarded
call). On chains with the EIP-1153 (Cancun) opcodes, use the transient variant — same semantics,
far cheaper, auto-clears at end of transaction:

- **`ReentrancyGuardTransient`** (OpenZeppelin) — `TSTORE`/`TLOAD` backed.
- **Solady `ReentrancyGuard`** (`src/utils/ReentrancyGuard.sol`) — also exposes a read-only variant
  (`nonReadReentrant`) for view functions that must not be entered during a callback.

Confirm your target EVM version is Cancun+ before switching.

## Cross-function and read-only reentrancy

A guard on one function does not protect a *different* unguarded function that reads the same state,
nor a `view` function an attacker calls mid-callback (read-only reentrancy — the caller reads an
inconsistent price/total). Guard every function that touches shared value state, and consider the
read-only variant for views that feed pricing.

## Checklist

- [ ] Every state mutation happens before the external call (CEI).
- [ ] `nonReentrant` on all state-mutating functions with external calls.
- [ ] Considered cross-function and read-only reentrancy, not just single-function.
- [ ] Transient-storage guard used when EVM target is Cancun+.

## References

- OpenZeppelin `contracts/utils/ReentrancyGuard.sol`, `ReentrancyGuardTransient.sol`
- Solady `src/utils/ReentrancyGuard.sol`
- Mastering Ethereum, ch. 9 — "Reentrancy"
