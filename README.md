# Stocks Mgt V2

Encrypted bookkeeping for equity trades in USD and OMR. It's a rebuild of the Stocks app around an
append-only **ledger**: you record facts (buys, sales, dividends, splits, prices), and every figure is
calculated from them. Nothing derived is stored, so nothing can drift out of step.

## Use it

| Task | Where |
|---|---|
| Record a buy, sale, dividend or split | **+** (phone) / **New transaction** or `N` (PC) |
| Choose which lots a sale closes, and see the P&L before saving | the sell form → **Choose lots** |
| Match the broker's contract note exactly | the form → **Settled amount differs** |
| Update many prices at once (or paste a list like `MSFT 410.20`) | **Prices** |
| Set a target price, stop-loss or notes for a stock | a holding → ✎ |
| Record cash in or out, or move money between accounts (e.g. OMR → USD) | Dashboard → **Cash** card |
| Start tracking cash, or correct it to the broker's figure | Cash card → **Set balance** / **Reconcile** |
| Undo any change | toast → **Undo**, the ↶ button, or `⌘Z` / `Ctrl+Z` |
| Sync with your other devices | Settings → **GitHub Gist sync** |

Keyboard (PC): `N` new transaction · `1`–`6` pages · `⌘Z` undo · `L` lock · `Esc` close.

## Cash

Each account's cash balance is calculated, never typed in: deposits − withdrawals − buys + sales +
dividends ± transfers ± interest and charges. A **transfer** records both the amount sent and the
amount received, so the exchange rate you actually got is kept.

V1 has no record of deposits, so migrated accounts start as **not tracked**. The book can't know how much
cash you began with. Use **Set balance** once per account: type the cash your broker shows, and the app
records the difference as an adjustment. After that, every trade keeps it up to date. Use **Reconcile**
whenever the broker and the book disagree. A tracked balance below zero is flagged on the dashboard.
The dashboard's headline becomes **total value** (holdings + cash) once any account is tracked.

## Moving from V1 (once)

1. In the V1 app: **Setup → Export Backup (JSON)**.
2. Open V2 and create its password. (It can be the same as V1's or a new one.)
3. Choose **Import from V1** on the welcome screen, or Settings → Data, and pick the file.
4. Check the report. Every V1 lot is recalculated and compared with V1's own realized P&L and
   remaining shares. The import button only appears after you've seen it.

How the data maps:

| V1 | V2 |
|---|---|
| portfolio | account (same id, currency and rate) |
| purchase | **buy** at `cost`, the amount actually charged. This isn't always qty × price + commission; for example lot #1 was charged 26,017.65, not 26,018.15. |
| sale | **sale** that closes exactly the V1 purchase it was recorded against |
| `company` ("Microsoft MSFT") | stock name "Microsoft". A lot with a different label, e.g. "(Ahmed)", keeps it as a note. |
| targetPrice / stopLoss per lot | the stock's target / stop-loss (latest lot that has one) |
| prices | price history |
| derived fields (`remQty`, `realizedPL`, …) | ignored and recalculated |

V1 and its Gist copy aren't touched. V2 syncs to its own Gist file (`stocks-mgt-v2.json`).

## Deploy to GitHub Pages

```bash
npm install
npm test
npm run build
```

`dist/` then contains everything the site needs: one self-contained `index.html`, plus `sw.js`,
`manifest.json` and the icons. Upload those files (not the `dist` folder itself) to the root of
**their own GitHub repository**, then turn on Settings → Pages (branch `main`, folder `/ (root)`).

**Don't put V2 inside V1's site.** An installed app owns every address below its own, so if V2 is
served from V1's repository, or from any folder under V1's address (e.g. `…github.io/Stocks/v2/`), the
phone treats V2 as part of the already-installed V1 and won't install it. Separate repositories give
separate addresses (`…github.io/Stocks/` and `…github.io/stocks-mgt-v2/`), which install as two apps.
V2's manifest also has its own fixed `id` (`/stocks-mgt-v2`), name and icons, so the two can't be
mistaken for each other.

**On every deploy, bump `CACHE` in `public/sw.js`** (`smv2-v1` → `smv2-v2` …), so installed copies pick
up the new version.

The app needs its **https** address (GitHub Pages provides it). Encryption and fingerprint unlock use
the browser's built-in cryptography, which only runs on secure pages.

## Security

- All data is encrypted with a random AES-256 key: the book, history, snapshots and the Gist token.
  That key is stored only in wrapped form: once under your password (PBKDF2-SHA256, 310,000 rounds) and
  once under your fingerprint or face passkey (WebAuthn PRF). Changing the password re-wraps it, so
  biometrics and sync keep working.
- Auto-lock defaults to 5 minutes. Locking drops every decrypted copy from memory.
- **There is no password recovery.** Keep Gist sync on, or download backups.

## Sync (secret Gist)

One **Sync** fetches the encrypted copy, **merges it record by record** with this device (the latest
edit wins, deletions travel as tombstones), and pushes the result. Trades added on two devices are both
kept. The Gist holds only ciphertext: the data is encrypted with a sync key, and that key is wrapped by
a key derived from your password. A new device asks for that password once. Automatic sync runs on
unlock, on returning to the app, and 15 seconds after each change.

## Code map

```
src/domain/     pure logic, no UI. Unit tested.
  types.ts        the book: accounts, instruments, txns, prices, settings
  money.ts        integer money (fils / cents), prices in millionths
  ledger.ts       replays transactions → lots, cost basis, realized P&L, ledger checks
  analytics.ts    XIRR, totals, monthly/yearly series, value over time, alerts
  merge.ts        per-record merge for sync
  migrate-v1.ts   once-off V1 import + verification report
src/store/      side effects
  crypto.ts       Web Crypto + WebAuthn PRF
  vault.ts        unlock, data key, encrypted IndexedDB records
  sync.ts         encrypted, merging Gist sync
  app.ts          signals state; commit() is the single path for every change (undo, history, snapshots)
src/ui/         Preact components and pages
tests/          ledger, merge and the V1 golden test (reads ../index.html at test time)
```

The golden test (`tests/migrate-v1.test.ts`) migrates the real V1 history and requires every lot's
realized P&L to match V1. It reads that data from the V1 app at test time, so no trades are copied into
this folder. It skips itself if V1's `index.html` isn't there.
