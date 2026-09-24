---
name: private-data-on-chain
description: Never store secrets on-chain `private`/`internal` only hide data from other Solidity, not from anyone reading storage. Commit a salted hash instead. Use whenever a password, key, answer, seed, or any confidential value would otherwise be written to contract state.
---
# Private data is public

## The problem

`private` and `internal` are **compile-time visibility** modifiers. They stop *other Solidity code*
from reading a variable; they do nothing about the actual data. Every storage slot of every contract
is world-readable directly from a node:

```bash
# Read slot 1 of a contract — no ABI, no access control, works on any private var.
cast storage <address> 1
# or eth_getStorageAt(address, slot, block)
```

Transaction calldata is public too, so a "secret" passed as an argument (even to set a private var)
is visible in the mempool and in history forever. The `PasswordStore` audit is the canonical case: a
password kept in a `private` variable, readable by anyone who queries the slot.

Storage slot layout is deterministic, so anyone can compute where to look:

- Fixed vars: slots `0, 1, 2, …` in declaration order.
- Mapping value: `keccak256(abi.encode(key, slotOfMapping))`.
- Dynamic array element: `keccak256(slotOfArray) + index`.

## The rule

1. **Nothing confidential goes on-chain in plaintext:** not in storage, not in calldata, not in
   events.
2. To *commit* to a secret without revealing it, store only a **salted hash**; reveal (if ever)
   later, off-chain or in a controlled reveal step.
3. Encryption doesn't rescue on-chain storage: the ciphertext is still public, and the key can't
   live on-chain either.

## Pattern: commit a salted hash, verify on reveal

```solidity
contract Commitment {
    error BadReveal();
    bytes32 private _commitment;   // still public in storage — but it's only a hash

    // Off-chain: commitment = keccak256(abi.encode(secret, salt)).
    // salt is a large random value that makes brute force / rainbow tables infeasible.
    function commit(bytes32 commitment) external {
        _commitment = commitment;
    }

    function reveal(bytes memory secret, bytes32 salt) external view returns (bool) {
        if (keccak256(abi.encode(secret, salt)) != _commitment) revert BadReveal();
        return true;
    }
}
```

Without the salt, a low-entropy secret (a word, a small number) is trivially recovered by hashing
candidates until one matches. The salt must be high-entropy and kept off-chain until reveal.

## Design notes

- **Reveal front-running.** Once `reveal` hits the mempool, the secret is public before it mines. If
  ordering matters (e.g. a guessing game), gate the reveal or use a two-phase scheme.
- **Events are not private** either they're in the logs bloom and readable by anyone.
- If data must stay confidential *and* be used in computation, that computation belongs off-chain
  (or in a dedicated privacy system); the chain can only verify a proof or a hash of it.

## Checklist

- [ ] No password, private key, seed, or answer is stored in a contract variable, `private` or not.
- [ ] Secrets passed as calldata are treated as public (they are visible in the mempool/history).
- [ ] Commitments store a **salted** hash; the salt is high-entropy and off-chain until reveal.
- [ ] Reveal steps account for mempool front-running.
- [ ] No confidential value is emitted in an event.

## References

- Solidity docs: "Layout of State Variables in Storage" (slot computation)
- Foundry `cast storage`, JSON-RPC `eth_getStorageAt`
- *Mastering Ethereum*, ch. 7: contract data is public on a public blockchain
- Cyfrin security course, no.3: Private data (motivating case: PasswordStore audit)
