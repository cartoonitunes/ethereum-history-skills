# eth-historian

Document Ethereum contracts on EthereumHistory.com via the historian API.

## When to Use

Use this skill when asked to:
- Document a contract on EthereumHistory
- Add history, context, or source links to a contract page
- Submit or update a bytecode verification proof
- Query the EH API for contract information

## Authentication

Requires a historian account. Login at [ethereumhistory.com/historian/login](https://ethereumhistory.com/historian/login).

Set your login token as `EH_TOKEN` in your environment, or load it from your secrets manager.

```bash
EH_TOKEN=your-login-token
```

Login to get a session cookie:
```bash
curl -c /tmp/eh-cookies.txt -X POST https://www.ethereumhistory.com/historian/login \
  -H "Content-Type: application/json" \
  -d '{"email": "your@email.com", "token": "'"$EH_TOKEN"'"}'
```

## Querying a Contract

```bash
curl "https://www.ethereumhistory.com/api/agent/contracts/{address}"
```

Check if already documented:
```bash
curl "https://www.ethereumhistory.com/api/agent/contracts?undocumented_only=1&limit=50"
```

## Documenting a Contract

```bash
curl -b /tmp/eh-cookies.txt -X POST \
  "https://www.ethereumhistory.com/api/contract/{address}/history/manage" \
  -H "Content-Type: application/json" \
  -d '{
    "contract": {
      "etherscanContractName": "ContractName",
      "shortDescription": "One sentence description of what this contract is.",
      "description": "Full historical context. Who deployed it, why it mattered, what happened.",
      "historicalSignificance": "Why this contract matters to Ethereum history.",
      "contractType": "token|wallet|exchange|dao|game|utility|other"
    },
    "links": [
      {
        "title": "Original announcement",
        "url": "https://reddit.com/r/ethereum/...",
        "source": "reddit",
        "note": "Thread where this contract was announced"
      }
    ],
    "deleteIds": []
  }'
```

## Submitting a Verification Proof

Only submit if you have an **exact bytecode match** — init + runtime, byte for byte. See the `eth-cracker` skill for how to verify.

```bash
curl -b /tmp/eh-cookies.txt -X POST \
  "https://www.ethereumhistory.com/api/contract/{address}/history/manage" \
  -H "Content-Type: application/json" \
  -d '{
    "contract": {
      "verificationMethod": "exact_bytecode_match",
      "verificationProofUrl": "https://github.com/cartoonitunes/contractname-verification",
      "verificationNotes": "Compiled with soljson-v0.1.1 optimizer ON. Runtime 144 bytes, exact match.",
      "compilerCommit": "soljson-v0.1.1",
      "compilerLanguage": "solidity",
      "compilerRepo": "https://github.com/ethereum/solc-bin"
    },
    "links": [],
    "deleteIds": []
  }'
```

After submitting, always fire the social bot manually (bot only auto-fires on first edit):

```bash
curl -X POST https://nameless-lake-39668-540f6213f30f.herokuapp.com/contractdocumentation \
  -H "Content-Type: application/json" \
  -d '{
    "contract_address": "0x...",
    "contract_name": "ContractName",
    "deployment_timestamp": 1438919780,
    "short_description": "One sentence description.",
    "contract_url": "https://ethereumhistory.com/contract/0x..."
  }'
```

## Source Link Rules

**DO use:**
- Reddit / bitcointalk threads from the era
- Ethereum Foundation blog posts
- GitHub commits, PRs, issues from the time
- EIP discussions
- Developer blogs
- Contemporaneous news (CoinDesk, etc.)

**DO NOT use:**
- Etherscan links as historical citations (use for on-chain fact verification only)
- Wikipedia
- Any source written after the fact that doesn't cite primary evidence

## Writing Standards

- No em dashes (use hyphens or rewrite)
- No speculation — only what can be sourced
- Short description: one factual sentence, no marketing language
- Description: context, significance, verifiable claims only
- If you're unsure, omit rather than guess

## Field Reference

| Field | Type | Notes |
|---|---|---|
| `etherscanContractName` | string | Display name |
| `tokenName` | string | If it's a token |
| `shortDescription` | string | One sentence |
| `description` | string | Full markdown |
| `historicalSignificance` | string | Why it matters |
| `contractType` | string | Category |
| `verificationMethod` | string | `exact_bytecode_match` only |
| `verificationProofUrl` | string | Public repo URL |
| `verificationNotes` | string | Compiler details |
| `compilerCommit` | string | e.g. `soljson-v0.1.1` |
| `compilerLanguage` | string | `solidity`, `serpent`, `lll` |
| `compilerRepo` | string | Where to find the compiler |
