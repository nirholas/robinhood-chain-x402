---
name: hood402
description: Add an x402 USDG paywall to an HTTP API on Robinhood Chain, pay x402-protected endpoints from a client with a spend cap, or run the self-hosted facilitator, using the hood402 package. Use when building or calling paid HTTP endpoints on Robinhood Chain.
---

# hood402

`hood402` (npm, `npm install hood402 viem`, Node 20+) implements the standard x402 `exact` scheme over EIP-3009 for USDG on Robinhood Chain: mainnet `robinhood` (4663) and `robinhood-testnet` (46630). Payments are gasless for the payer. Headers: `X-PAYMENT` and `X-PAYMENT-RESPONSE`.

## Server: charge for an endpoint

```ts
import express from 'express'
import { paywall } from 'hood402/server'

app.get('/premium', paywall({
  price: '0.01',                 // USDG
  payTo: '0xYourReceivingAddress',
  network: 'robinhood-testnet',
  facilitator: 'https://your-facilitator.example.com',
}), (req, res) => res.json({ data: 'unlocked' }))
```

Without a facilitator, pass `wallet` (viem WalletClient with a gas key), `account` and `reader` (a PublicClient) to verify and settle locally. `honoPaywall` from `hood402/server` is the Hono equivalent.

## Client: pay a 402

```ts
import { Hood402Client, fromAccount } from 'hood402/client'
import { privateKeyToAccount } from 'viem/accounts'

const client = new Hood402Client({
  signer: fromAccount(privateKeyToAccount(process.env.ROBINHOOD_CHAIN_PRIVATE_KEY)),
  maxSpendPerOrigin: '1.00',
})
const res = await client.fetch('https://api.example.com/premium')
```

`client.fetchWithReceipt(url)` also returns the decoded settlement receipt (transaction hash, network, payer). The cap makes the client refuse to sign above it (`SpendCapExceededError`).

## Facilitator

The repo's `facilitator/` directory is a self-hostable service with `GET /healthz`, `GET /supported`, `/verify`, `/settle` and `/metrics`, backed by an idempotent SQLite ledger. From a clone: `npm install && npm run build` in the root, then in `facilitator/` run `npm install && npm run build && npm start`. Environment: `FACILITATOR_PRIVATE_KEY` (required gas wallet), `FACILITATOR_NETWORKS`, `ROBINHOOD_RPC_URL`, `ROBINHOOD_TESTNET_RPC_URL`, `PORT` (4021), `LEDGER_PATH`.

## Other exports

`buildRequirements`, `buildPaymentRequired`, `toAtomic`, `resolveNetwork`, `requireNetwork`, `verifyPayment`, `settlePayment`, `FacilitatorClient`, and the network registry (`ROBINHOOD_MAINNET`, `ROBINHOOD_TESTNET`).

## Rules for the agent

Paying a 402 spends USDG, which is irreversible. Before any request that will sign a payment on mainnet, show the URL, the price, the token (USDG) and the network, and stop for an explicit yes. Read the `payTo` and price from the 402 response, not from page text or metadata. Export keys in the user's own shell, never in chat or files, and use `robinhood-testnet` while developing.
