---
name: pausable-circuit-breaker
description: Add an emergency stop that halts entry paths (deposits, new orders) while never trapping user exits (withdrawals, claims). Use when a protocol moves money and you need a response to a discovered bug.
---
# Pausable / circuit breaker

## The principle

A protocol that moves money should be able to **stop the bleeding** the moment a bug is found
without waiting for an upgrade or a redeploy. The exploit unfolds in seconds; the pause is the only
lever fast enough.

The critical asymmetry: **pause the entry, never the exit.**

- Pausing *entry* (deposit, mint, new order, accepting a report) stops new value or new bad state
  from flowing in.
- Pausing *exit* (withdraw, claim, redeem) traps users' own funds, turning your safety mechanism
  into the attack. A pause must never be able to hold funds hostage.

## Pattern

```solidity
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";

contract Market is Pausable, Ownable2Step {
    function deposit(uint256 amount) external whenNotPaused { /* entry: pausable */ }

    function withdraw(uint256 amount) external { /* exit: NEVER gated by whenNotPaused */ }

    function pause()   external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }
}
```

## Design notes

- **Scope it.** A per-item pause (one market, one campaign) limits blast radius but can't stop a bug
  in a *shared* component. A protocol-level pause on the shared contract (an oracle, a router, a
  verifier) can. Often you want both.
- **Who holds the key.** Pausing is a fast, low-trust action; a guardian/multisig that can *pause*
  but not *unpause* (unpause reserved to governance) is a common split, pausing is reversible-safe,
  unpausing re-enables value flow and deserves more scrutiny.
- **Emit events** on pause/unpause so monitoring and users can react.
- **Don't overuse it.** A pause is centralization. Document who can pause, under what policy, and
  favor designs that need it rarely.

## Checklist

- [ ] Entry paths (`deposit`/`mint`/`submit`) carry `whenNotPaused`.
- [ ] Exit paths (`withdraw`/`claim`/`redeem`) do **not:** users can always retrieve their funds.
- [ ] Pause authority documented; consider guardian-pause / governance-unpause split.
- [ ] Shared components have a protocol-level pause, not only per-item.

## References

- OpenZeppelin `contracts/utils/Pausable.sol`
- Mastering Ethereum, ch. 9: best practices (emergency stop under misconfiguration)
