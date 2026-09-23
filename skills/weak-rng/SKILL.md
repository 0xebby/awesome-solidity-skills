---
name: weak-rng
description: Generate randomness that can't be predicted or manipulated. Never seed from block data (timestamp, prevrandao, blockhash); use Chainlink VRF or a committed future value. Use for lotteries, raffles, NFT trait rolls, random selection, or any outcome with value at stake.
---
# Weak on-chain randomness

## The problem

The EVM is deterministic and fully public: every node must compute the same result, so there is no
private entropy on-chain. Any "random" seed built from block data is either **predictable** (a
contract can read the same values in the same transaction and pre-compute the outcome) or
**influenceable** (the block proposer chooses or withholds it). *Mastering Ethereum* calls this the
**Entropy Illusion**.

Sources that are all unsafe for value-bearing randomness:

- `block.timestamp`, `block.number`, `block.difficulty` / `block.prevrandao`, `blockhash(...)`,
  `gasleft()` — readable by any contract in the same transaction.
- `keccak256(abi.encode(msg.sender, block.timestamp, block.prevrandao))` — the canonical broken
  raffle seed. An attacker computes the winning index in a `view` and only enters when they win.
- `blockhash(block.number)` is always `0` (the current block hash isn't known yet), and
  `blockhash` returns `0` for blocks older than 256.

## The rule

1. **Never derive randomness from block data** for anything an attacker profits from manipulating.
2. Use a **verifiable external source** (Chainlink VRF) or a **commit–reveal** scheme bound to a
   value that is fixed *after* all participants commit.

## Pattern: Chainlink VRF (verifiable randomness)

```solidity
import {VRFConsumerBaseV2Plus} from "@chainlink/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol";
import {VRFV2PlusClient} from "@chainlink/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol";

contract Raffle is VRFConsumerBaseV2Plus {
    error Raffle__RequestUnknown();
    mapping(uint256 requestId => address[] entrants) private _entrants;

    function drawWinner(bytes32 keyHash, uint256 subId) external returns (uint256 requestId) {
        requestId = s_vrfCoordinator.requestRandomWords(
            VRFV2PlusClient.RandomWordsRequest({
                keyHash: keyHash, subId: subId, requestConfirmations: 3,
                callbackGasLimit: 200_000, numWords: 1,
                extraArgs: VRFV2PlusClient._argsToBytes(
                    VRFV2PlusClient.ExtraArgsV1({nativePayment: false}))
            })
        );
        _entrants[requestId] = _currentEntrants;   // snapshot the field for this draw
    }

    // Called back by the coordinator with proof-verified randomness. Do the payout here.
    function fulfillRandomWords(uint256 requestId, uint256[] calldata words) internal override {
        address[] storage e = _entrants[requestId];
        if (e.length == 0) revert Raffle__RequestUnknown();
        address winner = e[words[0] % e.length];
        // effects before any external transfer (see reentrancy-guards)
    }
}
```

VRF returns randomness with an on-chain-verified proof, so neither the caller nor the node operator
can bias or predict it.

## Alternative: commit–reveal

When an oracle isn't available, have each participant submit `keccak256(secret, salt)` first, then
reveal `secret` after commits close; combine the revealed secrets. The known weakness: **the last
revealer can withhold** their reveal if they dislike the outcome. Mitigate with a deposit slashed on
non-reveal, and never let a single party's reveal be the only entropy.

`block.prevrandao` (the RANDAO beacon) is *not* a fix: the proposer can bias the last bit and
withhold blocks. Acceptable only for outcomes where a 1-bit bias and re-roll are worth nothing.

## Checklist

- [ ] No lottery/selection/trait outcome is seeded from `block.*`, `blockhash`, or `gasleft()`.
- [ ] Value-bearing randomness comes from Chainlink VRF (or an equivalent verifiable source).
- [ ] Payout/selection happens in the VRF callback, not in the request transaction.
- [ ] Commit–reveal schemes penalize non-reveal and don't rely on one party's entropy.
- [ ] Entrant set is snapshotted at request time so it can't change before fulfillment.

## References

- Chainlink `@chainlink/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol`
- Solidity docs: `block` and `blockhash` semantics (blockhash valid for last 256 blocks only)
- *Mastering Ethereum*, ch. 9: "Entropy Illusion"
- Cyfrin security course, §4: Weak RNG (motivating case: Puppy Raffle)
