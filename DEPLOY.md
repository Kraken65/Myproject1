# Velox — 24/7 Deploy Guide 📦

Your bot currently runs **only while your PC is on**. To make it always-on, you deploy it to
a server. This guide covers two paths:

- **Path A — Railway** (easiest, ~$5/mo, recommended for your first deploy)
- **Path B — VPS** (Hetzner/DigitalOcean, more control, ~$5/mo)

Both keep the bot alive 24/7 and auto-restart it if it crashes.

---

## Before you deploy (do this once)

### 1. Push the code to GitHub (private repo)
Railway deploys from GitHub. Your `.env` is gitignored, so your secrets stay OUT of the repo — good.

```bash
cd C:/Users/bnc/solana-trade-bot
git init
git add .
git commit -m "Velox bot"
```

Then create a **PRIVATE** repo on github.com and follow its "push existing repo" instructions.
⚠️ Make sure the repo is **Private** — never public with this code.

### 2. Confirm secrets are NOT committed
```bash
git status --ignored
```
You should see `.env` and `prisma/dev.db` under "Ignored files." If `.env` shows as tracked,
STOP and tell me — we must not push it.

---

## Path A — Railway (recommended)

Railway runs your bot in webhook mode with a public URL and a real Postgres database.

### Step 1 — Sign up
Go to railway.app → sign in with GitHub.

### Step 2 — New project from your repo
- New Project → Deploy from GitHub repo → pick your Velox repo.
- Railway auto-detects Node and runs `npm install` + `npm run build`.

### Step 3 — Add a Postgres database
- In the project → New → Database → PostgreSQL.
- Railway creates a `DATABASE_URL` variable automatically.

### Step 4 — Switch Prisma to Postgres
Edit `prisma/schema.prisma`, change the datasource:
```prisma
datasource db {
  provider = "postgresql"   // was "sqlite"
  url      = env("DATABASE_URL")
}
```
Commit + push. (Tell me and I'll do this edit + regenerate the migration for you.)

### Step 5 — Set environment variables
In Railway → your service → Variables, add each of these (from your `.env`):

| Variable | Value |
|---|---|
| `BOT_TOKEN` | your **fresh** BotFather token (revoke the old one first!) |
| `MASTER_KEY` | your 64-hex-char key (same one — needed to decrypt existing wallets) |
| `FEE_WALLET` | your fee wallet public key |
| `FEE_BPS` | `50` |
| `RPC_URL` | a fast RPC (Helius/Triton/QuickNode — see below) |
| `JUPITER_BASE_URL` | `https://lite-api.jup.ag` |
| `NODE_ENV` | `production` |
| `BOT_MODE` | `webhook` |
| `WEBHOOK_URL` | your Railway public URL + no path, e.g. `https://velox-production.up.railway.app` |
| `PORT` | `8080` |
| `DATABASE_URL` | (auto-added by Railway Postgres) |

⚠️ `MASTER_KEY` must be the **exact same** key as before, or wallets created on your PC
can't be decrypted. If you're starting fresh with no real wallets yet, a new key is fine.

### Step 6 — Set the start command
Railway → Settings → Deploy → Start Command:
```
npx prisma migrate deploy && npm start
```
This applies the DB schema then boots the bot.

### Step 7 — Deploy & verify
- Railway builds and starts automatically. Watch the **Deploy Logs**.
- You want to see: `bot listening` (webhook mode) with your port.
- Message @VeloxSolBot `/start` — it should reply. Done. It now runs 24/7. ✅

---

## Path B — VPS (more control)

For a Hetzner/DigitalOcean/Contabo Ubuntu server.

### Step 1 — Get a server
Create the cheapest Ubuntu 22.04 VPS (~$5/mo). Note its IP. SSH in:
```bash
ssh root@YOUR_SERVER_IP
```

### Step 2 — Install Node + tools
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs git
npm install -g pm2
```

### Step 3 — Get the code + secrets
```bash
git clone https://github.com/YOU/velox.git
cd velox
npm install
nano .env        # paste your env values (see table above; polling mode is fine on a VPS)
npm run build
npx prisma migrate deploy
```
On a VPS you can keep `BOT_MODE=polling` (simpler — no webhook URL needed).

### Step 4 — Run it 24/7 with pm2
```bash
pm2 start dist/index.js --name velox
pm2 save
pm2 startup     # run the command it prints — makes the bot survive reboots
```

### Step 5 — Verify + monitor
```bash
pm2 logs velox      # watch output; look for "bot online"
pm2 status          # should say "online"
```
Message @VeloxSolBot `/start`. Runs 24/7 and restarts on crash/reboot. ✅

---

## Get a fast RPC (do this before real users)

The public `api.mainnet-beta.solana.com` is slow and rate-limited — trades will lag/fail
under load. Get a free-tier key from one of these and put its URL in `RPC_URL`:
- **Helius** (helius.dev) — popular for Solana bots
- **QuickNode** (quicknode.com)
- **Triton / others**

Free tiers are fine to start; upgrade as volume grows. This directly improves the "speed"
you're marketing.

---

## After deploy — the must-dos

- [ ] **Revoke the Telegram token** you pasted in chat (BotFather → `/revoke`) and use the
      new one in your server env. The old one is compromised.
- [ ] Confirm `FEE_WALLET` is a wallet **you control** (that's where your 0.5% lands).
- [ ] Set a real RPC URL.
- [ ] Do ONE tiny real trade yourself to confirm the full money path works.
- [ ] Only THEN start promoting (see LAUNCH_KIT.md).
- [ ] Security hardening before strangers deposit — it's custodial (ask me to do this).

---

## Costs summary

| Item | Cost |
|---|---|
| Railway or VPS | ~$5/mo |
| RPC (Helius etc.) | Free tier to start |
| Domain (optional) | ~$10/yr |
| Security audit (before scale) | $8k–$50k+ (from fee revenue later) |

Start cheap. Scale spending from the fees you earn.
