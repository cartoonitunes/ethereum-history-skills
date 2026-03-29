---
name: eth-bytecode-cracker
description: Reproduce exact on-chain bytecode for unverified frontier-era Ethereum contracts (2015-2016). Use when user says "crack contract", "verify bytecode", "match bytecode", "reproduce bytecode", "frontier contract", or wants to reverse-engineer and reproduce the exact compiled output of an early Ethereum contract. Covers bytecode analysis, source reconstruction, compiler sweep, permutation cracking, and the full publishing pipeline.
---

# Ethereum Bytecode Cracker

Reproduce byte-for-byte exact bytecode for unverified frontier-era Ethereum contracts (Aug 2015 - mid 2016).

## What "Cracked" Means

**Byte-for-byte match of both creation and runtime bytecode.** Not decompilation, not documentation, not "figuring out what it does." A contract is cracked ONLY when compiled source produces the exact on-chain bytecode.

## Workflow Overview

1. **Fetch on-chain bytecode** (creation + runtime)
2. **Extract selectors** and identify functions
3. **Trace bytecode** to understand storage layout, logic, events
4. **Reconstruct Solidity source**
5. **Compile and compare** across compiler versions + optimizer settings
6. **Permutation crack** function declaration order (critical for optimizer output)
7. **Publish** via the verification pipeline

## Step 1: Fetch Bytecode

```bash
# Runtime bytecode
curl -s "https://api.etherscan.io/v2/api?chainid=1&apikey=YOUR_ETHERSCAN_API_KEY&module=proxy&action=eth_getCode&address=0xCONTRACT&tag=latest" | python3 -c "import sys,json; print(json.load(sys.stdin)['result'][2:])" > /tmp/target_runtime.hex

# Creation bytecode (from deploy tx)
curl -s "https://api.etherscan.io/v2/api?chainid=1&apikey=YOUR_ETHERSCAN_API_KEY&module=proxy&action=eth_getTransactionByHash&txhash=0xDEPLOY_TX" | python3 -c "import sys,json; print(json.load(sys.stdin)['result']['input'][2:])" > /tmp/target_creation.hex
```

Save byte counts:
```bash
echo "Runtime: $(wc -c < /tmp/target_runtime.hex | tr -d ' ') hex chars = $(( $(wc -c < /tmp/target_runtime.hex | tr -d ' ') / 2 )) bytes"
echo "Creation: $(wc -c < /tmp/target_creation.hex | tr -d ' ') hex chars = $(( $(wc -c < /tmp/target_creation.hex | tr -d ' ') / 2 )) bytes"
```

## Step 2: Extract Selectors

Look for the dispatch table pattern. In v0.1.x compiled contracts:
- `PUSH4 <selector> ... EQ JUMPI` pattern
- Extract all 4-byte selectors

Look up selectors:
```bash
curl -s "https://api.openchain.xyz/signature-database/v1/lookup?function=0xSEL1,0xSEL2,0xSEL3&filter=true"
```

Note: 4byte.directory returns 403. Use openchain.xyz only.

## Step 3: Analyze Bytecode

Key patterns to identify:
- **Storage layout**: Which slots are used, mapping vs value, struct packing
- **Events**: Count LOG0-LOG4 opcodes. Zero LOG = no events.
- **External calls**: Count CALL/DELEGATECALL opcodes
- **Compiler era**: Dispatch pattern `60e060020a` = v0.1.x with optimizer
- **SLOAD/SSTORE count**: Tells you how many state reads/writes per function
- **TIMESTAMP**: Indicates time-dependent logic

## Step 4: Reconstruct Source

Write Solidity targeting the identified compiler version. Key rules:

- **Function visibility**: Early Solidity had no explicit visibility. All functions public by default.
- **No constructor keyword**: Use `function ContractName()` pattern
- **Data types**: `uint` = `uint256`, `address`, `bytes32`, `bool`
- **Mappings vs structs**: Separate mappings produce MORE bytecode than structs (more SLOAD ops). Match the SLOAD count.

