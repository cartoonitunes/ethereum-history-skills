# eth-cracker

Reproduce exact on-chain bytecode for unverified Frontier-era Ethereum contracts.

## The Standard

**A crack is not done until init bytecode AND runtime bytecode match on-chain, byte for byte.**

The following are NOT cracks:
- Functionally equivalent code that produces different bytecode
- Decompiled output (Dedaub, Panoramix, etc.)
- "Figured out what it does" without compiler proof
- Partial matches (only runtime, only init, or close-but-not-exact)

If you cannot reproduce the exact bytecode from source + compiler in a clean environment, you have not cracked it. Do not submit.

## Step 0: Check If Already Done

Before any crack work:

```bash
ETHERSCAN_KEY="your-key"
ADDRESS="0x..."

# 1. Check Etherscan verification
curl "https://api.etherscan.io/v2/api?chainid=1&module=contract&action=getsourcecode&address=$ADDRESS&apikey=$ETHERSCAN_KEY" \
  | python3 -c "import sys,json; r=json.load(sys.stdin)['result'][0]; print('VERIFIED' if r['SourceCode'] else 'NOT VERIFIED'); print('Name:', r['ContractName'])"

# If SourceCode is non-empty: pull the source, document on EH, done. No cracking needed.

# 2. Check EH database
curl "https://www.ethereumhistory.com/api/agent/contracts/$ADDRESS" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); c=d.get('contract',{}); print('CRACKED' if c.get('verificationMethod') else 'NOT CRACKED')"
```

## Step 1: Extract Target Bytecode

```bash
# Get on-chain runtime bytecode (what's stored in the EVM)
RUNTIME=$(curl -s "https://api.etherscan.io/v2/api?chainid=1&module=proxy&action=eth_getCode&address=$ADDRESS&tag=latest&apikey=$ETHERSCAN_KEY" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['result'][2:])")
echo "Runtime: ${#RUNTIME} chars = $((${#RUNTIME}/2)) bytes"
echo "$RUNTIME" > /tmp/target_runtime.txt

# Get creation bytecode from the deployment transaction
# First get creation tx hash
CREATION_TX=$(curl -s "https://api.etherscan.io/v2/api?chainid=1&module=contract&action=getcontractcreation&contractaddresses=$ADDRESS&apikey=$ETHERSCAN_KEY" \
  | python3 -c "import sys,json; r=json.load(sys.stdin)['result']; print(r[0]['txHash'] if r else 'not found')")

# Then get the input data
curl -s "https://api.etherscan.io/v2/api?chainid=1&module=proxy&action=eth_getTransactionByHash&txhash=$CREATION_TX&apikey=$ETHERSCAN_KEY" \
  | python3 -c "
import sys,json
tx = json.load(sys.stdin)['result']
inp = tx['input'][2:]
# Find f300 (RETURN in creation code) to split init from runtime
idx = inp.find('f300')
if idx >= 0:
    print('Init:', idx//2 + 2, 'B')
    print('Runtime from creation:', (len(inp)-idx-4)//2, 'B')
    rt = inp[idx+4:]
    target = open('/tmp/target_runtime.txt').read().strip()
    print('Runtime matches on-chain:', rt == target)
" 2>/dev/null
```

## Step 2: Identify the Compiler Era

Ethereum contract compilers by era:

| Era | Dates | Compiler | Tool |
|---|---|---|---|
| Pre-Frontier | Before Aug 7 2015 | Native C++ solc (webthree-umbrella) | Docker build |
| Frontier | Aug 7 - Dec 2015 | soljson v0.1.0 - v0.1.6 | Node.js |
| Homestead | Mar 2016 - Oct 2016 | soljson v0.1.7 - v0.3.x | Node.js |
| Later | 2016+ | soljson v0.4.x+ | solc binary |

Clues in the bytecode:
- `60606040` prefix = Solidity (common from v0.1.x)
- `60e060020a` in dispatcher = old-style selector dispatch
- LLL: starts with `(seq ...)` patterns, different structure
- Serpent: similar to Solidity but different codegen for loops/calls

## Step 3: Compiler Sweep (soljson)

Download all soljson versions:
```bash
# List all versions
curl https://ethereum.github.io/solc-bin/bin/list.json | python3 -c "
import sys,json
d=json.load(sys.stdin)
for v in sorted(d['builds'], key=lambda x: x['path']):
    print(v['path'])
" | grep "v0\.[0-3]\." | head -50
```

Compile and compare:
```javascript
// compile.js - run with node
const fs = require('fs');
const Module = require('/tmp/soljson-v0.1.1.js'); // adjust version

const src = fs.readFileSync('/tmp/candidate.sol', 'utf8');
const compile = Module.cwrap('compileJSON', 'string', ['string', 'number']);

// optimizer OFF
const outOff = JSON.parse(compile(src, 0));
// optimizer ON
const outOn = JSON.parse(compile(src, 1));

const target = fs.readFileSync('/tmp/target_runtime.txt', 'utf8').trim();

for (const [name, contract] of Object.entries(outOff.contracts || {})) {
  const bin = contract.bytecode || '';
  const idx = bin.indexOf('f300');
  const rt = idx >= 0 ? bin.substring(idx + 4) : '';
  if (rt === target) console.log(`EXACT MATCH (opt OFF): ${name}`);
  else if (rt.length === target.length) console.log(`Same size, no match (opt OFF): ${name}`);
}

for (const [name, contract] of Object.entries(outOn.contracts || {})) {
  const bin = contract.bytecode || '';
  const idx = bin.indexOf('f300');
  const rt = idx >= 0 ? bin.substring(idx + 4) : '';
  if (rt === target) console.log(`EXACT MATCH (opt ON): ${name}`);
  else if (rt.length === target.length) console.log(`Same size, no match (opt ON): ${name}`);
}
```

