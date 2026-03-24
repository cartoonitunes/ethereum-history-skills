# Skill: eth-historian

Document Ethereum contract history on ethereumhistory.com as Neo.

## When to use
- Julian asks to document a contract
- Heartbeat cadence: 1 contract per day (from queue in `tasks/eth-history-contracts-to-document.md`)
- Any "document [contract name/address]" request

## Neo's Historian Account

- **Site:** https://www.ethereumhistory.com
- **Name:** Neo
- **Email:** neo@openclaw.ai
- **Token:** `neo-historian-d4b105db78f760f0abcc58c13c4452f2`
- **Login:** https://www.ethereumhistory.com/historian/login (email + token above)
- **Trusted:** yes — can create invites, upload media, full edit access

Account creation requires a **trusted historian to generate an invite** at `/historian/invite`.
Ask Julian to create an invite if not yet done. Once created, store the login token in TOOLS.md.

## Research Workflow

### Step 1: Run the research script

```bash
bash ~/workspace/scripts/eth-history-research.sh <address> --out /tmp/<name>-research.md
```

This fetches (all free, no key required except Brave):
- **Blockscout API** — contract name, creator, deploy block, source code, ABI
- **CoinGecko API** — token price, supply, holders, genesis date (if token)
- **Brave Search** — Reddit threads mentioning the address (up to 6)
- **old.reddit.com** — top comments from each thread

Output: a structured Markdown report with on-chain facts + Reddit evidence.

### Step 2: Supplement the research

The script gets you 70% there. Fill gaps manually:

**For early contracts (pre-2018):** Reddit search may not find launch discussions.
Try these supplemental searches using `web_search`:
```
web_search: "0xADDRESS" site:reddit.com/r/ethereum
web_search: "CONTRACT_NAME" site:reddit.com/r/ethereum 2015 OR 2016 OR 2017
web_search: "CONTRACT_NAME" ethereum announcement
web_search: "CONTRACT_NAME" site:medium.com OR site:blog.ethereum.org
```

Also check:
- **Wayback Machine:** `https://web.archive.org/web/*/ethereumhistory.com/contract/<address>`
- **GitHub:** search for the contract address in repos from that era
- **Etherscan comments:** `https://etherscan.io/address/<address>#comments`

**Creator identity:** Look up the creator address on Etherscan for ENS name,
other deployed contracts, and social links. Cross-reference on Twitter/X.

### Step 3: Write the story

Fill in the checklist from the research report:
- What was the purpose of this contract?
- Who deployed it and why? (use ENS/known identity if verifiable)
- What date? (Blockscout block → date, or Etherscan)
- Community reaction at launch (Reddit posts from the time)
- Historically significant because? (first of its kind? inspired later projects?)
- Current status (still active? deprecated? migrated?)

### Step 4: Submit to ethereumhistory.com via API

Login and get cookie:
```bash
curl -sc /tmp/neo-cookies.txt -X POST "https://www.ethereumhistory.com/api/historian/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"neo@openclaw.ai","token":"neo-historian-d4b105db78f760f0abcc58c13c4452f2"}'
```

Submit documentation (use cookie for auth):
```bash
ADDR="0xcontractaddress"
curl -sb /tmp/neo-cookies.txt -X POST "https://www.ethereumhistory.com/api/contract/${ADDR}/history/manage" \
  -H "Content-Type: application/json" \
  -d '{
  "contract": {
    "etherscanContractName": "ContractName",
    "shortDescription": "1-sentence summary",
    "description": "Full story, 2-5 paragraphs, Markdown OK",
    "historicalSignificance": "Why it matters historically",
    "historicalContext": "What was happening in Ethereum at the time"
  },
  "links": [
    {
      "title": "Link display title",
      "url": "https://...",
      "source": "Reddit"
    }
  ]
}'
```

**Field names (exact):**
- Contract: `etherscanContractName`, `shortDescription`, `description`, `historicalSignificance`, `historicalContext`
- Links: `title` (not `label`), `url`, `source` (not `type`), `note` (optional)
- To delete links: add `"deleteIds": [id1, id2]` to the body

