---
name: escrow-accounting
description: Custody and release funds correctly; internal ledger over balanceOf, debit-before-transfer, pull payments, and an explicit state machine with terminal states. Use when building escrow, vesting, streaming, payment splitters, or any hold-and-release vault.
---
# Escrow accounting

## The principle

An escrow's job is to hold value and release it under rules, without ever paying out more than it
holds or stranding what it owes. Four disciplines, learned from Sablier, Merit, and every audited
vault:

### 1. Internal ledger, not `balanceOf(this)`

Track each account's entitlement in a mapping. Treat the token balance as opaque.

Reading `token.balanceOf(address(this))` for accounting invites a **donation/inflation grief**:
anyone can transfer tokens directly to the contract and skew a balance-derived computation. An
internal ledger, credited only by your own `deposit` path, is immune.

```solidity
mapping(address => uint256) private _entitlement;

function deposit(uint256 amount) external {
    uint256 before = token.balanceOf(address(this));
    token.safeTransferFrom(msg.sender, address(this), amount);
    uint256 received = token.balanceOf(address(this)) - before;   // fee-on-transfer safe
    _entitlement[msg.sender] += received;                          // credit what arrived
}
```

### 2. Debit before transfer (checks-effects-interactions CEI)

Every payout debits the ledger *before* the external transfer, so a token with a transfer hook
cannot re-enter and double-spend.

```solidity
function release(address to, uint256 amount) external nonReentrant {
    uint256 bal = _entitlement[msg.sender];
    require(amount <= bal, "insufficient");
    _entitlement[msg.sender] = bal - amount;    // effect first
    token.safeTransfer(to, amount);             // interaction last
}
```

### 3. Pull over Push

Let the payee withdraw against a computed entitlement rather than pushing funds to a list of
recipients. Push loops can be griefed (one reverting recipient blocks everyone) and hand control to
each recipient mid-loop. Sablier's `withdraw` is caller-initiated against `withdrawableAmountOf`.

### 4. Explicit state machine with terminal states

Model the lifecycle as an enum and enforce legal transitions. Sablier: `PENDING → STREAMING → SETTLED / CANCELED / DEPLETED`. A canceled escrow is a one-way door; a depleted one cannot pay again.
The invariant to preserve: **`refundable + withdrawn + withdrawable == deposited`** at all times.

## Design notes

- **Fees as basis points against a named constant** (e.g. `FEE_BASIS = 10_000`); fix the fee amount
  at deposit, don't recompute it at claim where inputs may have moved.
- **Grace / cliff windows:** if claimants get a window after an end event, gate reclaim-by-owner
  until the window closes, so the owner can't sweep funds a claimant is still owed.
- **`NoDelegateCall`**  for a singleton escrow, guard against being reached via `delegatecall`
  (which would run your logic against a foreign storage layout).
- **Round in the protocol's favor:** see the `fixed-point-rounding` skill; round *down* payouts.

## Checklist

- [ ] Accounting uses an internal ledger; `balanceOf` is never the source of truth.
- [ ] Deposits credit the measured delta, not the requested amount.
- [ ] Every payout debits before transferring; guarded with `nonReentrant`.
- [ ] Payments are pull-based where feasible.
- [ ] Lifecycle is an explicit enum with enforced transitions and terminal states.
- [ ] Owner reclaim is gated behind any claimant grace window.

## References

- Sablier `lockup/src/SablierLockup.sol`, `bob/src/SablierEscrow.sol`
- Merit `src/Escrow.sol`
- Mastering Ethereum, ch. 9: "Reentrancy", "Denial of Service"