## Step 4: Native C++ Solc (for Aug 2015 contracts)

Some Frontier-era contracts were compiled with the native C++ solc built into cpp-ethereum or the webthree-umbrella. The soljson JS versions did not exist yet or produced different codegen.

Use Docker builds for specific compiler commits:

```bash
# webthree-umbrella era (Feb 2016 vintage, produces Frontier-compatible bytecode)
docker run --rm solc-umbrella sh -c \
  'echo "SOURCE" > /tmp/t.sol && /umbrella/build/solidity/solc/solc --bin-runtime /tmp/t.sol'

# Compare output
docker run --rm -v /tmp/candidate.sol:/tmp/t.sol solc-umbrella \
  /umbrella/build/solidity/solc/solc --bin-runtime /tmp/t.sol
```

## Step 5: Function Selector Analysis

Extract selectors from target bytecode to understand the ABI:

```python
target = open('/tmp/target_runtime.txt').read().strip()
data = bytes.fromhex(target)
sels = []
i = 0
while i < min(500, len(data)):
    if data[i] == 0x63 and i+5 < len(data):
        sels.append(data[i+1:i+5].hex())
        i += 5
    else:
        i += 1
print('Selectors:', sels)
```

Look up selectors:
```bash
curl "https://www.4byte.directory/api/v1/signatures/?hex_signature=0x{selector}"
```

Use selectors to reconstruct function signatures, which constrains source reconstruction.

## Step 6: Source Reconstruction

When you have the ABI but not the source:

1. Check dapp-bin (the official example library): `https://github.com/ethereum/dapp-bin`
2. Check early Solidity docs examples via Wayback Machine
3. Search GitHub for function names from the ABI
4. Search Reddit/bitcointalk for the contract address or deployer
5. Check if the deployer deployed other contracts with known source

Common patterns to recognize:
- `SimpleStorage`: `set(uint256)` + `get()`, 144B runtime
- `Greeter`: `greet()` + `kill()` + optional `setGreeting(string)`
- `Mortal`: just `kill()`, extends Greeter
- `Coin/Token`: `sendCoin(address,uint256)` + `coinBalanceOf(address)`
- `NameReg`: `register(bytes32)` + `unregister()` + `owner(bytes32)`

## Step 7: Bytecode Diff Analysis

When you're close but not matching:

```python
target = open('/tmp/target_runtime.txt').read().strip()
candidate = open('/tmp/candidate_runtime.txt').read().strip()

t = bytes.fromhex(target)
c = bytes.fromhex(candidate)

print(f"Target: {len(t)}B, Candidate: {len(c)}B, diff: {len(c)-len(t)}B")

# Find first difference
for i in range(min(len(t), len(c))):
    if t[i] != c[i]:
        print(f"First diff at byte {i} (0x{i:02x}): target=0x{t[i]:02x} candidate=0x{c[i]:02x}")
        print(f"Context: {target[max(0,i*2-10):i*2+20]}")
        break
```

Common causes of near-misses:
- Wrong function declaration order (Solidity optimizer output depends on order)
- Comparison operand flipped (`a > b` vs `b < a` — same logic, different opcode)
- Constructor argument type mismatch
- Wrong optimizer setting (on vs off)
- Wrong compiler version (try adjacent versions)
- Missing or extra state variable

## Step 8: Verify the Match

Before claiming a crack, verify in a clean environment:

```bash
# 1. Compile in fresh environment
node compile.js  # should output EXACT MATCH

# 2. Verify runtime hash matches
python3 -c "
import hashlib
target = open('/tmp/target_runtime.txt').read().strip()
candidate = open('/tmp/candidate_runtime.txt').read().strip()
assert target == candidate, 'MISMATCH'
print('VERIFIED: exact match')
print('Hash:', hashlib.sha256(bytes.fromhex(target)).hexdigest()[:16])
"

# 3. Verify init bytecode if available
# (compare creation TX input with your compiled output)
```

## Step 9: Create Verification Repo

Every proof requires a public repo. Structure:

```
{contractname}-verification/
├── README.md          # What this is, the match result, compiler details
├── {contract}.sol     # The exact source that produces the match
├── verify.js          # Reproducible verification script
├── target_runtime.txt # On-chain bytecode (for reference)
└── RESULTS.md         # Detailed findings
```

README must include:
- Contract address
- Deployment block and timestamp
- Compiler version and optimizer settings
- Bytecode hash (SHA-256 of runtime hex)
- Command to reproduce the verification

## Step 10: Publish

**Note:** Once a contract is verified on EthereumHistory, its proof fields are locked. Only admins can overwrite an existing proof. Do not attempt to re-submit a proof for an already-verified contract unless you have admin access and a legitimate correction.

Once verified, follow the full publishing pipeline:

1. Create verification repo in the `cartoonitunes/` GitHub org
2. Add row to `repos/awesome-ethereum-proofs/README.md`
3. Submit to EthereumHistory via `eth-historian` skill (manage endpoint)
4. Manually fire social bot (bot only auto-fires on first edit per contract)
5. Confirm contract appears on [ethereumhistory.com/proofs](https://ethereumhistory.com/proofs)

Do not claim a contract is cracked until all 5 steps are complete.

## Common Mistakes

| Mistake | Why it fails |
|---|---|
| Submitting without exact match | Automated check will reject it |
| Using decompiled code as "source" | Decompilation produces functionally equivalent code, not original |
| Wrong function order | Solidity optimizer output is order-dependent |
| Claiming "same logic" | The standard is same bytes, not same behavior |
| Skipping the repo | No public repo = no accepted proof |
| Starting crack work before checking Etherscan | Wasted effort if already verified |
