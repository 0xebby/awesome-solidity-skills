---
name: safe-erc20-transfers
description: Move ERC-20 tokens safely across non-standard implementations (no-return, fee-on-transfer, approve-race). Use whenever a contract calls transfer, transferFrom, or approve on a token it does not control.
---

# Safe ERC-20 transfers

## The problem

The ERC-20 standard is under-specified in ways that break naive code:

- **No return value.** USDT and others return nothing from `transfer`/`approve`. A call to a
  `bool`-returning interface either reverts on decode or, worse, mis-reads stack garbage as `true`.
- **Returns `false` instead of reverting.** Some tokens signal failure with a `false` return that a
  naked `token.transfer(...)` silently ignores — the tokens never move, your accounting says they did.
- **Fee-on-transfer / deflationary.** The recipient receives *less* than `amount`. If you credit
  `amount`, your internal ledger drifts above the real balance and the last withdrawers are stranded.
- **Approve race (`approve` front-running).** Some tokens require the allowance be set to zero before
  a new non-zero value, or reject a change from non-zero to non-zero.

## The rule

1. Never call `transfer`/`transferFrom`/`approve` directly on an untrusted token. Use a safe wrapper.
2. When the exact received amount matters, **measure the balance delta** — do not trust `amount`.
3. Use `forceApprove` (or zero-then-set) for allowances.

## Pattern — OpenZeppelin SafeERC20 (conservative default)

```solidity
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

using SafeERC20 for IERC20;

// Credit what was ACTUALLY received (fee-on-transfer safe).
function deposit(IERC20 token, uint256 amount) external {
    uint256 before = token.balanceOf(address(this));
    token.safeTransferFrom(msg.sender, address(this), amount);
    uint256 received = token.balanceOf(address(this)) - before;
    require(received != 0, "nothing received");
    _ledger[msg.sender] += received;          // never += amount
}

// Approve-race safe.
token.forceApprove(spender, newAllowance);
```

`SafeERC20` treats "no return data" as success and "returned false" as failure, closing the
no-return and silent-false gaps.

## When to reach for the gas-optimized variant

- **Solady `SafeTransferLib`** (`src/utils/SafeTransferLib.sol`) — same guarantees in hand-written
  assembly, much cheaper. Also ships `forceSafeTransferETH` and a `GAS_STIPEND_NO_GRIEF` (100000)
  for DoS-resistant ETH sends, plus Permit2 helpers.
- **Solmate `SafeTransferLib`** — the ancestor. Caveat: it does **not** verify the token address has
  code, so a "transfer" to an empty address *succeeds*. Prefer OZ or Solady unless you add that check.

## Checklist

- [ ] No direct `.transfer` / `.transferFrom` / `.approve` on tokens you don't control.
- [ ] Deposit paths measure `balanceOf` delta, not the passed `amount`.
- [ ] Allowance changes go through `forceApprove` or zero-then-set.
- [ ] Rebasing tokens are documented as unsupported unless explicitly handled.

## References

- OpenZeppelin `contracts/token/ERC20/utils/SafeERC20.sol`
- Solady `src/utils/SafeTransferLib.sol`
- Solmate `src/utils/SafeTransferLib.sol`
- Mastering Ethereum, ch. 9 — "Unchecked CALL Return Values"
