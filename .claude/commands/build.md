# Build an Earn Product with LI.FI

You are building a complete DeFi "Earn" frontend product powered by LI.FI Earn API + LI.FI Composer.

## Step 1: Ask the User for Configuration

Before writing any code, ask these questions **one at a time**, waiting for each answer:

1. **"What should we call this project?"** (used as folder name under `projects/`)
2. **"Enter your LI.FI API key"** (from https://li.fi/plans/ — used for deposit quotes via `li.quest`). Store it but do not log or display it.
3. **"Describe the visual style you want"** — e.g., "dark mode with purple accents", "clean fintech", "Morocco medina vibes", "cyberpunk neon". You will translate this into a Tailwind color palette and design system.
4. **"Which vault protocols do you want to show?"** — Options: "all" (show everything available) or list specific ones like "morpho, aave, euler". Default: all.
5. **"Which chains do you want to support?"** — Options: "all" (fetch from API) or list like "base, arbitrum, ethereum". Default: all.
6. **"Want me to set up Vercel deployment too?"** (yes/no, default: no)

## Step 2: Create the Project

Create the project folder at `projects/<project-name>/` relative to the repo root.

### 2.1 Initialize Next.js

```bash
cd projects/
npx create-next-app@latest <project-name> --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-pnpm
cd <project-name>
```

### 2.2 Install Dependencies

```bash
pnpm add @rainbow-me/rainbowkit wagmi@^2.9.0 viem@2.x @tanstack/react-query
```

### 2.3 Fix tsconfig.json

Change the `target` from `ES2017` to `ES2020` (required for BigInt literals used by viem/wagmi):

```json
"target": "ES2020"
```

### 2.4 Environment File

Create `.env.local`:
```
NEXT_PUBLIC_LIFI_API_KEY=<user's API key>
NEXT_PUBLIC_WALLET_CONNECT_PROJECT_ID=<generate or ask user — can use "demo" for testing>
```

If the user doesn't have a WalletConnect project ID, use the RainbowKit default demo ID for development.

## Step 3: Build the Application

### Project Structure

```
src/
├── app/
│   ├── layout.tsx              # Root layout with Providers wrapper
│   ├── page.tsx                # Home: vault list + filters
│   ├── portfolio/
│   │   └── page.tsx            # User positions page
│   └── globals.css             # Tailwind + custom theme vars
├── components/
│   ├── Providers.tsx           # Wagmi + RainbowKit + ReactQuery setup
│   ├── Header.tsx              # Nav + ConnectButton from RainbowKit
│   ├── VaultList.tsx           # Main vault list with loading/error states
│   ├── VaultCard.tsx           # Individual vault display (APY, TVL, protocol, chain)
│   ├── VaultFilters.tsx        # Chain + protocol + asset dropdowns
│   ├── DepositModal.tsx        # Full deposit flow: amount → quote → approve → send
│   └── PositionList.tsx        # Portfolio positions display
├── lib/
│   ├── earn.ts                 # Earn Data API client
│   ├── quote.ts                # LI.FI Quote API client (for deposits)
│   └── chains.ts               # Wagmi chain config
└── types/
    └── vault.ts                # TypeScript types for API responses
```

### 3.1 Types (`src/types/vault.ts`)

Define these types based on the Earn API response:

```typescript
export interface Vault {
  address: string
  chainId: number
  network: string
  slug: string
  name: string
  description?: string
  protocol: {
    name: string
    logoUri?: string
    url: string
  }
  underlyingTokens: {
    address: string
    symbol: string
    decimals: number
  }[]
  lpTokens: {
    address: string
    symbol: string
    decimals: number
    priceUsd?: string
  }[]
  analytics: {
    apy: { base: number | null; reward: number | null; total: number }
    apy7d: number | null
    apy30d: number
    tvl: { usd: string }
  }
  isTransactional: boolean
  isRedeemable: boolean
  depositPacks: { name: string; stepsType: 'instant' | 'complex' }[]
  redeemPacks: { name: string; stepsType: 'instant' | 'complex' }[]
  tags: string[]
}

export interface VaultListResponse {
  data: Vault[]
  nextCursor?: string
  total: number
}

export interface Chain {
  name: string
  chainId: number
  networkCaip: string
}

export interface Protocol {
  name: string
  logoUri?: string
  url: string
}

export interface Position {
  chainId: number
  protocolName: string | null
  asset: {
    address: string
    name: string
    symbol: string
    decimals: number
  }
  balanceUsd: string | null
  balanceNative: string
}
```

### 3.2 API Clients

#### `src/lib/earn.ts` — Earn Data API

```typescript
const EARN_API_URL = 'https://earn.li.fi'

const headers: Record<string, string> = {}
// Earn API key is optional for now but may be required later
const apiKey = process.env.NEXT_PUBLIC_LIFI_API_KEY
if (apiKey) headers['x-lifi-api-key'] = apiKey

export async function listVaults(params?: {
  chainId?: number
  asset?: string
  protocol?: string
  minTvlUsd?: number
  sortBy?: 'apy' | 'tvl'
  limit?: number
  cursor?: string
}): Promise<VaultListResponse> {
  const searchParams = new URLSearchParams()
  if (params?.chainId) searchParams.set('chainId', String(params.chainId))
  if (params?.asset) searchParams.set('asset', params.asset)
  if (params?.protocol) searchParams.set('protocol', params.protocol)
  if (params?.minTvlUsd) searchParams.set('minTvlUsd', String(params.minTvlUsd))
  if (params?.sortBy) searchParams.set('sortBy', params.sortBy)
  if (params?.limit) searchParams.set('limit', String(params.limit))
  if (params?.cursor) searchParams.set('cursor', params.cursor)

  const res = await fetch(`${EARN_API_URL}/v1/earn/vaults?${searchParams}`, { headers })
  if (!res.ok) throw new Error(`Earn API error: ${res.status}`)
  return res.json()
}

export async function getVault(chainId: number, address: string): Promise<Vault> {
  const res = await fetch(`${EARN_API_URL}/v1/earn/vaults/${chainId}/${address}`, { headers })
  if (!res.ok) throw new Error(`Vault not found: ${res.status}`)
  return res.json()
}

export async function listChains(): Promise<Chain[]> {
  const res = await fetch(`${EARN_API_URL}/v1/earn/chains`, { headers })
  if (!res.ok) throw new Error(`Chains API error: ${res.status}`)
  return res.json()
}

export async function listProtocols(): Promise<Protocol[]> {
  const res = await fetch(`${EARN_API_URL}/v1/earn/protocols`, { headers })
  if (!res.ok) throw new Error(`Protocols API error: ${res.status}`)
  return res.json()
}

export async function getPositions(userAddress: string): Promise<{ positions: Position[] }> {
  const res = await fetch(`${EARN_API_URL}/v1/earn/portfolio/${userAddress}/positions`, { headers })
  if (!res.ok) throw new Error(`Portfolio API error: ${res.status}`)
  return res.json()
}
```

#### `src/lib/quote.ts` — LI.FI Quote API (for deposits)

```typescript
const LIFI_API_URL = 'https://li.quest/v1'

function getHeaders(): Record<string, string> {
  const headers: Record<string, string> = {}
  const apiKey = process.env.NEXT_PUBLIC_LIFI_API_KEY
  if (apiKey) headers['x-lifi-api-key'] = apiKey
  return headers
}

export async function getDepositQuote(params: {
  fromChain: number
  toChain: number
  fromToken: string
  toToken: string       // This is the VAULT ADDRESS (LP token)
  fromAmount: string    // In smallest unit (e.g., wei for ETH, 6 decimals for USDC)
  fromAddress: string
}): Promise<any> {
  const searchParams = new URLSearchParams({
    fromChain: String(params.fromChain),
    toChain: String(params.toChain),
    fromToken: params.fromToken,
    toToken: params.toToken,
    fromAmount: params.fromAmount,
    fromAddress: params.fromAddress,
    toAddress: params.fromAddress,
  })

  const res = await fetch(`${LIFI_API_URL}/quote?${searchParams}`, {
    headers: getHeaders(),
  })
  if (!res.ok) {
    const error = await res.json().catch(() => ({}))
    throw new Error(error.message || `Quote API error: ${res.status}`)
  }
  return res.json()
}

export async function pollStatus(txHash: string, fromChain: number, toChain: number) {
  const searchParams = new URLSearchParams({
    txHash,
    fromChain: String(fromChain),
    toChain: String(toChain),
  })

  const res = await fetch(`${LIFI_API_URL}/status?${searchParams}`, {
    headers: getHeaders(),
  })
  if (!res.ok) throw new Error(`Status API error: ${res.status}`)
  return res.json()
}
```

### 3.3 Wagmi + RainbowKit Setup

#### `src/lib/chains.ts`

Configure wagmi chains based on user's preferences. Include at minimum: base, arbitrum, mainnet, optimism. Use public transports.

#### `src/components/Providers.tsx`

```typescript
'use client'

import { getDefaultConfig, RainbowKitProvider } from '@rainbow-me/rainbowkit'
import { WagmiProvider } from 'wagmi'
import { base, arbitrum, mainnet, optimism } from 'wagmi/chains'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import '@rainbow-me/rainbowkit/styles.css'

const config = getDefaultConfig({
  appName: '<project-name>',
  projectId: process.env.NEXT_PUBLIC_WALLET_CONNECT_PROJECT_ID || 'demo',
  chains: [base, arbitrum, mainnet, optimism],
})

const queryClient = new QueryClient()

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <RainbowKitProvider>
          {children}
        </RainbowKitProvider>
      </QueryClientProvider>
    </WagmiProvider>
  )
}
```

### 3.4 Key Components

#### Header (`src/components/Header.tsx`)

- App name/logo on the left
- Navigation: "Earn" | "Portfolio"
- `<ConnectButton />` from RainbowKit on the right

#### VaultList (`src/components/VaultList.tsx`)

- Fetch vaults using `listVaults()` with TanStack Query (`useQuery`)
- Show loading skeleton while fetching
- Display each vault using `VaultCard`
- Include `VaultFilters` component above the list
- Support pagination via cursor

#### VaultCard (`src/components/VaultCard.tsx`)

Each card shows:
- Vault name + protocol name and logo
- Chain badge
- Underlying token symbol(s)
- **APY** prominently displayed (format: `vault.analytics.apy.total.toFixed(2)%` — the API already returns percentages, do NOT multiply by 100)
- **TVL** (format: `$${Number(vault.analytics.tvl.usd).toLocaleString()}`)
- Tags (e.g., "Lending", "Staking")
- "Deposit" button (disabled if `!vault.isTransactional`, shows "View Only" instead)
- Clicking Deposit opens DepositModal

#### VaultFilters (`src/components/VaultFilters.tsx`)

- Chain dropdown (populated from `listChains()`)
- Protocol dropdown (populated from `listProtocols()`)
- Sort by: APY / TVL toggle
- Min TVL slider or input (default: $100,000)

#### DepositModal (`src/components/DepositModal.tsx`)

This is the critical component. The deposit flow:

1. **Select source token**: Show a dropdown with common tokens on the current chain. For the demo, default to native ETH or USDC on the vault's chain. Include the native token (address: `0x0000000000000000000000000000000000000000`) and USDC.

2. **Enter amount**: Input field with balance display. Use wagmi's `useBalance` hook.

3. **Get quote**: Call `getDepositQuote()` with:
   - `fromChain`: current connected chain ID
   - `toChain`: vault's chain ID
   - `fromToken`: selected source token address
   - `toToken`: **vault's address** (this is the key — the vault address IS the toToken)
   - `fromAmount`: amount in smallest unit
   - `fromAddress`: connected wallet address

4. **Show quote details**: Display estimated output, fees, route steps.

5. **Approve (if needed)**: If `fromToken` is NOT the native token (not `0x000...000`):
   - Check allowance using wagmi's contract read
   - If insufficient, send approval transaction to `quote.estimate.approvalAddress`
   - Wait for confirmation
   - **CRITICAL: Re-fetch the quote after approval** (the original quote's nonce becomes stale)

6. **Execute deposit**: Send the transaction using wagmi's `useSendTransaction`:
   ```typescript
   sendTransaction({
     to: quote.transactionRequest.to,
     data: quote.transactionRequest.data,
     value: BigInt(quote.transactionRequest.value || '0'),
     gas: quote.transactionRequest.gasLimit ? BigInt(quote.transactionRequest.gasLimit) : undefined,
   })
   ```

7. **Poll status**: After tx confirms, poll `pollStatus()` until DONE or FAILED.

8. **Success**: Show success message with link to portfolio page.

**Important implementation notes:**
- Use wagmi hooks: `useAccount`, `useBalance`, `useSendTransaction`, `useWaitForTransactionReceipt`, `useReadContract`, `useWriteContract`
- Handle chain switching: if user is on a different chain than the vault, prompt to switch using wagmi's `useSwitchChain`
- For native token deposits (ETH), skip the approval step entirely
- The approval target is `quote.estimate.approvalAddress`, NOT the vault address
- After approval confirms, ALWAYS re-fetch the quote before sending the deposit tx

#### PositionList (`src/components/PositionList.tsx`)

- Fetch positions using `getPositions(address)` when wallet is connected
- Display each position: token symbol, protocol, chain, USD balance
- Show "Connect wallet to view positions" when disconnected
- Auto-refresh after a deposit succeeds

### 3.5 Styling

Translate the user's style description into a cohesive design:

- Configure `tailwind.config.ts` with a custom color palette that matches the requested style
- Use CSS variables in `globals.css` for theming
- Cards should have subtle borders, hover effects, and smooth transitions
- The vault list should feel like a professional financial dashboard
- APY numbers should be highlighted (green for high APY)
- Use proper number formatting (commas for thousands, 2 decimal places for APY)
- Responsive layout but optimize for desktop (this may be screen-recorded)
- Add a subtle "Powered by LI.FI" footer with the LI.FI logo or text

### 3.6 Protocol/Chain Filtering (from user preferences)

If the user specified specific protocols or chains in Step 1:
- Pre-filter the vault list to only show those protocols/chains
- Still allow the user to change filters in the UI
- Set the defaults in the filter dropdowns

## Step 4: Start the Dev Server

After all files are created:

```bash
cd projects/<project-name>
pnpm dev
```

Tell the user: "Your Earn product is running at http://localhost:3000. Connect your wallet and try a deposit!"

## Step 5: Optional — Vercel Deployment

If the user said yes to Vercel:
```bash
pnpm add -g vercel
vercel --yes
```

## Reference: Common Token Addresses

```
# Native (all chains)
ETH = 0x0000000000000000000000000000000000000000

# USDC
USDC (Base)     = 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
USDC (Ethereum) = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
USDC (Arbitrum) = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831

# Example vault addresses (for testing)
Morpho USDC (Base) = 0x7BfA7C4f149E7415b73bdeDfe609237e29CBF34A
```

## Reference: API URLs

| Service | URL | Auth |
|---------|-----|------|
| Earn Data API | `https://earn.li.fi` | `x-lifi-api-key` header (optional for now) |
| LI.FI Quote API | `https://li.quest` | `x-lifi-api-key` header |
| Status Polling | `https://li.quest/v1/status` | `x-lifi-api-key` header |

## Quality Checklist

Before reporting done, verify:
- [ ] `pnpm dev` starts without errors
- [ ] Vault list loads and displays real data from the Earn API
- [ ] Chain and protocol filters work
- [ ] Wallet connects via RainbowKit
- [ ] Deposit modal opens when clicking a vault
- [ ] Quote fetches successfully (test with a small amount)
- [ ] The UI matches the user's requested style
- [ ] No TypeScript errors (`pnpm build` succeeds)
- [ ] "Powered by LI.FI" footer is visible
