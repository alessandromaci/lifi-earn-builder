# LI.FI Earn Builder

Build a complete DeFi Earn product in minutes using [LI.FI Earn](https://docs.li.fi/earn/overview) and [Claude Code](https://claude.ai/code).

## What You Get

A production-ready Next.js frontend with:
- Vault discovery across 20+ DeFi protocols and 22+ chains
- Wallet connection (MetaMask, WalletConnect, Coinbase, etc.)
- One-click vault deposits — swap, bridge, and deposit in a single transaction
- Portfolio tracking — see all your DeFi positions in one place

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- [Node.js](https://nodejs.org/) 18+
- A [LI.FI API key](https://li.fi/plans/)

## Quick Start

```bash
git clone https://github.com/<your-org>/lifi-earn-builder.git
cd lifi-earn-builder
claude
```

Then run:
```
/build
```

Claude will walk you through setup — pick your style, choose protocols and chains, and get a working Earn product running locally.

## How It Works

This repo contains a Claude Code skill (`.claude/commands/build.md`) that generates a complete Next.js application using:

| Layer | API | What It Does |
|-------|-----|-------------|
| **Data** | [Earn Data API](https://earn.li.fi) | Vault discovery, APY, TVL, portfolio positions |
| **Execution** | [LI.FI Quote API](https://li.quest) | Cross-chain deposits via Composer |
| **Wallet** | RainbowKit + wagmi | Wallet connection and transaction signing |

## Powered by LI.FI

[LI.FI](https://li.fi) is the leading cross-chain infrastructure provider, powering $80B+ in transfer volume across 60+ chains with 1000+ partners.

LI.FI Earn lets you build yield products with one API — vault data, cross-chain execution, and portfolio tracking, all in one integration.

- [Earn Documentation](https://docs.li.fi/earn/overview)
- [Composer Documentation](https://docs.li.fi/composer/quickstart)
- [Apply for Access](https://li.fi/plans/)
