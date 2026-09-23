# Awesome Solidity Smart Contract Skills for Agents.

Battle-tested smart-contract patterns, packaged as **[Claude Code](https://claude.com/claude-code)
skills** you can drop into your own projects. Each skill is a focused, self-contained playbook for
one recurring Solidity problem: what goes wrong, the pattern that fixes it, a working snippet, a
checklist, and pointers to the reference implementations it was distilled from.

Every skill is grounded in real source read from the protocols indexed by
[awesome-smart-contracts](https://github.com/shafu0x/awesome-smart-contracts) (OpenZeppelin, Solady,
Solmate, Morpho-blue, Sablier, Merit, Uniswap, Aave, …) and the security chapters of
[*Mastering Ethereum*](https://github.com/ethereumbook/ethereumbook).

## Skills

| Skill | What it covers | Reach for it when |
| --- | --- | --- |
| [defensive-programming](skills/defensive-programming/SKILL.md) | The baseline mindset + full antipattern catalogue | Starting any contract, or doing a pre-audit self-review |
| [safe-erc20-transfers](skills/safe-erc20-transfers/SKILL.md) | No-return / fee-on-transfer / approve-race tokens | Calling `transfer`/`transferFrom`/`approve` on a token you don't control |
| [reentrancy-guards](skills/reentrancy-guards/SKILL.md) | CEI, `nonReentrant`, transient-storage guards, read-only reentrancy | Any state-mutating function that makes an external call |
| [two-step-access-control](skills/two-step-access-control/SKILL.md) | `Ownable` vs `Ownable2Step` vs roles; least privilege | Adding any privileged function |
| [pausable-circuit-breaker](skills/pausable-circuit-breaker/SKILL.md) | Emergency stop that halts entry but never traps exits | A protocol that moves money and needs a bug response |
| [eip712-signature-verification](skills/eip712-signature-verification/SKILL.md) | Domain binding, nonces, expiry, malleability, k-of-n | Accepting any signed message (permits, meta-tx, attestations) |
| [escrow-accounting](skills/escrow-accounting/SKILL.md) | Internal ledger, debit-before-transfer, pull payments, state machine | Building escrow, vesting, streaming, splitters, hold-and-release vaults |
| [singleton-vs-clones](skills/singleton-vs-clones/SKILL.md) | Full deploy vs EIP-1167 clones vs params-hash singleton | A factory that creates many instances of the same logic |
| [fixed-point-rounding](skills/fixed-point-rounding/SKILL.md) | Full-precision `mulDiv`, round in the protocol's favor | Computing shares, proportional payouts, interest, prices |
| [oracle-safety](skills/oracle-safety/SKILL.md) | Staleness, bounds, TWAP over spot, report/dispute/apply | On-chain logic depending on any value from outside the contract |
| [proxy-upgrade-safety](skills/proxy-upgrade-safety/SKILL.md) | Disable initializers, guard `_authorizeUpgrade`, storage layout/gaps | A contract sits behind a proxy (Transparent/UUPS/Beacon) or upgrades |

## Install

These are [Claude Code Agent Skills](https://docs.claude.com/en/docs/claude-code/skills). Claude
loads a skill on its own when a task matches the skill's `description`.

**All skills (clone into your project):**

```bash
git clone https://github.com/mystic0xx/awesome-solidity-smart-contracts-skills.git
mkdir -p .claude/skills
cp -r awesome-solidity-smart-contracts-skills/skills/* .claude/skills/
```

**Personal (available in every project):**

```bash
cp -r awesome-solidity-smart-contracts-skills/skills/* ~/.claude/skills/
```

**One skill only:**

```bash
cp -r awesome-solidity-smart-contracts-skills/skills/reentrancy-guards ~/.claude/skills/
```

Each skill is just a directory containing a `SKILL.md` with YAML frontmatter (`name`, `description`)
— so it works anywhere skills are supported, and reads fine as plain documentation even if you don't
use Claude Code at all.

## Using them

- **In Claude Code**, once installed, ask for the work ("add a withdraw function", "verify this
  signed claim") and the matching skill is pulled in automatically. You can also invoke one by name.
- **As a checklist**, open the `SKILL.md` — each ends with a pre-ship checklist you can run against a
  diff by hand or in review.

## Scope

Solidity, on-chain, security and correctness-oriented. These are patterns and disciplines, not a
substitute for an audit. They encode what **audited protocols** already do so your first draft starts
closer to safe.

## Contributing

New skills, corrections, and better reference snippets are welcome via PRs.
See [CONTRIBUTING.md](CONTRIBUTING.md). The bar: every claim traceable to real, audited source or to a cited section of *Mastering Ethereum*.

## Author

Created and maintained by [**mystic0xx**](https://github.com/mystic0xx).

## License

[MIT](LICENSE) — do whatever you want.