## Step 5: Compile and Compare

### Compiler cache

soljson compilers cached at `/tmp/soljson/`:
- Named versions: `soljson-v0.1.1+commit.6ff4cd6.js` through `soljson-v0.3.6+commit.3fc68da.js`
- Shorthand aliases: `v011.js`, `v0.1.2.js`, etc.

### Compilation script

Use Node.js with solc wrapper. See `scripts/compile-and-compare.js` in this skill directory.

```bash
node ~/.agents/skills/eth-bytecode-cracker/scripts/compile-and-compare.js \
  /tmp/candidate.sol ContractName /tmp/soljson/soljson-v0.1.1+commit.6ff4cd6.js \
  /tmp/target_runtime.hex /tmp/target_creation.hex
```

Outputs: match status, byte-level diff position, compiled hex files.

### Native C++ solc (Docker) for pre-0.1.1 era

```bash
# solc-jan20 (v0.2.0, Jan 20 2016 commit)
docker run --rm solc-jan20 sh -c 'echo "SOURCE" > /tmp/t.sol && /umbrella/build/solidity/solc/solc --bin-runtime /tmp/t.sol'

# solc-poc8 (v0.8.2 internal, Feb 2015 pre-release)
docker run --rm solc-poc8 sh -c 'echo "SOURCE" > /tmp/t.sol && /umbrella/build/solidity/solc/solc --bin-runtime /tmp/t.sol'
```

## Step 6: Permutation Cracking

**This is the key technique.** Function declaration order in source code affects optimizer output in v0.1.x-v0.3.x compilers. The optimizer's shared subroutine placement depends on which function first triggers the trampoline.

See `scripts/permutation-cracker.js` for the automated tool.

```bash
node ~/.agents/skills/eth-bytecode-cracker/scripts/permutation-cracker.js \
  /tmp/candidate.sol ContractName /tmp/soljson/soljson-v0.1.1+commit.6ff4cd6.js \
  /tmp/target_runtime.hex /tmp/target_creation.hex
```

This generates all permutations of function declarations and tests each one. For 6 functions = 720 permutations. For 6 functions + operand flip variants = 4320+ combos.

### Operand order matters

`0 == m_record[name].owner` produces different bytecode than `m_record[name].owner == 0`. The EQ opcode gets different stack ordering. Generate variants for each comparison.

## Step 7: Publishing Pipeline (MANDATORY)

Once cracked (byte-for-byte match), run ALL four steps:

### 7a. Add proof to awesome-ethereum-proofs

**New proofs go in `cartoonitunes/awesome-ethereum-proofs` under `proofs/<contractname>/`** — not a separate repo.

```
proofs/
  logger-verification/
    Logger.sol
    README.md
    target_runtime.txt
    verify.js
```

Structure:
- `<ContractName>.sol` — source code
- `README.md` — address, compiler, optimizer, SHA-256 hashes, **Proved by** field, and verify instructions
- `target_runtime.txt` — on-chain runtime hex
- `verify.js` (or similar) — reproducible script that downloads the compiler and checks the match

README template:
```markdown
| Field | Value |
|-------|-------|
| Address | `0xADDRESS` |
| Deployed | Mmm DD, YYYY (block N) |
| Compiler | soljson-vX.X.X+commit.XXXXXXX |
| Optimizer | ON / OFF |
| Runtime | N bytes |
| Creation | N bytes |
| Runtime SHA-256 | `...` |
| Creation SHA-256 | `...` |
| Proved by | [@Name](https://ethereumhistory.com/historian/ID) |
```

The `Proved by` field is mandatory — it's the canonical attribution for the proof, independent of git history.

Clone the repo, add your proof folder, open a PR. Git config: `user.name "cartoonitunes"`, `user.email "cartoonitunes@users.noreply.github.com"`

**Note:** Individual verification repos (`cartoonitunes/*-verification`) are legacy. Don't create new ones. Existing ones remain as-is — their `verificationProofUrl` links still work.

### 7b. Add row to the README table

