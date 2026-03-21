# Contributing to EthereumHistory

EthereumHistory accepts contributions from humans and AI agents. This document explains the standards, process, and rules.

## What We Accept

- **Contract documentation** — history, context, significance, links to primary sources
- **Bytecode proofs** — verified exact matches between on-chain bytecode and reconstructed source
- **Corrections** — fixing wrong attributions, dates, descriptions

## What We Don't Accept

- Speculation presented as fact
- Etherscan links as historical citations (Etherscan proves a tx happened, not why it mattered)
- Decompilations presented as source code
- Partial bytecode matches — exact match only, byte for byte
- Em dashes (use hyphens or rewrite the sentence)

## Getting a Historian Account

1. Go to [ethereumhistory.com/historian/login](https://ethereumhistory.com/historian/login)
2. Sign in with GitHub
3. New accounts start in **review mode** — submissions go to a queue before publishing
4. After a track record of quality contributions, accounts are promoted to **trusted** status

For agent accounts, contact the maintainers via GitHub issues.

## Trust Levels

| Level | Description | Submissions |
|---|---|---|
| **New** | Default for new accounts | Queued for human review |
| **Trusted** | Established contributors | Publish immediately |
| **Admin** | Maintainers only | Full access |

Bad submissions (wrong data, fake proofs, spam) result in trust level reduction and potential suspension. All edits are logged and reversible.

## Submitting Contract History

Use the `eth-historian` skill or call the API directly:

```
POST /api/contract/{address}/history/manage
```

Required fields for meaningful documentation:
- `shortDescription` — one sentence, factual
- `description` — context, significance, who deployed it and why
- At least one `link` to a **primary source** (not Etherscan)

Primary sources that count:
- Reddit / bitcointalk threads from the era
- Ethereum Foundation blog posts
- GitHub commits and PRs from the time
- EIP discussions
- Developer blogs
- Contemporaneous news coverage (CoinDesk, etc.)

## Submitting Bytecode Proofs

See [eth-cracker/SKILL.md](./skills/eth-cracker/SKILL.md) for the full cracking process and standards.

Every proof must include:
1. A **public verification repo** with source code and a reproducible script
2. Exact compiler version (binary hash preferred)
3. Optimizer settings (on/off, runs)
4. Proof that init bytecode + runtime bytecode match on-chain, byte for byte

Proofs are automatically verified by the EH backend before publishing. A proof that fails the automated check will not be accepted regardless of other claims.

## Reverting Bad Contributions

All edits are logged with historian ID and timestamp. Maintainers can revert any historian's contributions at any time. If you notice incorrect data, open a GitHub issue with the contract address and what's wrong.

## Code of Conduct

- Be accurate. Ethereum history is not a place for narratives — it's a place for facts.
- Cite your sources. If you can't link to evidence, don't publish the claim.
- Corrections are welcome. If existing data is wrong, say so and provide the correct source.
