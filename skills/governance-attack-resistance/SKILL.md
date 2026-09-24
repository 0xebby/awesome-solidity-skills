---
name: governance-attack-resistance
description: Stop flash-loan and instant-execution attacks on token governance; count votes from a past snapshot, not current balance, and route execution through a timelock. Use when building on-chain voting, DAO proposals, or any parameter change gated by token-weighted votes.
---
# Governance attack resistance

## The problem

Token-weighted governance is only as safe as *when* and *how* it counts votes.

- **Flash-loan / borrowed voting power.** If voting power is read from the **current** token balance,
  an attacker borrows a huge amount in one transaction, creates or passes a proposal, and returns the
  tokens all atomically, at near-zero cost. The Beanstalk hack drained ~$182M this way: a flash-loan
  supermajority passed a malicious proposal that transferred the treasury.
- **Instant execution.** A proposal that executes the moment it passes gives honest users no time to
  react or exit before a malicious change takes effect.
- **No proposal threshold / quorum.** Lets a tiny or empty vote move the protocol.

## The rule

1. **Snapshot voting power at a past block.** Use checkpointed balances (`ERC20Votes` /
   `getPastVotes`) measured at the proposal's snapshot block a balance flash-loaned in the current
   block simply doesn't count.
2. **Voting delay + voting period:** delay between proposal creation and voting start (so the
   snapshot precedes any reaction), then a fixed voting window.
3. **Timelock on execution:** a queued delay between a passed vote and its execution, so users can
   review and exit.
4. **Proposal threshold + quorum** so trivial stakes can't propose or pass.

## Pattern: OpenZeppelin Governor + votes snapshot + timelock

```solidity
import {Governor} from "@openzeppelin/contracts/governance/Governor.sol";
import {GovernorVotes} from "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";
import {GovernorTimelockControl}
    from "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";
import {IVotes} from "@openzeppelin/contracts/governance/utils/IVotes.sol";

// The token must be checkpointed (ERC20Votes): getPastVotes(account, pastBlock) is the vote weight.
contract MyGovernor is Governor, GovernorVotes, GovernorTimelockControl {
    function votingDelay()  public pure override returns (uint256) { return 1 days; }   // snapshot then vote
    function votingPeriod() public pure override returns (uint256) { return 1 weeks; }
    function proposalThreshold() public pure override returns (uint256) { return 100_000e18; }

    // Weight is read at the proposal's snapshot block — flash-loaned balance in the current block is ignored.
    // Passed proposals are queued in the TimelockController and only executable after its minDelay.
}
```

The timelock (an OZ `TimelockController`) is set as the executor and typically owns the treasury, so
even a passed proposal can't move funds until the delay elapses.

## Design notes

- **Delegation & checkpoints.** `ERC20Votes` records balance checkpoints and requires holders to
  `delegate` (even to themselves) to activate voting power; `getPastVotes` reads a specific past
  block. Reading `balanceOf` directly is the bug.
- **Snapshot must precede the borrow window.** A non-zero `votingDelay` ensures the snapshot block is
  in the past relative to when anyone could act on the proposal.
- **Timelock delay is a tradeoff:** long enough for users to exit and for defenders to respond, short
  enough to stay governable. Emergency changes should still go through it (or a separately-scoped
  guardian): see [[two-step-access-control]] and [[pausable-circuit-breaker]].

## Checklist

- [ ] Vote weight comes from `getPastVotes` at a snapshot block, never current `balanceOf`.
- [ ] Governance token is checkpointed (`ERC20Votes` / `Votes`) and holders delegate.
- [ ] Non-zero `votingDelay` places the snapshot before any reaction window.
- [ ] Execution is routed through a `TimelockController` with a meaningful delay.
- [ ] Proposal threshold and quorum are set so trivial stakes can't propose/pass.

## References

- OpenZeppelin `contracts/governance/` (`Governor`, `GovernorVotes`, `GovernorTimelockControl`, `TimelockController`)
- OpenZeppelin `contracts/token/ERC20/extensions/ERC20Votes.sol`, `governance/utils/Votes.sol`
- Beanstalk flash-loan governance exploit (2022) as the motivating real-world case
- Cyfrin security course, no.8: Governance attack (Vault Guardians)
