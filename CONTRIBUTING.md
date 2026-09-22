# Contributing

Thanks for helping grow this collection. Pull requests are welcome — new skills, corrections to
existing ones, better reference snippets, and gas/security updates as the ecosystem moves.

## The one rule

**Every claim must be traceable to real, audited source or to a cited section of *Mastering
Ethereum*.** This repo's value is that nothing in it is guessed. If you can't point to where a
pattern is actually used in a reputable codebase (OpenZeppelin, Solady, Solmate, a top-tier
protocol) or to a specific security reference, it doesn't go in.

## Adding a new skill

1. Create `skills/<your-skill-name>/SKILL.md`. Use kebab-case for the directory name.
2. Start the file with YAML frontmatter:

   ```yaml
   ---
   name: your-skill-name          # must match the directory name
   description: One or two sentences. Say what it does AND when to use it — this is what Claude matches on.
   ---
   ```

3. Follow the structure of the existing skills:
   - **The problem / principle** — what goes wrong, or the rule.
   - **Pattern** — a minimal, compilable Solidity snippet.
   - **Design notes / variants** — tradeoffs, gas, alternatives.
   - **Checklist** — a `- [ ]` list a reviewer can run against a diff.
   - **References** — file paths in the source repos and/or *Mastering Ethereum* sections.
4. Add a row to the **Skills** table in `README.md`.

## Editing an existing skill

- Keep the tone factual and dense. State what a thing is, then stop.
- Don't add a rejected-alternative essay; a one-line tradeoff note is enough.
- If you change a claim, update or add its reference.

## Style

- Solidity snippets should compile in spirit (imports shown, `pragma`-agnostic) and be as short as
  the point allows.
- Prefer custom errors over string reverts in examples.
- Keep line length reasonable (~100 cols) so diffs read well.
- Neutral, vendor-agnostic where possible; name a specific library when it's the reference for the
  pattern.

## What gets rejected

- Unsourced patterns or "I think this is how it works."
- Skills that duplicate an existing one — extend the existing skill instead.
- Anything encouraging insecure shortcuts (skipping validation, disabling guards for gas, etc.)
  without a clearly documented, bounded rationale.
- Marketing for a specific protocol or token.

## Local check

Frontmatter is validated in CI (see `.github/workflows/validate.yml`). To check locally that every
skill has a `name` and `description` and the name matches its directory:

```bash
for f in skills/*/SKILL.md; do
  dir=$(basename "$(dirname "$f")")
  grep -q "^name: $dir$" "$f" || echo "MISMATCH: $f (name must equal '$dir')"
  grep -q "^description:" "$f" || echo "MISSING description: $f"
done
```

By contributing you agree your work is released under the repository's [MIT License](LICENSE).
