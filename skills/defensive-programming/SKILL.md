---
name: defensive-programming
description: The baseline mindset and antipattern catalogue for smart-contract security — minimalism, reuse, input validation, and the common vulnerability classes with their fixes. Use at the start of any contract work and as a pre-audit self-review.
---
# Defensive programming for smart contracts

Smart contract code is unforgiving: it runs in a public, adversarial environment, and once deployed
an exploit "unfolds in a single transaction," long before you can intervene. This skill is the
baseline mindset from *Mastering Ethereum* ch. 9, plus a checklist of the common vulnerability
classes. The other skills in this repo go deep on individual topics; start here.

## The five priorities (in order)

1. **Minimalism / simplicity.** More code means more bugs, not more value. Before writing a
   component, ask whether it's needed; after, cut edge cases and nonessential features. Simpler
   contracts are easier to reason about, test, and audit.
2. **Code reuse.** Don't reinvent the wheel: reuse battle-tested libraries (OpenZeppelin, Solady).
   Reused code beats freshly written code "no matter how confident you feel.".
3. **Code quality.** Treat it like aerospace, not a web widget. Rigorous methodology; assume no
   second chance to fix.
4. **Readability / auditability.** Public bytecode; write for the auditor and other devs. Clear names, documented
   invariants.
5. **Test coverage.** "Never assume that input is well formed or properly bounded or that it has a
   benign purpose." Test all arguments against expected ranges and formats.

## The antipattern catalogue

| Class                                | The bug                                                              | Fix (see also)                                                                                         |
| ------------------------------------ | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **Reentrancy**                 | External call re-enters before state settles                         | Checks-effects-interactions;`nonReentrant`; pull payments → `reentrancy-guards`                   |
| **DELEGATECALL**               | Executing foreign code against your storage layout                   | Don't delegatecall untrusted code;`NoDelegateCall` on singletons                                     |
| **Entropy illusion**           | On-chain "randomness" (blockhash, timestamp) is validator-influenced | Use a VRF/commit-reveal; never gamble on block values                                                  |
| **Unchecked CALL returns**     | Ignoring a`false`/no-return from a low-level call                  | Check return; use`SafeERC20` → `safe-erc20-transfers`                                             |
| **Race / front-running**       | Outcome depends on tx ordering in the mempool                        | `minAmountOut`-style bounds, commit-reveal, oracle price → `oracle-safety`                        |
| **Denial of service**          | One reverting/unbounded element blocks everyone                      | Pull over push; bound every loop; no external call in a loop that can revert                           |
| **Floating point / precision** | Truncation and wrong rounding leak value                             | Full-precision`mulDiv`, round against the extractor → `fixed-point-rounding`                      |
| **Price manipulation**         | Single spot reading moved in one block                               | TWAP / vetted feed, staleness + bounds →`oracle-safety`                                             |
| **Improper input validation**  | Trusting caller-supplied addresses/amounts/lengths                   | Validate every argument: non-zero, in-range, array-length-matched, bounded                             |
| **Signature replay**           | A signature reused across time/chains/contracts                      | EIP-712 domain + nonce + expiry + malleability check →`eip712-signature-verification`               |
| **Misconfiguration**           | Wrong owner, no emergency stop, over-broad authority                 | Two-step ownership, least privilege, pause →`two-step-access-control`, `pausable-circuit-breaker` |

## Input validation

Because "anyone can execute your contract with whatever input they want":

- Reject `address(0)` where a real address is required.
- Bound every array length that drives a loop (`require(arr.length <= MAX)`), and check parallel
  arrays are equal length.
- Enforce numeric ranges and monotonicity where the domain requires it (e.g. cumulative totals must
  not decrease).
- Prefer **custom errors** (`error TooMany(uint256 got, uint256 max);`) over string reverts: cheaper
  and machine-readable. Keep the codebase consistent; don't mix `revert("1")` string codes with
  custom errors.

## Pre-ship self-review

- [ ] Every external input validated (zero-address, ranges, array lengths).
- [ ] Every loop is bounded; no revert-prone external call inside a loop.
- [ ] CEI + guards on all value-moving functions.
- [ ] Reused audited libraries instead of hand-rolled crypto/token/math.
- [ ] Emergency stop on entry paths; ownership is two-step; authority is least-privilege.
- [ ] Invariants written down and covered by tests, including adversarial inputs.
- [ ] No on-chain randomness for anything valuable; no `tx.origin` auth.

## References

- Mastering Ethereum, ch. 9 "Smart Contract Security" (full antipattern catalogue)
- The other skills in this repository, referenced per row above
