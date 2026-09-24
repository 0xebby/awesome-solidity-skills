---
name: eip712-signature-verification
description: Verify off-chain signatures safely: EIP-712 domain binding, single-use nonces, expiry, and malleability rejection. Use whenever a contract accepts a signed message (permits, meta-tx, attestations, claims, gasless approvals).
---
# EIP-712 signature verification

## The principle

A signature is a **bearer token**. Whoever holds it can present it. It is only safe if the signed
payload pins down everything that matters and can be used exactly once:

1. **What and for whom:** the struct fields (amount, recipient, subject).
2. **Where:**`chainId` + `verifyingContract`, so it can't be replayed on another chain or a
   sibling deployment.
3. **When it stops being valid:** an `expiresAt`.
4. **Only once:** a nonce or a consumed-id set.
5. **Not malleable:** reject the high-`s` half; never treat `ecrecover`'s `address(0)` as valid.

## Pattern: OpenZeppelin EIP712 + ECDSA

```solidity
import {EIP712} from "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import {ECDSA}  from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract Claimer is EIP712 {
    bytes32 private constant CLAIM_TYPEHASH =
        keccak256("Claim(address subject,uint256 amount,uint256 nonce,uint64 expiresAt)");

    mapping(address => uint256) public nonces;   // single-use, per subject

    constructor() EIP712("MyProtocol", "1") {}   // name + version -> domain

    function claim(address subject, uint256 amount, uint64 expiresAt, bytes calldata sig) external {
        require(block.timestamp <= expiresAt, "expired");

        uint256 nonce = nonces[subject];
        bytes32 structHash = keccak256(abi.encode(CLAIM_TYPEHASH, subject, amount, nonce, expiresAt));
        bytes32 digest = _hashTypedDataV4(structHash);              // binds chainId + this contract

        (address signer, ECDSA.RecoverError err, ) = ECDSA.tryRecover(digest, sig);
        require(err == ECDSA.RecoverError.NoError && signer == subject, "bad sig");

        nonces[subject] = nonce + 1;               // consume BEFORE effects
        // ... effects ...
    }
}
```

## Why each piece matters

- **`_hashTypedDataV4`** folds the domain separator (name, version, `chainId`, `verifyingContract`)
  into the digest. Without it, a signature valid on testnet is valid on mainnet, and one valid for
  contract A is valid for contract B.
- **`ECDSA.tryRecover`** rejects malleable signatures (both `(r,s)` and `(r,-s mod n)` would
  otherwise recover the same signer, letting an attacker mint a *second distinct* signature for the
  same message) and returns an explicit error rather than `address(0)`. Never gate on
  `signer != address(0)` from raw `ecrecover`.
- **Nonce / consumed-id:** increment (or mark used) before doing the work, so a re-submission of the
  same signature fails. For multi-signer schemes, also enforce *distinct* signers so one key can't
  fill a k-of-n threshold alone.

## Threshold (k-of-n) attestations

When multiple signers must agree, additionally: bound the array length (an O(n²) distinctness scan
must not be griefable), check each signer is in the allowed set, reject duplicate signers, and
verify each has the expected nonce. Derive a unique bundle id from the accumulated struct hashes and
mark it consumed.

## Checklist

- [ ] Struct includes `chainId`+contract binding (via `_hashTypedDataV4`), an expiry, and a nonce/id.
- [ ] `ECDSA.tryRecover` (not raw `ecrecover`); malleability + zero-address handled.
- [ ] Nonce consumed / id marked used before effects.
- [ ] Multi-sig: distinct signers enforced, array length bounded, each signer authorized.
- [ ] Typehash string matches the struct exactly (field order and types).

## References

- OpenZeppelin `contracts/utils/cryptography/EIP712.sol`, `ECDSA.sol`, `Nonces.sol`
- Solady `src/utils/EIP712.sol`, `ECDSA.sol`
- EIP-712; EIP-2612 (permit)
- Mastering Ethereum, ch. 9: "Signature Replay Attack"
