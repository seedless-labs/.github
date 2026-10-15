<p align="center">
  <img src="assets/banner-2026-10.png" alt="seedless" width="100%">
</p>

# seedless labs

**seedless is a money app for people who get paid in dollars.** hold dollars and stocks, send money to anyone, and pay any nigerian bank account straight from the app. it runs on solana under the hood. there's no seed phrase, and nobody needs to hold SOL for network fees.

this page is the technical reference for how seedless works. for the product itself, see [seedlesslabs.xyz](https://seedlesslabs.xyz).

| | |
|---|---|
| **status** | live since 24 september 2026 |
| **network** | solana mainnet |
| **platforms** | google play and the solana dApp store, iOS on the app store |
| **app** | react native (expo), `com.seedless.wallet` |
| **company** | Seedless Labs, Inc. (delaware) · Seedless Labs Limited (nigeria) |

---

## what seedless does

### pay a bank account
- **straight to a nigerian bank.** pick a bank, enter an account number, and the account holder's name is returned *before* you confirm. the amount is entered in naira. the app sends the exact USDC the order requires, and a licensed local partner settles the naira.
- **pay from whatever you hold.** if the account is short of USDC for a payout, seedless sells just enough of another holding to cover it and shows which asset it used. an account that already holds enough USDC never touches this path.
- **region-aware.** bank payouts are live in nigeria. everywhere else, the payout screen explains that it isn't available there yet, before anyone fills in bank details.

### hold and grow
- **stocks.** tokenized equities (xStocks) held next to your dollars: tesla, apple, nvidia, alphabet, microsoft, amazon, meta, coinbase, robinhood, circle, the S&P 500, the nasdaq 100 and gold. each position has **sell and send to bank**.
- **earn.** put USDC into jupiter lend and take it out at any time. the rate is variable. the balance you see comes from the chain, and the live ticker is a projection shown separately.

### send
- **to any solana address**, with a saved address book and solana pay payment requests.
- **privately**, through umbra: a private balance and private sends.
- **to someone who doesn't have seedless.** a claim link carries a one-time key in the URL fragment, which browsers never send to a server. the recipient opens it and the money lands. unclaimed links can be taken back by the sender.

### swap
- any supported pair through jupiter, and $SEED through bags.

### the account itself
- **passkey account** (lazorkit smart account). the authority is a P-256 passkey on the device, confirmed with Face ID or fingerprint.
- **gasless.** network fees are sponsored through the kora paymaster.
- **session keys.** short-lived keys authorized for a fixed slot window, so repeated everyday actions don't prompt for biometrics every time.
- **multiple accounts**, **burner addresses** (isolated keypairs with no on-chain link to the main account), and **stealth addresses**.
- **curated token list.** holdings are joined against jupiter's verified list, so airdropped look-alike tokens don't appear as real balances.
- transaction history and an optional biometric lock when the app returns from the background.

---

## architecture

```mermaid
flowchart LR
  subgraph Phone["seedless app (react native)"]
    UI[screens] --> Core[account core]
    Core --> PK[passkey + session keys]
    Core --> DK[device key, ed25519]
  end

  PK -->|sign| LZ[lazorkit smart account]
  LZ -->|sponsored fees| KO[kora paymaster]
  LZ --> SOL[(solana mainnet)]

  Core -->|quotes + instructions| JUP[jupiter swap + lend]
  Core -->|private balance| UMB[umbra]
  Core -->|RPC| RPC[alchemy]
  RPC --> SOL

  DK -->|signed requests| API[seedless-api<br/>cloudflare workers + D1]
  API -->|naira settlement| BANK[licensed payout partner → nigerian banks]
  Core -->|exact USDC to deposit address| SOL
```

| layer | built on |
|---|---|
| accounts and signing | lazorkit passkey smart accounts (P-256), session keys |
| fees | kora paymaster |
| swaps and yield | jupiter swap API, jupiter lend |
| stocks | xStocks (SPL Token-2022) |
| privacy | umbra sdk |
| RPC | alchemy |
| backend | cloudflare workers + D1 (`seedless-api`) |
| payouts | licensed nigerian payout partner, called only from the backend |

**how a bank payout moves.** the app never holds a partner credential. it asks `seedless-api` for an order, signed with a device key: a stable ed25519 key, separate from the passkey and from rotating session keys. the API returns a deposit address and an exact USDC amount. the app sends that exact amount on-chain from the user's own account, and the partner settles naira to the bank. the payout partner API key lives only in the worker.

---

## fees

one rate, charged on what the user asked for, never on the steps taken to deliver it.

| action | fee |
|---|---|
| pay a nigerian bank account | 0.5% |
| swap one token for another | 0.5% |
| hold, receive, send to any address | free |
| selling a holding to cover a payout | free (already covered by the payout fee) |
| network fees | sponsored |

the fee is shown on screen before signing. swap fees are collected on-chain through jupiter's platform fee, and only on pairs where USDC is on one side.

---

## engineering notes

a few problems worth writing down:

- **Token-2022 balances.** deriving a classic associated token account for an xStock points at an address that can never exist, and the balance silently reads as zero. holdings are read by token program instead, in one call for the whole portfolio.
- **stock splits.** xStocks carry a scaled-amount multiplier. balances use the parsed amount, never `raw / 10^decimals`, so a future split can't misreport a holding.
- **impostor mints.** a search for `TSLAx` also returns an unrelated pump.fun token with the same symbol. every stock mint is pinned; nothing is resolved by symbol.
- **exact payouts from any token.** a bank order needs an exact USDC figure, and jupiter's exact-output mode has no route for stock pools. a solver seeds from a reverse quote and verifies with exact-input quotes. it works on any solana token and converges in three to four quotes.
- **transaction size.** swaps use direct routes, so the swap plus the passkey authorization stays under solana's 1,232-byte limit. larger payloads use lazorkit's authorize-then-execute flow.
- **fee headroom.** a platform fee comes off the swap output, the same headroom slippage uses. selling to cover a payout carries no fee, so a payment can’t land a cent short.
- **sponsored-send limits.** kora can't enforce spend limits per account, so rate limits on sponsored sends live in the app.

the app has 704 automated tests.

---

## security

- keys never leave the device. the account authority is a passkey. session, burner and device keys live in the platform secure store.
- the payout partner's credentials live only in the backend.
- payout requests are signed by a device key and verified server-side.
- stock mints are pinned, and token lists are curated.

**seedless orchestrates audited infrastructure. seedless itself has not been audited.**

responsible disclosure: [seedlesslabs.xyz/security](https://seedlesslabs.xyz/security)

---

## hackathons

### won

| hackathon | result | prize | date |
|---|---|---|---|
| bags hackathon | winner (#5) | $26,000 | may 2026 |
| colosseum frontier: umbra side track | 2nd place | 3,000 USDC | may 2026 |
| colosseum frontier: encrypt / ika side track | award | 1,000 USDC | june 2026 |
| colosseum frontier: 100xdevs track | 2nd place | 2,000 USDG | july 2026 |

every prize was announced publicly by the sponsor and paid on-chain.

### entered

| hackathon | organiser | focus |
|---|---|---|
| stocklana | solana foundation | stocks held and sold straight to a bank |
| world's fair | colosseum | seedless |
| monad metropolis | monad | consumer products and payments |
| clock in | solana mobile | mobile |

---

## repositories

the app and backend are private. public here:

| repository | what it is |
|---|---|
| [`passkey-dwallet-controller`](https://github.com/seedless-labs/passkey-dwallet-controller) | research: a pinocchio program that lets a WebAuthn passkey approve ika dwallet messages. devnet, not part of the launch app. |
| [`seedless-labs-landing`](https://github.com/seedless-labs/seedless-labs-landing) | the website at seedlesslabs.xyz |

---

<p>
  <img src="assets/logo.png" alt="seedless logo" width="40" align="left">
  <a href="https://seedlesslabs.xyz">seedlesslabs.xyz</a> · <a href="https://x.com/seedless_wallet">@seedless_wallet</a> · founder <a href="https://x.com/francis_codex">@francis_codex</a> · community <a href="https://t.me/seedless_users">t.me/seedless_users</a>
</p>
