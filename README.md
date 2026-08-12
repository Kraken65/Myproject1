# Velox — Fast, Low-Fee Solana Trading Bot (Telegram)

**Velox** (Latin for *swift*) is a fast, **low-fee (0.5%)**, custodial Solana trading bot for
Telegram. It brings together the core features of the leading Solana trading bots (Trojan,
BONKbot, Photon, BullX, Maestro, Banana Gun) — buy/sell by contract address, positions & PnL,
slippage and priority-fee controls, referrals — while charging **half** the ~1% fee the
incumbents take, and with a speed-first execution path.

Bot: [@VeloxSolBot](https://t.me/VeloxSolBot)

> ⚠️ **Read this first.** This is an MVP for you to build on, not a finished, audited product.
> It is **custodial**: the bot generates and holds users' private keys (encrypted). If you run
> it with real funds you are responsible for other people's money. **Get a professional
> security audit before going live.** No codebase can guarantee becoming "#1" — that depends on
> marketing, liquidity, trust, and execution. This gives you a solid, competitive technical
> foundation; the rest is up to you.

---

## What it does

- **/start** — onboards a user and auto-creates an encrypted Solana wallet.
- **Buy** — paste a token mint → pick a SOL amount (presets or custom) → confirm → swap via
  Jupiter. A **0.5% fee** (configurable) is collected in SOL to your fee wallet in the same tx.
- **Sell** — pick a token from Positions → sell a % (presets or custom) → confirm.
- **Positions** — live token holdings valued in SOL.
- **Settings** — slippage, priority-fee amount, priority mode (auto / fixed / jito), confirm on/off.
- **Referrals** — every user gets a link; referred users pay a reduced fee, referrers accrue earnings.
- **Wallet** — view address, deposit, and export private key (behind an explicit warning).

## Why it's competitive

**Cheaper.** Fee is `FEE_BPS` (default `50` = 0.5%), collected as a SOL transfer to `FEE_WALLET`
inside the swap transaction — works uniformly for buys and sells with a single fee wallet.

**Faster (the speed stack):**
- Configurable **fast/staked RPC** via `RPC_URL` (use Helius/Triton/QuickNode in prod).
- **Dynamic priority fees** estimated from recent on-chain prioritization fees (75th percentile).
- **Jito tip** mode for landing in bundles during congestion.
- Fresh quote at execution time; compact **v0 transactions** with address lookup tables.
- Short-TTL token-metadata caching; non-blocking "executing…" UX so the UI never stalls.

**Scalable.** Handlers are stateless; all state lives in the DB and session store. Dev runs on
long-polling + SQLite + in-memory sessions (zero infra). Prod switches to **webhook + Postgres +
Redis** by changing env only — the same code scales horizontally behind a load balancer.

---

## Quick start

Requires **Node 18+** (tested on Node 24).

```bash
npm install
cp .env.example .env
```

Edit `.env`:

1. `BOT_TOKEN` — from [@BotFather](https://t.me/BotFather).
2. `MASTER_KEY` — generate one:
   ```bash
   node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
   ```
3. `FEE_WALLET` — the public key that receives your 0.5% fee.
4. `RPC_URL` — a fast RPC endpoint (the public one works for testing but is slow/rate-limited).

Set up the database and run:

```bash
npm run prisma:migrate   # creates the SQLite dev DB
npm run dev              # starts the bot on long-polling
```

Message your bot on Telegram and send `/start`.

## Verify without spending

```bash
npm run selftest:crypto   # AES-256-GCM encrypt/decrypt + keypair round-trip
npm run test:quote        # live Jupiter routing (read-only, no funds moved)
npm run smoke             # user/wallet/referral provisioning against your RPC
```

To test a **real** buy: fund your bot wallet with a small amount of SOL and buy a tiny amount
of a liquid token (e.g. via its mint). Start small.

---

## Architecture

```
src/
  index.ts               entrypoint (polling dev / webhook prod)
  config.ts              zod-validated env
  bot/
    bot.ts               grammY instance + middleware wiring
    session.ts           per-user session types
    keyboards.ts         inline keyboards
    handlers/            start, wallet, menu, buy, sell, confirm,
                         positions, settings, referral, help, text router
  services/
    crypto.ts            AES-256-GCM
    wallet.service.ts    keygen, encrypt/decrypt, withKeypair (in-memory sign)
    jupiter.service.ts   quote + swap-instructions
    trade.service.ts     assemble tx: swap ix + SOL fee + priority fee, sign, send, confirm
    price.service.ts     token metadata, holdings, SOL valuation
    referral.service.ts  codes, attribution, fee discount
    user.service.ts      provisioning (user + wallet + settings)
    solana.ts            RPC connection, priority-fee estimate, confirmation
    pending.service.ts   short-TTL store for trades awaiting confirmation
  db/prisma.ts           Prisma client singleton
  utils/                 logger (redacts secrets), validate, format
prisma/schema.prisma     User, Wallet, Trade, Referral, Settings
```

### How the fee works

Jupiter's native `platformFeeBps` takes the fee on the **output** mint and needs a
pre-existing fee token account **per mint** — impractical for buying arbitrary tokens. So we
fetch the swap as **instructions** and append our own **SOL transfer** to `FEE_WALLET`:
- **Buy:** fee = `FEE_BPS` of SOL spent.
- **Sell:** fee = `FEE_BPS` of SOL received.

One fee wallet, one currency (SOL), single transaction.

## Security notes

- Private keys are encrypted with **AES-256-GCM** (per-record nonce + auth tag). Plaintext keys
  exist only in memory during signing and are zeroed immediately after.
- `MASTER_KEY` is the root secret. If it leaks, all custodial keys are compromised; if it's lost,
  all wallets are unrecoverable. Store it in a secret manager in prod, **not** a file.
- The logger redacts secret-ish fields; never log decrypted keys.
- Input is validated (mint format, amounts); trades require explicit confirmation by default.
- `.env` and `*.db` are gitignored — never commit the database (it holds encrypted keys).

## Production checklist

- [ ] **Security audit** of key handling and the trade path.
- [ ] Postgres (`DATABASE_URL`) + `prisma migrate deploy`.
- [ ] Redis session storage + webhook mode (`BOT_MODE=webhook`, `WEBHOOK_URL`).
- [ ] A real staked RPC and (optionally) a paid Jupiter API key (`JUPITER_BASE_URL=https://api.jup.ag`).
- [ ] Proper Jito bundle submission (this MVP adds a tip transfer; full bundle relay is a next step).
- [ ] Fee-wallet key custody, monitoring/alerting, and legal/regulatory review for your jurisdiction.

## Roadmap (deferred from MVP)

Limit orders · copy trading · sniping/auto-buy on launch · DCA · multi-wallet · full Jito
bundles · EVM chains (Base/BSC/ETH) · web dashboard.

## License

MIT. Provided as-is, with no warranty. Trading crypto is risky; you are responsible for how you
deploy and operate this software.
