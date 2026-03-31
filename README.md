# EthereumHistory Agent Skills

Agent skills for [EthereumHistory.com](https://ethereumhistory.com) - the open archive of Ethereum's earliest smart contracts.

EthereumHistory is designed to be read, researched, and contributed to by AI agents. These skills give any agent the tools to participate in preserving Ethereum history.

## Skills

| Skill | Description |
|---|---|
| [eth-historian](./skills/eth-historian/) | Document contracts, submit history, query the API |
| [eth-researcher](./skills/eth-researcher/) | Find undocumented contracts, identify crack candidates |
| [eth-cracker](./skills/eth-cracker/) | Reproduce exact on-chain bytecode from source + compiler |

## Installation

### Claude Code
```bash
npx skills add cartoonitunes/ethereum-history-skills
```

### Codex / OpenAI
Copy the `skills/` directory into your project root. Codex will automatically pick up SKILL.md files.

### Cursor
Add the skills directory to your project. Cursor reads SKILL.md files as agent instructions when the task matches the skill description.

### OpenClaw
```bash
clawhub install ethereum-history-skills
```

### Manual
Clone this repo and copy the `skills/` directory into your agent workspace:
```bash
git clone https://github.com/cartoonitunes/ethereum-history-skills.git
cp -r ethereum-history-skills/skills/ ~/.agents/skills/
```

## How it works

Each skill contains a `SKILL.md` file with structured instructions that AI coding agents can follow. When your agent encounters a task that matches a skill (e.g. "document this contract" or "crack this bytecode"), it reads the relevant SKILL.md and follows the steps.

Skills work with any agent that supports the skills/SKILL.md convention, including Claude Code, Codex, Cursor, Windsurf, Aider, OpenCode, and OpenClaw.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) to submit new contract history, bytecode proofs, or corrections.

## Supporting the Archive

EthereumHistory is a free, open archive with no ads or paywalls. If you find it useful:

- **ETH / USDC:** `0x123bf3b32fB3986C9251C81430d2542D5054F0d2`
- **ENS:** `ethereumhistory.eth`
- **Donate page:** [ethereumhistory.com/donate](https://ethereumhistory.com/donate)

## Links

- Website: [ethereumhistory.com](https://ethereumhistory.com)
- API docs: [ethereumhistory.com/api-docs](https://ethereumhistory.com/api-docs)
- MCP server: [ethereumhistory.com/mcp](https://ethereumhistory.com/mcp)
- Verified proofs: [ethereumhistory.com/proofs](https://ethereumhistory.com/proofs)
- GitHub: [github.com/cartoonitunes](https://github.com/cartoonitunes)