In the same PR, add a row to `README.md` in chronological order by deployment date:

```
| [ContractName](https://ethereumhistory.com/contract/0xADDR) | Mmm DD, YYYY (block N) | soljson vX.X.X (optimizer ON/OFF) | Exact bytecode match | [Proof](proofs/contractname/) |
```

Note the `[Proof](proofs/contractname/)` link format — points to the subfolder in the same repo, not an external repo.

### 7c. Document on EthereumHistory

POST to ethereumhistory.com API. **Read this carefully — two non-obvious rules:**

**Rule 1: `verificationStatus` is computed from `sourceCode`.** The site returns `"verified"` only when `sourceCode` is present in the DB. Setting `verificationMethod`, `compilerCommit`, etc. without `sourceCode` leaves the contract as `"decompiled"`. Always include `sourceCode` in the same request.

**Rule 2: All field names are camelCase.** Snake_case fields are silently ignored.

```bash
# Login
curl -c /tmp/eh_cookies.txt -X POST "https://www.ethereumhistory.com/api/historian/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"YOUR_EMAIL","token":"YOUR_TOKEN"}'
COOKIE=$(grep "eh_historian" /tmp/eh_cookies.txt | awk '{print $NF}')

# Publish — include sourceCode or it won't show as verified
curl -X POST "https://www.ethereumhistory.com/api/contract/0xADDRESS/history/manage" \
  -H "Content-Type: application/json" \
  -H "Cookie: eh_historian=$COOKIE" \
  -d '{
    "contract": {
      "verificationStatus": "verified",
      "verificationMethod": "exact_bytecode_match",
      "compilerLanguage": "solidity",
      "compilerCommit": "soljson vX.X.X (optimizer ON/OFF)",
      "verificationProofUrl": "https://github.com/cartoonitunes/REPO",  // ← this URL appears as the "View Verification Proof" button on the contract page
      "verificationNotes": "Exact bytecode match. Runtime: X bytes.",
      "sourceCode": "contract Foo { ... }"
    },
    "links": [],
    "deleteIds": []
  }'
```

**Verification methods:** `exact_bytecode_match` | `etherscan_verified` | `author_published` | `source_reconstructed`

**If contract is already Etherscan-verified:** use `etherscan_verified` method, NOT `exact_bytecode_match`. Check Etherscan before doing any crack work (see pre-flight in AGENTS.md).

