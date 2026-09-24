---
name: proxy-upgrade-safety
description: Ship upgradeable contracts without bricking them; disable initializers on the implementation, guard the upgrade authorization, and never break storage layout between versions. Use whenever a contract sits behind a proxy (Transparent, UUPS, or Beacon) or uses delegatecall to shared logic.
---
# Proxy & upgrade safety

## The problem

A proxy holds the storage and `delegatecall`s into an implementation, so the implementation's code
runs against the *proxy's* storage. Three failure classes dominate proxy audits (all three appear in
the `ThunderLoan` audit):

- **Uninitialized implementation.** Constructors run in the *implementation's* own context, not the
  proxy's, so upgradeable contracts move setup into an `initialize()` function. If the implementation
  contract itself is left uninitialized, an attacker can call `initialize()` on it directly and for
  **UUPS**, become its owner and `upgradeToAndCall` into a `selfdestruct`, bricking every proxy that
  points at it. This is the OpenZeppelin UUPS uninitialized-implementation issue.
- **Storage collision.** Reordering, inserting, changing the type of, or removing a state variable
  between versions makes the new code read the old slot's bytes as something else — silent
  corruption. The proxy's own admin/implementation pointers avoid colliding with logic by living at
  fixed pseudo-random **ERC-1967** slots.
- **Bad / unauthorized upgrade.** A UUPS `_authorizeUpgrade` with no access control lets *anyone*
  upgrade the logic — full compromise. Missing or wrong authorization is a critical every time.

## The rule

1. **Disable initializers on the implementation** (`_disableInitializers()` in its constructor) and
   use `initializer` / `reinitializer` guards.
2. **Access-control the upgrade path** (`_authorizeUpgrade` for UUPS; a trusted admin for
   Transparent).
3. **Never break storage layout:** append new variables only, preserve order and types, and reserve
   a **storage gap** (or use ERC-7201 namespaced storage) so a parent can grow without shifting a
   child's slots.

## Pattern: UUPS done safely

```solidity
import {Initializable} from "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import {OwnableUpgradeable} from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract VaultV1 is Initializable, OwnableUpgradeable, UUPSUpgradeable {
    uint256 public totalAssets;          // slot order is a permanent contract of every future version

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() { _disableInitializers(); }   // implementation can never be initialized directly

    function initialize(address owner_) external initializer {   // runs once, in the PROXY's storage
        __Ownable_init(owner_);
        __UUPSUpgradeable_init();
    }

    // Only the owner can push a new implementation. Missing this = anyone upgrades.
    function _authorizeUpgrade(address newImpl) internal override onlyOwner {}

    uint256[49] private __gap;           // reserve slots so subclasses/new vars don't collide
}
```

To add state in `VaultV2`, **append** after `totalAssets` (consuming from the gap) — never insert
above it.

## Design notes

- **Transparent vs UUPS vs Beacon.** Transparent puts upgrade logic in the proxy (admin can't call
  through it); UUPS puts it in the implementation (cheaper, but you must include `_authorizeUpgrade`
  or the contract is not upgradeable — and must not remove it); Beacon upgrades many proxies at once.
- **Validate layout automatically.** The OpenZeppelin Upgrades plugin (Hardhat/Foundry) diffs
  storage layout and blocks unsafe upgrades in CI use it rather than eyeballing slots.
- **Namespaced storage (ERC-7201)** places a contract's state at a hashed base slot, eliminating
  collision between modules and making gaps unnecessary; prefer it in new upgradeable code.
- `immutable`/`constant` values live in bytecode, not storage, so they're safe across upgrades — but
  each new implementation must set them, and they can't be initialized per-proxy.

## Checklist

- [ ] Implementation constructor calls `_disableInitializers()`.
- [ ] `initialize()` is guarded by `initializer` and can't be re-run (`reinitializer(n)` for later versions).
- [ ] UUPS `_authorizeUpgrade` exists and is access-controlled.
- [ ] Storage variables are append-only across versions; no reorder, retype, or removal.
- [ ] A storage gap or ERC-7201 namespaced storage protects layout.
- [ ] Upgrade layout is verified with the OZ Upgrades plugin in CI.

## References

- OpenZeppelin `contracts-upgradeable`: `Initializable`, `UUPSUpgradeable`, `proxy/ERC1967/`
- ERC-1967 (proxy storage slots), ERC-7201 (namespaced storage layout)
- OpenZeppelin Upgrades plugin (storage-layout validation)
- Cyfrin security course, no.6: Centralization, Proxies & Oracles (motivating case: Thunder Loan)
