---
name: invariant-testing
description: Find the bugs unit tests miss by stating what must ALWAYS be true and fuzzing random call sequences to break it. Write Foundry invariant tests with a bounded handler. Use when hardening or auditing any stateful protocol (vaults, AMMs, lending, accounting) — this is how core-invariant breaks are discovered.
---
# Invariant testing

## The principle

Most high-severity exploits are **invariant breaks**: a property the protocol assumes always holds
is violated by some sequence of calls no one wrote a unit test for. Unit tests check *known* cases;
**stateful fuzzing** throws random sequences of calls with random arguments at the system and checks
the invariant after each one, surfacing the *unknown* case. Auditing a protocol like `TSwap` is
largely the work of naming its invariants and then trying to break them.

Typical invariants:

- **Solvency:** `sum(userBalances) <= token.balanceOf(protocol)` — the protocol can always pay out.
- **AMM constant:** `x * y >= k` after every swap (fees only grow it).
- **Accounting identity:** `totalShares == sum(shares)`, `totalSupply == sum(balances)`.
- **No free money:** you can never end a sequence richer than you started without providing value.

## The rule

1. Write invariants down explicitly as `invariant_*` functions asserting the property.
2. Drive them through a **handler** that only makes *valid* state transitions (bounded inputs, real
   actors), so the fuzzer spends its runs on reachable states instead of trivial reverts.
3. Track cross-call expectations with **ghost variables** and assert them in the invariant.

## Pattern: Foundry handler-based invariant test

```solidity
// Handler: the only surface the fuzzer calls. It bounds inputs to valid ranges.
contract Handler is Test {
    Pool pool; uint256 public ghost_depositedSum;   // ghost: mirrors what we believe is true

    constructor(Pool _pool) { pool = _pool; }

    function deposit(uint256 amount) external {
        amount = bound(amount, 1, 1e24);             // keep the fuzzer in reachable states
        token.mint(address(this), amount);
        token.approve(address(pool), amount);
        pool.deposit(amount);
        ghost_depositedSum += amount;
    }

    function withdraw(uint256 shares) external {
        shares = bound(shares, 0, pool.sharesOf(address(this)));
        pool.withdraw(shares);
    }
}

contract PoolInvariants is Test {
    Pool pool; Handler handler;

    function setUp() public {
        pool = new Pool();
        handler = new Handler(pool);
        targetContract(address(handler));           // fuzz ONLY the handler, not the pool directly
    }

    // Checked after every random sequence the fuzzer generates.
    function invariant_solvent() public view {
        assertGe(token.balanceOf(address(pool)), pool.totalDeposits());
    }
}
```

Run with `forge test`; configure depth/runs via `[invariant]` in `foundry.toml`
(`runs`, `depth`, `fail_on_revert`). A failure prints the exact call sequence that broke the
invariant — your proof of concept.

## Design notes

- **Handler-based vs open testing.** Pointing the fuzzer straight at the protocol (`targetContract`
  on the pool) wastes runs on reverts and can't express multi-step setup. A handler bounds inputs and
  manages actors, so runs land in meaningful states.
- **`fail_on_revert`.** Start `true` to catch unexpected reverts, then move to `false` (with tight
  bounds) once you only want invariant violations, not expected input rejections.
- **Ghost variables** capture "what should be true so far" across calls (running sums, last actor) —
  the invariant compares real state against the ghost.
- **Alternatives.** Echidna and Medusa (property-based fuzzers) and Halmos/Kontrol (symbolic
  execution) prove or fuzz the same invariants; the discipline of *naming the invariant* is what
  matters and transfers across tools.

## Checklist

- [ ] The protocol's core invariants (solvency, accounting identity, AMM constant) are written down.
- [ ] Each invariant is an `invariant_*` assertion, not just a unit test.
- [ ] Fuzzing goes through a handler that bounds inputs to valid state.
- [ ] Ghost variables track cross-call expectations where a single-call assert can't.
- [ ] `runs`/`depth` are set high enough to explore; failing sequences are turned into PoC tests.

## References

- Foundry Book: "Invariant Testing" (`targetContract`, handlers, `[invariant]` config, `bound`)
- Echidna / Medusa (property fuzzers), Halmos / Kontrol (symbolic) as alternatives
- Cyfrin security course, §5: Invariants & core-invariant breaking (T-Swap)