**Verify submission:**
```bash
curl -s "https://www.ethereumhistory.com/api/agent/contracts/${ADDR}" | python3 -c "
import sys,json; d=json.load(sys.stdin)['data']
print('desc:', len(d.get('description') or ''), 'chars')
print('significance:', (d.get('historical_significance') or '')[:80])
"
```

First save auto-posts to erc20sales bot (for contracts with names). Subsequent edits do not re-post.

## Quality Standards

**The cardinal rule: No hallucinated facts.**

Every claim must come from:
- On-chain data (Blockscout, Etherscan)
- CoinGecko verified data
- Reddit posts/comments with links
- Verified primary sources (blog posts, GitHub, official announcements)

**Red flags to check:**
- Deployment date — verify via Blockscout block number and timestamp
- Creator identity — don't assume; verify via ENS or known contracts
- Token supply/holders — pull from on-chain, not memory
- "First ever" claims — search extensively before making this claim
- Community reaction — must have an actual Reddit post/comment to cite

**Tone:** Neutral, encyclopedic, factual. No marketing language. No speculation.
Match Wikipedia/encyclopedia style, not a blog post.
**No em dashes (—).** Use a hyphen, comma, or rewrite the sentence. Zero exceptions.

## Contract Queue

See `tasks/eth-history-contracts-to-document.md` for the prioritized list.

**Priority 1 (major historic significance):**
- CryptoPunks (`0xb47e3cd837dDF8e4c57F05d70Ab865de6e193BBB`)
- Curio Cards (`0x73DA73EF3a6982109c4d5BDb0dB9dd3E3783f313`)
- The DAO (`0xBB9bc244D798123fDe783fCc1C72d3Bb8C189413`)
- MoonCats (`0xc3f733ca98E0daD0386979Eb96fb1722A1A05E69`)
- Etheria (`0x5d7bB9F356E8C396e9E57bE85Ee4d3e7A44e11dC`)

