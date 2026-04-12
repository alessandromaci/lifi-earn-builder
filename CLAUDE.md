# LI.FI Earn Builder

This repo lets you build a complete DeFi Earn product frontend using LI.FI Earn API + Composer. Run `/build` to get started.

## What This Creates

A Next.js app with:
- Vault discovery and filtering (powered by LI.FI Earn Data API)
- Wallet connection (RainbowKit + wagmi)
- One-click vault deposits (powered by LI.FI Composer)
- Portfolio position tracking

## Commands

- `/build` — Create a new Earn product (interactive setup)

## Project Layout

Generated projects live under `projects/`. Each `/build` run creates a new folder.

```
projects/
├── my-earn-app/          # Generated project 1
├── another-project/      # Generated project 2
└── ...
```

## LI.FI Earn — Quick Context

LI.FI Earn is a B2B enterprise API for DeFi yield:

- **Earn Data API** (`https://earn.li.fi`) — vault discovery, APY/TVL data, portfolio tracking
- **LI.FI Quote API** (`https://li.quest`) — deposit execution via Composer (handles swap + bridge + deposit in one tx)

Together they let you build a full "Earn" product with just 3 API calls:
1. `GET /v1/earn/vaults` — discover vaults
2. `GET /v1/quote` — get deposit quote (pass vault address as `toToken`)
3. `GET /v1/earn/portfolio/:address/positions` — track user positions

## Key Integration Details

- The vault address IS the `toToken` in the quote request
- After ERC20 approval, always re-fetch the quote (nonce becomes stale)
- Native token deposits (ETH) skip the approval step
- The approval target is `quote.estimate.approvalAddress`
- Status polling: `GET /v1/status?txHash=...&fromChain=...&toChain=...`

## Links

- [LI.FI Earn Docs](https://docs.li.fi/earn/overview)
- [LI.FI Composer Docs](https://docs.li.fi/composer/quickstart)
- [Get an API Key](https://li.fi/plans/)
