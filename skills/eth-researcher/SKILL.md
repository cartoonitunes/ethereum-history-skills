# eth-researcher

Find undocumented Ethereum contracts and identify candidates for bytecode verification.

## When to Use

Use this skill when asked to:
- Find undocumented Frontier-era contracts
- Research the history of a specific contract or deployer
- Identify crack candidates (unverified bytecode worth investigating)
- Cross-reference on-chain data with historical sources

## Required Tools

- Etherscan API (for on-chain data)
- Brave Search or web_search (for historical context)
- EthereumHistory API (for documentation status)

## Finding Undocumented Contracts

```bash
# Get undocumented contracts from EH, ordered by ETH balance (highest value first)
curl "https://www.ethereumhistory.com/api/agent/contracts?undocumented_only=1&limit=50"

# Filter by era
curl "https://www.ethereumhistory.com/api/agent/contracts?undocumented_only=1&from_timestamp=1438300000&to_timestamp=1438920000&limit=50"
```

## Verifying On-Chain Facts

**Always verify before documenting.** Never trust memory or secondary sources for on-chain data.

```bash
ETHERSCAN_KEY="your-key"

# Get contract creation info
curl "https://api.etherscan.io/v2/api?chainid=1&module=contract&action=getcontractcreation&contractaddresses={address}&apikey=$ETHERSCAN_KEY"

# Check if verified on Etherscan (do this FIRST before cracking)
curl "https://api.etherscan.io/v2/api?chainid=1&module=contract&action=getsourcecode&address={address}&apikey=$ETHERSCAN_KEY"

# Get bytecode
curl "https://api.etherscan.io/v2/api?chainid=1&module=proxy&action=eth_getCode&address={address}&tag=latest&apikey=$ETHERSCAN_KEY"

# Get transaction list
curl "https://api.etherscan.io/v2/api?chainid=1&module=account&action=txlist&address={address}&sort=asc&apikey=$ETHERSCAN_KEY"

# Get ETH balance
curl "https://api.etherscan.io/v2/api?chainid=1&module=account&action=balance&address={address}&tag=latest&apikey=$ETHERSCAN_KEY"
```

## Crack Candidate Scoring

Score candidates by:

| Factor | Weight | Notes |
|---|---|---|
| Historical significance | High | Deployer identity, era, known project |
| ETH balance | Medium | ETH locked = compelling story |
| Transaction count | Medium | Activity = people used it |
| Bytecode size | Medium | 100-1000B = tractable for cracking |
| Source leads | High | GitHub, Reddit, blog posts from the era |

## Researching Deployer Identity

1. Get all contracts deployed by the address
2. Check if address received ETH from known genesis/foundation addresses
3. Search GitHub for the address or associated contract names
4. Search Reddit/bitcointalk for the address
5. Check if address matches any known developer from the era

Known significant deployers to recognize:
- `0x32be343b...` — funded multiple Frontier Day 1 developers
- `0x2605...` — EF-adjacent deployer cluster
- `0xd1220a0c...` — avsa (Alex Van de Sande)
- `0xde0b2956...` — Ethereum Foundation main wallet

## Checking If Already Cracked

Before starting crack work:

1. Check EH API: `GET /api/agent/contracts/{address}`
   - If `verificationMethod` is non-null, it's already cracked
2. Check Etherscan: `getsourcecode` endpoint
   - If `SourceCode` is non-empty, it's Etherscan-verified — use that source, don't crack
3. Check `cartoonitunes` GitHub org for `{name}-verification` repos

## Research Output Format

When documenting research findings, write to a file with:

```markdown
## Contract: {address}

- **Deployed:** Block {n}, {date}
- **Deployer:** {address} ({identity if known})
- **Bytecode:** {size}B
- **ETH:** {balance} ETH
- **Txs:** {count}
- **Etherscan verified:** yes/no
- **EH documented:** yes/no

### Source Leads
- {url}: {why relevant}

### Deployer Context
- {what else deployer deployed}
- {who funded deployer}

### Crack Assessment
- Size: {tractable/large/unknown}
- Compiler era: {solidity v0.x.x / serpent / lll / native solc}
- Difficulty: {low/medium/high}
- Priority: {why}
```