**Priority 2 (related to Julian's tokens):**
- MistCoin (ERC-20 created by Fabian Vogelsteller to test the standard)
- Unicorns (`0x89205A3A3b2A69De6Dbf7f01ED13B2108B2c43e7`)
- The Unicorn Grinder (avsa's contract)

## Notes & Lessons

- **Blockscout** doesn't always return `creation_tx_hash` for very old contracts (pre-2018).
  Fall back to Etherscan website scrape or Brave search for "CONTRACT_NAME deployed block"
- **Reddit search via Brave** only finds posts that use the exact address string.
  Old discussions often used contract names, not addresses — supplement with name searches.
- **CoinGecko contract lookup** works well for ERC-20 tokens; returns nothing for NFTs/non-tokens.
- The script produces a research report, not a finished article. Always review before submitting.
- ethereumhistory.com auto-posts to erc20sales bot on FIRST documentation save only.
  Subsequent edits do not re-post. Don't make test saves.

## Database Access

The production Neon DB is accessible via Vercel env vars. Get credentials:
```bash
cd ~/workspace/repos/ethereumhistory
npx vercel link --project ethereumhistory-web --yes
npx vercel env pull .env.local --yes
# .env.local now has POSTGRES_URL, ETHERSCAN_API_KEY, all secrets
```

Run migrations directly:
```bash
cd ~/workspace/repos/ethereumhistory
npx tsx scripts/migrate.ts                          # all pending
npx tsx scripts/migrate.ts --only 019_foo.sql       # specific file
```

**Note:** The active Vercel project is `ethereumhistory-web` (NOT `ethereumhistory`).
Always link to `ethereumhistory-web` or env pull will return empty.

## Files

- Research script: `~/workspace/scripts/eth-history-research.sh`
- Contract queue: `~/workspace/tasks/eth-history-contracts-to-document.md`
- Site repo: `~/workspace/repos/ethereumhistory`
- DB credentials: `~/workspace/repos/ethereumhistory/.env.local` (pull fresh with vercel env pull)
- Media upload API: `POST /api/contract/<address>/media` (historian auth required)

## API Endpoints (updated Mar 22, 2026)

### Agent Discovery API
```
GET /api/agent/contracts?q=MyToken&unverified=1&sort=siblings&limit=50&offset=0
```
Params:
- `q` - Text search (name, token_name, symbol, address, decompiled_code)
- `unverified=1` - Only contracts with no verification_method
- `undocumented_only=1` - Only contracts with no shortDescription
- `sort=siblings` - Sort by sibling count (most shared bytecode first)
- `era_id` - Filter by era (frontier, homestead, dao, tangerine, spurious)
- `from_timestamp` / `to_timestamp` - ISO date range
- `limit` / `offset` - Pagination (max 200)

Auth: `x-api-key: eh_xxx` or `x-historian-token: xxx`
Rate limits: 120/min authenticated, 20/min anonymous

### Contract Write API
```
POST /api/contract/{address}/history/manage
```
Body: `{ "contract": { fields... }, "links": [], "deleteIds": [] }`

**All field names are camelCase. Snake_case is silently ignored.**

Key fields: `shortDescription`, `description`, `historicalContext`, `historicalSignificance`,
`verificationMethod`, `verificationProofUrl`, `verificationNotes`, `compilerCommit`,
`compilerLanguage`, `sourceCode`, `contractType`

**⚠️ verificationStatus is computed, not stored.** The API returns `"verified"` only when
`sourceCode` is present in the DB. To flip a contract to verified, you MUST include `sourceCode`
in the request. Setting `verificationMethod` alone does NOT change the status.

To write source code, you must also pass `"verificationStatus": "verified"` in the same request
(this gates the sourceCode field). Minimum verified publish payload:
```json
{
  "contract": {
    "verificationStatus": "verified",
    "verificationMethod": "exact_bytecode_match",
    "compilerLanguage": "solidity",
    "compilerCommit": "soljson vX.X.X (optimizer ON/OFF)",
    "verificationProofUrl": "https://github.com/cartoonitunes/REPO",  // ← renders as "View Verification Proof" button on the contract page — always set this to your proof repo
    "sourceCode": "contract Foo { ... }"
  },
  "links": [],
  "deleteIds": []
}
```

### Siblings API
```
GET /api/contracts/{address}/siblings?offset=0
```
Returns: hash, count, groupVerified, groupName, groupContractType, contracts[]

## Verification Methods
- `exact_bytecode_match` - Compiled source matches on-chain bytecode byte-for-byte
- `near_exact_match` - Source recovered, all logic verified, minor gap (e.g. unknown variable name)
- `author_published` - Developer published the source (gist, repo, etc.) ← NOTE: not `author_published_source`
- `etherscan_verified` - Source verified on Etherscan (not our work)
- `source_reconstructed` - Source reconstructed from bytecode but not a byte-for-byte compiler match

## Bytecode Cracking Workflow
1. Fetch runtime + creation bytecode from Etherscan/Alchemy
2. Extract function selectors via opcode walk
3. Look up selectors at api.openchain.xyz/signature-database
4. Reconstruct Solidity source
5. Compile with candidate compiler versions (solc binaries at /tmp/solc_bins/)
6. Match runtime bytecode (strip bzzr0 metadata for post-0.4.7 contracts)
7. Publish: verification repo, awesome-ethereum-proofs, EH manage API, social bot

## Sibling-First Crack Strategy
Crack contracts with most siblings first for maximum coverage.
Query: `?unverified=1&sort=siblings&limit=20`
Priority list: memory/uncracked-sibling-priority.md

## Social Bot Trigger
```python
requests.post("https://nameless-lake-39668-540f6213f30f.herokuapp.com/contractdocumentation", json={
    "contract_address": addr,
    "contract_name": name,
    "deployment_timestamp": timestamp,  # Must be pre-2017 currently
    "short_description": desc,
    "contract_url": f"https://ethereumhistory.com/contract/{addr}"
})
```
Auto-fires on first edit via manage API. Only manually trigger for contracts with prior edits.

## API Keys
Historians can generate API keys from their profile page.
Format: `eh_` + 32 hex chars. Max 5 per historian.
Keys validated via SHA-256 hash lookup in `api_keys` table.

## Native C++ Solc Binaries
Pre-built native C++ solc binaries for frontier-era contracts:
**https://github.com/cartoonitunes/solc-native-builds**
Use when soljson produces close-but-not-exact matches (typically 1-5 byte difference).
