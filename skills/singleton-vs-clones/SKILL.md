---
name: singleton-vs-clones
description: Choose deployment architecture when you need many instances of the same logic: full deploy per instance, EIP-1167 minimal-proxy clones, or a single singleton keyed by a params hash. Use when a factory does `new Contract(...)` in a loop or per user action.
---
# Singleton vs. Clones vs. Per-instance deploy

## The decision

You have logic that recurs, be it a market, a vault, a campaign, a stream... and you'll create many of
them. Three architectures, from most expensive to cheapest per instance:

### A. Full deploy per instance:`new Contract(...)`

Each instance is its own full bytecode at its own address.

- **Pro:** simplest mental model; per-instance immutables; blast radius is isolated (a bug in one
  instance's state doesn't touch others); "one instance = one address" is clean for integrations.
- **Con:** you pay the full contract creation cost *every time*. For a large contract created often,
  deploy gas dominates user cost.

### B. EIP-1167 minimal-proxy clones

Deploy the logic **once** as an implementation; each instance is a ~45-byte proxy that
`delegatecall`s to it.

```solidity
import {Clones} from "@openzeppelin/contracts/proxy/Clones.sol";

address public immutable implementation;

function create(bytes calldata initData) external returns (address instance) {
    instance = Clones.clone(implementation);       // ~45 bytes, cheap
    IInitializable(instance).initialize(initData); // immutables become init-set storage
}
```

- **Pro:** keeps the per-instance-address model; slashes deploy gas to near-constant.
- **Con:** `immutable` constructor args become `initialize`-set storage, so you must guard
  `initialize` against re-calling, and re-check reentrancy on the init path. Slight runtime overhead
  per call (the delegatecall hop). Use `Clones.cloneDeterministic` for CREATE2 addresses.
- **Tooling:** OpenZeppelin `Clones`, Solady `LibClone` (also supports clones-with-immutable-args,
  which restores real immutables by appending them to the proxy bytecode).

### C. Singleton keyed by a params hash

The *entire* protocol is one immutable contract; every "instance" is an entry in a mapping keyed by
the hash of its parameters. This is **[Morpho-blue](https://github.com/morpho-org/morpho-blue/blob/main/src/Morpho.sol/)**: all lending markets live in one ~600-line singleton,
and creating a market is an `SSTORE`, not a `CREATE`.

```solidity
mapping(bytes32 id => InstanceState) internal _state;   // id = keccak256(abi.encode(params))

function create(Params calldata p) external {
    bytes32 id = keccak256(abi.encode(p));
    require(!_state[id].exists, "exists");
    _state[id] = InstanceState({ ... });                // cheapest possible "deploy"
}
```

- **Pro:** cheapest creation by far; one audited surface instead of N copies; immutable core means
  no upgrade key to compromise. Extensibility is pushed *outward* into permissionless periphery.
- **Con:** a bug is protocol-wide (no per-instance isolation); per-instance customization is lost;
  the core must stay tiny and rigid to keep the shared surface auditable.

## How to choose

- Instances rarely created, or per-instance isolation is a genuine security virtue → **A** is fine.
- Same logic, created often, want to keep per-instance addresses → **B (clones)**. Usually the
  pragmatic win.
- Creation cost dominates and you can accept a rigid, protocol-wide core → **C (singleton)**.

Whichever you pick, **decide explicitly** and record why: "we deploy per instance to isolate blast
radius" is a fine decision; defaulting into full deploys because `new` was easiest is not.

## Checklist

- [ ] Architecture chosen against creation frequency and isolation needs, and documented.
- [ ] Clones: `initialize` is guarded against re-entry/re-call; no logic assumes constructor immutables.
- [ ] Singleton: core is minimal and immutable; per-instance state is fully namespaced by id.

## References

- Morpho-blue `src/Morpho.sol` (singleton)
- OpenZeppelin `contracts/proxy/Clones.sol`; Solady `src/utils/LibClone.sol`
- EIP-1167 (minimal proxy)