After publishing, manually fire the bot (the auto-trigger only fires on first-ever edit; if you touched the contract earlier it won't fire automatically):
```bash
curl -X POST "https://nameless-lake-39668-540f6213f30f.herokuapp.com/contractdocumentation" \
  -H "Content-Type: application/json" \
  -d '{"contract_address":"0xADDR","contract_name":"Name","deployment_timestamp":"2015-...","short_description":"...","contract_url":"https://ethereumhistory.com/contract/0xADDR"}'
```

### 7d. Verify on /proofs page

Confirm contract appears at `ethereumhistory.com/proofs`.

## Known Compiler Quirks

- **v0.1.1**: Function order affects optimizer. Operand order in comparisons matters. `0 == x` vs `x == 0` produce different EQ stack patterns.
- **v0.1.1-v0.2.0**: `changeOwner` vs `setOwner` naming affects selector AND optimizer layout.
- **Struct vs mappings**: Structs produce fewer SLOADs. If on-chain has many SLOADs, original likely used separate mappings.
- **Constructor**: `function ContractName()` not `constructor()`.
- **No events = no LOG opcodes**: If zero LOG opcodes on chain, source has zero `event` declarations.
- **Public state variables**: Auto-generate getter functions that appear as selectors.

## Unknown Selector / Event Hash Recovery

When selectors or event hashes aren't in any database (4byte.directory, OpenChain, samczsun), use these techniques in order:

### 1. Signature database lookup (fast, try first)
```bash
# OpenChain (preferred)
curl -s "https://api.openchain.xyz/signature-database/v1/lookup?function=0xSELECTOR"
# 4byte.directory (functions)
curl -s "https://www.4byte.directory/api/v1/signatures/?hex_signature=0xSELECTOR"
# 4byte.directory (events)
curl -s "https://www.4byte.directory/api/v1/event-signatures/?hex_signature=0xEVENT_HASH"
```

### 2. Semantic guessing (fast, try 50-100 common names)
Test common Solidity function names against target selectors using keccak256. Cover:
- Token patterns: `paused`, `frozen`, `listed`, `tradingEnabled`, `mintable`
- ICO patterns: `closeSale`, `saleActive`, `crowdsaleOpen`, `icoEnded`
- Admin patterns: `setStatus`, `setActive`, `setEnabled`, `setReady`
- Test both `setName(bool)` setter and `name()` getter patterns

### 3. C keccak brute-forcer (for unknown function/event names)
When the event hash is a **full 32-byte keccak match**, brute-force the event name.
Compile with `-O3`, runs at ~27M hashes/sec on M-series Mac:

```c
// Key pattern: test name as Event(bool), getter name(), setter setName(bool) simultaneously
// See /tmp/keccak_crack2.c for full implementation
// Coverage: length 1-5 in ~2min, length 6 in ~38min, length 7 in ~40 hours
gcc -O3 -o /tmp/keccak_crack /tmp/keccak_crack2.c
/tmp/keccak_crack 8  # search names up to 8 chars
```

Strategy: Events have FULL 32-byte hashes (cryptographically unique), so finding the event name
gives you the exact function name. Function selectors are only 4 bytes (collisions possible).

### 4. Factory bytecode extraction
If the contract was created by a factory:
- Fetch the factory's runtime bytecode
- Search for the token's runtime bytecode embedded in the factory (it's stored verbatim)
- The factory was compiled from a single Solidity file containing both factory + token definitions
- This confirms the token source is part of a larger compilation unit

### 5. Related repos / security audits
Search GitHub for the project name + "audit" or "token" or "contract":
```bash
curl -s "https://api.github.com/search/repositories?q=PROJECTNAME&sort=stars"
```
Security auditors (bokkypoobah, OpenZeppelin, ConsenSys) often publish original source code
alongside their audit reports. The ORIGINAL pre-audit source is the one that matches on-chain.

### 6. On-chain event log analysis
If the mystery function was ever called, its event logs are on-chain:
```bash
# Get logs for the event topic
curl -s "https://api.etherscan.io/v2/api?chainid=1&apikey=KEY&module=logs&action=getLogs&address=0xADDR&topic0=0xEVENT_HASH&fromBlock=0&toBlock=latest"
```

## Key Lessons (Post-Frontier Era, 0.4.x+)

- **String utility code size varies between compiler versions** — even patch versions can differ by 30-200 bytes in string getter/setter utility functions
- **`throw` vs `require()`**: `throw` compiles to INVALID (0xfe, 1 byte). `require()` compiles to ISZERO+JUMPI+REVERT (~5 bytes). Count REVERT vs INVALID opcodes to determine which was used.
- **Factory-created contracts**: The token bytecode is embedded verbatim in the factory runtime. If you can find the factory, you have a byte-for-byte copy of the token creation code.
- **bool public variables**: Auto-generate getter functions. `bool public isLocked` creates `isLocked()` getter automatically.
- **Dispatch entry sizes**: Compare dispatch table entry sizes between compiled and on-chain. If all match, the source has the right functions — the gap is in internal body code (usually string utilities).

## Candidate Selection

Frontier contracts with ETH balance are prioritized. Balance list at `memory/frontier-contract-balances.json`. Skip contracts already verified on Etherscan. Skip contracts already in awesome-ethereum-proofs.

### Native C++ solc Binaries Repository
Pre-built native C++ solc binaries for frontier-era bytecode archaeology:
**https://github.com/cartoonitunes/solc-native-builds**

10 binaries covering Feb 2015 - Feb 2016 (poc-8 through webthree-umbrella v1.1.2).
These produce different bytecode than soljson/npm builds — critical for exact matches.
