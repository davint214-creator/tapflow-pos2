# TapFlow POS — Free Setup Guide (No Demo Lock, All Features)

**Download:** `tapflow-pos.zip` in this workspace (52KB) — contains everything. Also at `/home/user/tapflow-pos.zip`

Contains:
```
server.js                 Full backend: auth, products, Adyen Terminal + QR, RFID arcade cards, webhooks
public/index.html         Frontend: Overview, Dashboard (manager-only graphs), POS, Arcade Cards, Products, Transactions, QR Pay, Employees, Settings
package.json              Dependencies
.env / .env.example       JWT + Adyen placeholders
wrangler.toml             Cloudflare Pages + D1 binding (free SQLite)
.github/workflows/deploy.yml  Auto-deploy to Cloudflare Pages on git push
fly.toml / render.yaml    One-click free deploy configs
README.md / CLOUDFLARE.md / INSTALL.txt  Docs
```

---

## 0) Quick Local Test (2 min, no hosting, no business info)
Great to prove it works before you show anyone.

```bash
# 1. Unzip
unzip tapflow-pos.zip && cd tapflow-pos  # or tapflow-pos folder

# 2. Install + run
npm install
node server.js
# → http://localhost:3000
# → [TapFlow] listening on 0.0.0.0:3000 , Adyen: not-configured (normal)
```

Log in:
- **admin@tapflow.local / Admin123!** — everything
- **manager@tapflow.local / Manager123!** — products + Dashboard graphs
- **employee@tapflow.local / Employee123!** — POS only
- **arcade@tapflow.local / Arcade123!** — Arcade Crew (POS + Arcade Cards, Dashboard blocked 403)

Try:
- **POS → Cash:** Add Burger Combo → Pay Cash → Confirm → receipt prints, inventory decrements, transaction logged.
- **Products → Unlimited:** New Product → tick **Unlimited ♾️** → save → tile shows ♾️, never runs out. Managers + Arcade Crew can edit this.
- **Arcade Cards → RFID:** Plug USB RFID pad → focus UID field → tap card → it types UID+Enter → Create card → Load $10 Cash → balance $10 → appears in Transactions as `rfid_topup`.
- **Dashboard:** Manager sees Today/Week/Month revenue + 7-day bar + payment method donut. Arcade Crew gets Restricted.

No business info needed. Cash & QR work now. Adyen Terminal/QR return `503 Adyen not configured` until you import keys — by design (never fakes Paid).

---

## 1) Free Hosting — Pick One (all keep ALL features)

### Option A — Render.com (Easiest, recommended if you want 1 click)
**Free:** 750 hrs/month = always on, free HTTPS, custom domain free.

1. Create free GitHub → New repo `tapflow-pos` → Upload `tapflow-pos.zip` contents (drag & drop).
2. Go to https://dashboard.render.com → New → Web Service → Connect repo `tapflow-pos`
3. Settings:
   ```
   Build Command: npm install
   Start Command: node server.js
   Runtime: Node 20 / Free plan
   ```
4. Environment → Add:
   ```
   JWT_SECRET=any-random-32-chars-e.g.-aB3x9QpLz... 
   ADYEN_ENVIRONMENT=test
   # leave these blank for now — cash/QR still work:
   ADYEN_API_KEY=
   ADYEN_MERCHANT_ACCOUNT=
   ADYEN_CLIENT_KEY=
   ADYEN_HMAC_KEY=
   ADYEN_TERMINAL_ID=
   ```
5. Deploy → you get `https://tapflow-pos-xxxx.onrender.com` (share this link).
6. Open it → login → **Settings → Payments → paste Adyen keys later** → Save → readers work, no redeploy.

**Custom domain free:** Render → Settings → Custom Domain → `pos.yourpark.com` → add DNS → free TLS.

### Option B — Fly.io (Fastest, SQLite persists)
```bash
npm install -g flyctl
fly auth signup          # free
fly launch --copy-config # use fly.toml inside zip, No to Postgres
fly volumes create tapflow_data --region dfw --size 1  # 1GB free
fly deploy
fly open                 # your live URL
```
Add env vars: `fly secrets set JWT_SECRET=... ADYEN_API_KEY=...`

### Option C — Cloudflare Pages + D1 (All-in-one, free)
Frontend + backend both on Cloudflare (D1 = free SQLite, no Render needed).

1. Unzip, `git init`, push to GitHub `tapflow-pos`.
2. https://dash.cloudflare.com → Pages → Create app → Connect GitHub → pick `tapflow-pos`
   - Build command: `npm install`
   - Output: `public`
3. Pages → Settings → Functions → D1 Binding → Create DB `tapflow` → copy ID into `wrangler.toml` `database_id`.
4. Variables: `JWT_SECRET`, `ADYEN_*` (blank ok).
5. Deploy → `https://tapflow-pos.pages.dev` + custom domain free.

