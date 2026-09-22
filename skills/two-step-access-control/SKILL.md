---
name: two-step-access-control
description: Choose and wire the weakest sufficient access-control primitive: Ownable, Ownable2Step, or roles: and never leave a funds-affecting owner on single-step transfer. Use when adding any privileged function.
---
# Two-step Access Control

## The principle

Pick the **weakest tool that suffices**, and make ownership transfer **unbrickable** wherever the
owner can affect funds. More authority than the job needs is more to lose when a key is compromised.

## The three tiers

### 1. `Ownable`: one owner, one privilege level

Fine for a contract whose owner only tweaks non-critical config. The danger: `transferOwnership` is
**single-step**. Send it to a typo'd or uncontrolled address and the contract is governance-bricked
forever.

### 2. `Ownable2Step:` the default for anything holding value

Transfer is *proposed* by the current owner, then must be *accepted* by the new owner. A wrong
address simply never accepts, and nothing changes.

```solidity
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";

contract Treasury is Ownable2Step {
    constructor(address owner_) Ownable(owner_) {}

    function setFee(uint256 bps) external onlyOwner { /* ... */ }
}
// transfer: current owner calls transferOwnership(next)
//           next calls acceptOwnership()   <-- required second step
```

**Rule of thumb:** if the owner can move money, change payout gates, or rotate a trusted key, use
`Ownable2Step`, not `Ownable`.

### 3. Roles: when there is more than one principal

When a governor (disputes/pauses), a keeper (reports), and an admin (configures) are distinct
actors, don't fold them into one owner. Give each a role and check the specific role per action.

```solidity
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

bytes32 public constant REPORTER_ROLE = keccak256("REPORTER_ROLE");

function report(...) external onlyRole(REPORTER_ROLE) { /* ... */ }
```

- **OpenZeppelin `AccessControl:`** grant/revoke, role admins, enumerable.
- **Solady `OwnableRoles`** (`src/auth/OwnableRoles.sol`): bitmap roles, gas-cheap.
- **Sablier** separates an admin surface (`Adminable`/`RoleAdminable` + a `Comptroller`) from the
  operational surface: a clean model when governance and operations differ.

## Anti-patterns

- `tx.origin` for authorization: phishable; a malicious intermediate contract passes the check. Use
  `msg.sender`.
- One `owner` holding every unrelated power: a single compromised key loses everything.
- Single-step transfer on a funds-affecting contract.

## Checklist

- [ ] Each privileged function checks the narrowest principal that needs it.
- [ ] Funds-affecting ownership uses `Ownable2Step` (or a role with a two-step admin).
- [ ] No `tx.origin` in access checks.
- [ ] Distinct actors (governor/keeper/admin) get distinct roles, not one owner.

## References

- OpenZeppelin `contracts/access/Ownable2Step.sol`, `AccessControl.sol`
- Solady `src/auth/Ownable.sol`, `OwnableRoles.sol`
- Sablier `utils/src/RoleAdminable.sol`, `Adminable.sol`
- Mastering Ethereum, ch. 9: "Smart Contracts Misconfiguration"
