# Reflect Money — Agent Skills

Agent skills for AI coding assistants integrating with the Reflect Money protocol.

## Skills

### [integrate-reflect](./skills/integrate-reflect/SKILL.md)

Complete integration reference for the Reflect Money protocol — yield-bearing stablecoins, whitelabel branded tokens, restaking, and oracle price feeds on Solana.

Covers:
- **Stablecoin SDK** (`@reflectmoney/stable.ts`) — mint and redeem USDC+, with full VersionedTransaction and Address Lookup Table patterns
- **Whitelabel SDK** (`@reflectmoney/proxy.ts`) — issue branded yield-bearing stablecoins backed by Reflect collateral
- **Restaking SDK** (`@reflectmoney/restaking`) — contribute to the Reflect Liquidity Pool (post-beta)
- **Oracle SDK** (`@reflectmoney/oracle.ts`) — read and update Doppler price feeds
- **REST API** — all endpoints across stablecoin, integration, analytics, stats, and events routes
- Error handling, production hardening, and integration patterns

## Installation

### Claude Code

```bash
npx skills add palindrome-eng/agent-skills
```

Or install directly inside Claude Code:

```
/plugin marketplace add palindrome-eng/agent-skills
/plugin install integrate-reflect@palindrome-eng-skills
```

### Manual (Cursor, Windsurf, any agent)

```bash
git clone https://github.com/palindrome-eng/agent-skills.git
cp -r agent-skills/skills/* ~/.claude/skills/
```

### Direct reference

Paste the raw URL into any agent's system prompt or context:

```
https://raw.githubusercontent.com/palindrome-eng/agent-skills/main/skills/integrate-reflect/SKILL.md
```

## Usage

Once installed, the skill activates automatically when your agent detects relevant context — `usdc+`, `stable.ts`, `yield stablecoin`, `whitelabel stablecoin`, `reflect money`, and [other triggers](./skills/integrate-reflect/SKILL.md#use--do-not-use).

You can also invoke it explicitly:

```
Use the integrate-reflect skill to help me mint USDC+ in my dApp.
```

## Repository layout

```
skills/
└── integrate-reflect/
    └── SKILL.md
```

## Links

- [Reflect Money](https://reflect.money)
- [Documentation](https://docs.reflect.money)
- [GitHub](https://github.com/palindrome-eng)

## License

MIT