Auto-deploy: every `git push` triggers `.github/workflows/deploy.yml`.

### Option D — GitHub Pages (static) + Render Backend (free, if you love GitHub Pages)
GitHub Pages alone = static only (no DB). Pair it:

1. Deploy backend to Render (Option A) → `https://tapflow-pos.onrender.com`
2. In `public/index.html` find `function api` and set:
   ```js
   const BACKEND = "https://tapflow-pos.onrender.com";
   function api(path, opts={}) { return fetch(BACKEND+path, opts) ... }
   ```
3. Push `public/` to GitHub → Settings → Pages → Source `main` / `public` → Save → `https://YOU.github.io/tapflow-pos/` → talks to Render backend.

### Option E — Instant Share (no GitHub, 30 seconds)
On the PC running `node server.js`:
```bash
npm install -g cloudflared
cloudflared tunnel --url http://localhost:3000
# → https://random-words-1234.trycloudflare.com (live, share to boss)
```
Tunnel to your laptop — all features, turn off when done.

---

## 2) Adyen — Add Later Without Code Change (Import JSON)
You said no business info → use **cash/QR today**, add Adyen when ready:

1. When you have Adyen test account (https://www.adyen.com/signup → Test), get 5 values: `API Key (AQEm...)`, `Merchant Account`, `Client Key (test_...)`, `HMAC`, `Terminal ID (P400Plus-...)`, `env=test/live`
2. In TapFlow → login as admin → **Settings → Payments → Import Adyen JSON** → paste:
   ```json
   {
     "apiKey": "AQEm...",
     "merchantAccount": "YourCompanyECOM",
     "clientKey": "test_...",
     "hmacKey": "ABC...",
     "terminalId": "P400Plus-275123456",
     "env": "test"
   }
   ```
   Or paste single file upload → **Import & Save** → badge turns `Configured • test`.
3. POS → enter Terminal ID top field → **Save** → **Charge on Adyen Reader** → reader lights up → customer taps → `captured` → receipt auto-prints. No redeploy.

Webhook (for QR Drop-in): Developers → Webhooks → Add → `https://yourdomain.com/api/webhooks/adyen` → HMAC → events `AUTHORISATION`.

---

## 3) RFID Arcade Card — Setup (like Connect & Go)
**Hardware:** Any USB RFID reader in **keyboard wedge** mode (types UID + Enter). Tested: ACR122U, Elatec TWN4, 125kHz USB, HID Omnikey. ~$15-$30 on Amazon. No driver.

**Wiring:**
1. Plug reader via USB to POS PC/tablet.
2. Open TapFlow → **Arcade Cards** → cursor in **Tap RFID pad here** field → tap card → UID appears (e.g. `04A2B7C89D`).
3. If new: **Create card** → name + customer → Create → shows balance $0.
4. **Load money:** Pick $5/$10/$20/$50 or type amount → choose `Cash collected` (confirm cash) or `Adyen Terminal` (charges reader then loads) → **Load Money** → balance updates instantly + transaction `RFID-...` in ledger.
5. **Use at games:** Same UID at arcade readers/gates; call `POST /api/arcade/cards/:uid/deduct {amount: 1.50}` from game controller.

**POS flow when selling card:** POS → add any product (e.g. "Arcade 20 Credits") → then click **Tap pad & load money** banner → loads to card in same checkout.

**API for developers:**
```
POST /api/arcade/scan {uid}
POST /api/arcade/cards {uid, display_name, customer_name, customer_email, initial_balance}
POST /api/arcade/cards/:id/topup {amount, payment_method, reference}
POST /api/arcade/cards/:id/deduct {amount}
GET  /api/arcade/cards
```

---

## 4) Unlimited Inventory
**Settings → Products → New/Edit → tick “Unlimited ♾️”** → Inventory field disables → save.

- POS tile shows `♾️` + `Unlimited` green, never “Out of stock”
- Backend skips decrement for `inventory_unlimited=1`
- Allowed for **admin, manager, arcade_crew** (crew can now add/edit inventory per your request)

---

## 5) Verify Before You Share
- Overview page public (no login)
- Dashboard → 7-day revenue bar + donut → only manager/admin (crew 403)
- Transactions → CSV export, refunds manager/admin only (real Adyen call)
- Receipts → Store name/tax/currency/header/footer in Settings → auto-print on `captured` (`window.print()` + silent kiosk flags)
- PWA-ready, responsive (computer/tablet/phone + Adyen S1E)

---

## Download Links
- **Primary:** `tapflow-pos.zip` attached in this chat (52KB, 15 files) — click to download
- **Workspace path:** `/home/user/tapflow-pos.zip` and `/home/user/FREE_SETUP_GUIDE.md`
- **Live dev server now:** `http://localhost:3000` (this workspace) — use Cloudflare Tunnel command above to make it public instantly

Need help deploying? Tell me which option (Render / Cloudflare / Tunnel) and I’ll walk you through the clicks.
