# Buzz Wallet: Telegram Mini App + server

One Node.js server does everything: it serves the app, stores all data in Postgres, talks to your bot, connects Trust Wallet, sends automatic payouts and runs the Admin panel. Nothing is lost when users close the app.

## How referrals work
1. A friend opens `t.me/YourBot?start=ref_<referrer id>` and sends /start.
2. The bot replies with your welcome message and an **Open Buzz Wallet** button. The referral is saved as **pending** and the referrer gets a message.
3. The friend opens the Mini App. A **Join to continue** screen lists your required channels and cannot be skipped — the server checks membership, not just the screen.
4. The moment they pass, the referral becomes **completed**, the reward is credited, and the referrer is notified.

Only brand-new users count, and nobody can refer themselves.

## How wallets and withdrawals work
1. In **Withdrawal**, the user taps **Connect wallet**. This opens Trust Wallet (via WalletConnect) and asks only to read their BEP20 (BNB Smart Chain) address — no spending permission is requested.
2. Once approved, that address is saved to their account permanently (an admin can reset it in Users if the user needs to switch wallets).
3. When they request a withdrawal, if **Auto payout** is on in Admin > Settings, the server calls your payout API immediately and the user gets a message once it's sent.
4. If auto payout is off, or the payout API gives an unclear answer, the withdrawal waits in Admin > Withdrawals for you to send or reject it. The user's balance is held (not lost) until you decide.
5. A failed payout (rejected by the API) is refunded to the user automatically. An unclear result (timeout, 5xx, or an odd reply) is never auto-refunded — it's flagged for you to check by hand, so nobody is refunded for a payout that may have already gone through.

## Folder layout
```
package.json  .env.example  README.md
server/
  index.js        starts everything, sets the webhook and menu button
  config.js       environment settings
  db.js           tables, default settings, transactions
  auth.js         verifies Telegram login data, admin check, channel gate
  initdata.js     signature check
  telegram.js     Bot API helper
  services.js     referrals, gate, tasks, withdrawals, payouts, balance, broadcast
  errors.js
  routes/  user.js  admin.js  webhook.js
public/           the Mini App (index.html, style.css, js/...)
  js/wallet.js    WalletConnect / Trust Wallet address connection
  js/pages/       gate, dashboard, profile, history, referral, task, withdrawal, admin
  js/pages/admin/ overview, settings, channels, tasks, proofs, payouts, users, broadcast
```

## Deploy (Render + free Postgres)

### ⚠️ The #1 cause of "Exited with status 1" on first deploy
Render must see **`package.json` at the very top of your repository**, not inside a subfolder. When you unzip `buzz-wallet.zip` you get a `buzz-wallet` folder — upload what's **inside** that folder, not the folder itself.
```
✅ correct: repo root has package.json, server/, public/
❌ wrong:   repo root has buzz-wallet/ (containing package.json, server/, public/)
```
If you already deployed the wrong way: on GitHub, open the repo, move every file out of the `buzz-wallet` subfolder into the root (drag-and-drop upload again, or delete and re-upload), commit, and Render will redeploy automatically.

### Steps
1. **Database:** create a free Postgres at neon.tech and copy its connection string.
2. **Code:** upload the *contents* of the `buzz-wallet` folder (not the folder itself) to a GitHub repository.
3. **Render:** New > Web Service > pick the repo.
   - Build command: `npm install`
   - Start command: `npm start`
4. **Environment variables** (Render > Environment):
   - `BOT_TOKEN` = token from @BotFather
   - `DATABASE_URL` = the Postgres connection string
   - `ADMIN_IDS` = `7995243814` (comma separate to add more admins)
   - `PUBLIC_URL` = your Render address, like `https://buzz-wallet.onrender.com`
5. Deploy. The Logs tab should show `Server running on port ...`, `Bot: @yourbot`, and `Webhook and menu button set to ...`. The server creates every table automatically; deploying again never deletes data.
6. Open your bot and send /start — you should get the welcome message with the button.

> Static hosts (Vercel, Netlify) cannot run this — it needs a real server. Render's free plan sleeps when idle, so the first open after a while can take ~30 seconds.

## Admin panel
Open the app with the admin account and tap **Admin panel**. Nothing is hardcoded — everything is set here:

| Tab | What it does |
|-----|--------------|
| Overview | users, balances, referrals, tasks, proofs, withdrawals, payouts needing review |
| Settings | referral reward, min/max withdrawal, welcome message, **auto payout on/off, payout API address, API key, token contract address**, WalletConnect project ID |
| Channels | channels/groups every user must join before using the app |
| Tasks | create tasks: **Auto** (bot checks membership) or **Screenshot** (you approve uploads) |
| Proofs | review task screenshots |
| Withdrawals | see auto payouts in progress, send a waiting one manually, mark paid, or reject (refunds the user) |
| Users | search by ID or @username, add/remove balance, reset a user's connected wallet |
| Broadcast | send a message to all users through the bot |

### Setting up automatic payouts
In Admin > Settings:
1. Turn on **Pay withdrawals automatically**.
2. Paste your **payout API address** (defaults to `https://pt-kappa-ten.vercel.app/pay/bep20`).
3. Paste your **API key** (kept private — never shown again, only a short hint).
4. Paste the **token contract address** (the BEP20 token users get paid in).
5. Save.

The server sends `{ api_key, to_address, token_address, amount }` to that API for every withdrawal. Only `to_address` (the user's wallet) and `amount` (what the user typed) come from the user — the API key and token address always come from your saved settings, never from the browser.

### Setting up wallet connect
1. Go to **cloud.reown.com** (WalletConnect's dashboard), sign up free, create a project (type "App").
2. Copy the **Project ID** into Admin > Settings > WalletConnect Project ID, and save.
3. If connecting fails for users, add your Mini App's web address to that project's allowed domains on Reown's dashboard.

### Channels and auto-verify tasks
The bot must be an **admin** in every channel/group you use, or it can't check membership. The panel warns you if it isn't, and refuses to save a channel it can't see. Admins skip the join screen, so test it with a normal account.

## Run on your computer (optional)
Copy `.env.example` to `.env`, fill it in, then `npm install` and `npm start`. Telegram needs an https address, so use a tunnel such as ngrok and set `PUBLIC_URL` to it.
