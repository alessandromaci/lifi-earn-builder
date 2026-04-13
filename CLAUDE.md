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

---

## Development Skills

These skills apply to ALL work in this repo — during `/build`, when customizing, adding features, or fixing bugs.

### Frontend Development

**Stack:** Next.js 14+ (App Router) · TypeScript · Tailwind CSS · RainbowKit · wagmi v2 · viem · TanStack React Query

**Patterns to follow:**
- All components that use hooks (`useState`, `useEffect`, wagmi hooks) must have `'use client'` at the top
- Use TanStack Query (`useQuery`) for data fetching — never raw `useEffect` + `fetch` for component-level API calls
- Debounce user inputs that trigger API calls (600ms for quote fetching, 300ms for filters)
- Loading states: use skeleton placeholders, never empty screens
- Error states: show inline error messages near the action that failed, not toast notifications
- Keep components focused — one file per component, under 300 lines. If it grows beyond that, extract sub-components
- Number formatting: use `toLocaleString()` for TVL, `toFixed(2)` for APY. The Earn API returns APY already as percentage — do NOT multiply by 100

**Tailwind conventions:**
- Define the color palette in `globals.css` using `@theme` inline or in `tailwind.config.ts` under `extend.colors`
- Use semantic color names that map to the user's chosen theme (e.g., `sand-*`, `accent-*`), not raw hex
- Always include hover, focus, and disabled states on interactive elements
- Responsive: optimize for desktop but ensure mobile doesn't break (`sm:`, `md:`, `lg:` breakpoints)

**Dependencies — version constraints:**
- `wagmi@^2.9.0` — do NOT install wagmi 3.x (RainbowKit 2.x requires wagmi ^2)
- `viem@2.x` — matches wagmi v2
- tsconfig.json `target` must be `ES2020` or higher (required for BigInt literals like `0n`)

### Web3 Integration

**Wallet interaction rules:**
- ALWAYS use async hook variants (`sendTransactionAsync`, `writeContractAsync`) — the non-async versions (`sendTransaction`, `writeContract`) don't propagate wallet rejections to try/catch, leaving the UI stuck in a loading state
- Catch `UserRejectedRequestError` explicitly and reset UI to idle so the user can retry
- Use `err.shortMessage || err.message` for user-facing errors — wagmi wraps RPC errors with verbose details, `shortMessage` is cleaner
- Handle chain switching before any transaction: check `connectedChainId !== vault.chainId` and prompt `useSwitchChain`

**ERC20 approval flow:**
1. Check allowance via `useReadContract` with `erc20Abi`
2. If insufficient, call `writeContractAsync` for `approve` to `quote.estimate.approvalAddress`
3. Wait for approval tx confirmation via `useWaitForTransactionReceipt`
4. **CRITICAL: Re-fetch the quote after approval** — the original quote's nonce is stale
5. Then send the deposit tx with the fresh quote

**Native token (ETH) deposits:** Skip the approval step entirely. The value is passed via `value` field in the transaction.

**Quote API integration:**
- The vault address IS the `toToken` parameter
- `fromAmount` must be in smallest unit (wei for ETH, 6 decimals for USDC) — use `parseUnits()`
- Always set `toAddress` equal to `fromAddress` (user deposits for themselves)
- Transaction execution: use all fields from `quote.transactionRequest` — `to`, `data`, `value`, `gasLimit`

### Security

**Transaction safety:**
- Never hardcode private keys, mnemonics, or signing logic — all signing goes through the user's wallet via wagmi hooks
- Validate that `quote.transactionRequest.to` is a reasonable address before sending — don't blindly forward API responses to `sendTransaction`
- Set `gas` from the quote's `gasLimit` when available — don't leave it to the wallet's estimate for complex Composer txs
- Display what the user is signing: show token amounts, destination vault, estimated output BEFORE the wallet popup

**API key handling:**
- Store API keys in `.env.local` only, never commit them
- `.gitignore` must include `.env*` and `!.env.example`
- Access keys via `process.env.NEXT_PUBLIC_LIFI_API_KEY` — the `NEXT_PUBLIC_` prefix exposes it client-side (required for browser API calls)
- Provide a `.env.example` with placeholder values so users know what to set

**Input validation:**
- Validate amounts are positive numbers before `parseUnits()` — malformed strings will throw
- Check that `balance >= amount` before enabling the deposit button
- Sanitize token addresses: ensure they match `0x${string}` format and are 42 characters

**Dependency hygiene:**
- Pin major versions in `package.json` (`wagmi@^2.9.0`, not `wagmi@latest`)
- Run `pnpm build` after changes to catch type errors — TypeScript strict mode is on
- Never install packages with `--ignore-scripts` disabled unless you trust the package

**Common web3 pitfalls to avoid:**
- Don't store sensitive data in React state that persists across renders (clear quotes/tx data on modal close)
- Don't re-use stale quotes — they contain nonces and timestamps that expire
- Don't catch and swallow errors silently — always surface them to the user
- Don't assume `chainId` from the wallet matches the vault's chain — always verify
