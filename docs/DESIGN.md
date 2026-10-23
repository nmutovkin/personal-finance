# Family Finance Service — Design Proposal

## 1. Problem

The family holds money in many accounts across several banks (and possibly in
several currencies), plus physical cash. Today there is no single place that
answers:

- How much do we have in total, right now, and where is it?
- How did that change over time (net worth trend)?
- Where does the money go (spending by category, by person, by month)?
- How much cash is in the wallet / at home?

## 2. Requirements

### Must have (MVP)

| # | Requirement |
|---|-------------|
| R1 | Register any number of accounts: bank cards, current accounts, savings/deposits, cash wallets, (later) brokerage. |
| R2 | Get transactions/balances into the system from every bank (automatic where possible, statement import otherwise). |
| R3 | Enter cash transactions manually, fast, from a phone. |
| R4 | Total balance across all accounts, converted to one base currency. |
| R5 | Transfers between own accounts are not counted as income/expense. |
| R6 | Categories + monthly statistics (spending by category, income vs. expense). |
| R7 | Several family members, each seeing the shared picture. |
| R8 | Data is private and backed up. |

### Nice to have (later)

- Net worth history chart, budgets per category with alerts.
- Auto-categorization rules (merchant → category), optionally LLM-assisted.
- Investments / deposits with interest, loans and their schedules.
- Recurring payments detection (subscriptions).
- Telegram bot for entering cash and getting a daily/weekly summary.

## 3. The hard part: getting data out of banks

Everything else (ledger, reports, UI) is standard CRUD and is easy. Bank data
ingestion decides the architecture, and it depends **entirely on which country
the banks are in**. Options, from most to least automatic:

| Method | How it works | Where it works | Effort / reliability |
|--------|--------------|----------------|----------------------|
| **Open-banking aggregator API** | You authorize read-only access in the bank's own UI; an aggregator returns accounts, balances, transactions via REST. | EU/UK (PSD2): e.g. Enable Banking, Salt Edge, Tink, TrueLayer. US/Canada: SimpleFIN Bridge, Plaid. Brazil: Pluggy. | Best UX. Consent must be renewed every ~90–180 days. Free/cheap personal tiers exist but change often — verify current terms before committing. |
| **Bank's own API** | Some banks expose personal APIs (often only for investments or business accounts). | Varies by bank. | Good where available, but rare for personal accounts. |
| **Statement import (CSV / OFX / QIF / XLSX / PDF)** | Download a statement from internet banking, upload it; a per-bank parser maps it to transactions. | Everywhere. | Always possible. Manual step (~1 min per bank per week/month). Needs one parser per bank format. |
| **Notification parsing** | Parse bank SMS/push/e-mail notifications (forwarded to a mailbox or captured by a phone app). | Everywhere banks send notifications. | Near-real-time, but fragile (format changes) and usually lacks a reliable balance. Good as a supplement. |
| **Manual balance snapshot** | Type the current balance of an account once in a while. | Everywhere. | Zero integration. Enough for deposits/savings that rarely move — gives net worth without transactions. |
| ~~Screen scraping with stored bank passwords~~ | Log in as you with a headless browser. | — | **Not recommended**: breaks 2FA, violates bank ToS, and storing bank passwords is a serious security risk. |

**Realistic design:** support several ingestion methods per account and let each
account pick one. Typically: aggregator API for banks that support it, statement
import for the rest, manual entry for cash, balance snapshots for deposits.

### 3.1 Russian banks (our case)

The banks are Russian, which changes the picture:

- **No PSD2-style aggregator.** International aggregators (Plaid, Tink, TrueLayer,
  Salt Edge, Enable Banking, …) do not serve Russian banks.
- **Bank of Russia Open API is coming, but not usable for us yet.** The CBR has
  published standards for access to account information with user consent
  (orders ОД-2887 / ОД-2895 of 19.12.2025), and the open-API law planned to make
  it mandatory for the largest banks starting 2026. Access is meant for
  registered financial institutions, not hobby apps, so we should not count on
  it near-term. Keep the importer interface ready to add it later.
- **ZenMoney (Дзен-мани) as a sync layer.** ZenMoney already syncs with most
  large Russian banks (Sber, T-Bank, Alfa, VTB, …; check coverage for each of
  your banks) and has an API (`POST https://api.zenmoney.ru/v8/diff/`, OAuth
  token, incremental sync) with community clients in Python/Node. Paid
  subscription. Its bank connectors are the open-source `ZenPlugins` project.
  **This is the most practical way to get automatic sync for Russian banks.**
- **Statement import works with every bank.** Banks export statements, usually
  as PDF; some also offer CSV/XLSX/OFX (T-Bank, for example, exports operations
  from its web version). Expect to write one parser per bank. PDF parsing is
  more fragile but doable (`pdfplumber` / `camelot`).
- **Push/SMS notifications** give near-real-time data without third parties:
  - Android: an app such as MacroDroid/Tasker (or a small custom app) forwards
    bank push notifications or SMS to our webhook;
  - iPhone: iOS can't read other apps' push notifications, but a Shortcuts
    automation can forward incoming SMS to a webhook (only if the bank still
    sends SMS; many now charge for SMS or have moved to push only).
  - Treat notifications as provisional; reconcile with statements monthly.
- **Investments:** T-Bank's brokerage has an official token-based API
  (T-Invest API). Other brokers: import their broker reports.
- **FX rates:** the Bank of Russia publishes official daily rates for free
  (`https://www.cbr.ru/scripts/XML_daily.asp`). Base currency: RUB.
- **Hosting:** a Russian VPS (paid with a Russian card) or a home server behind a
  VPN. Telegram bots work but Telegram has had disruptions in Russia, so keep a
  PWA quick-add as a fallback for cash entry.

**Ingestion plan per account type:**

| Account | Primary source | Fallback |
|---------|----------------|----------|
| Cards / current accounts at large banks | ZenMoney sync → our DB via API | Statement import + notifications |
| Small banks without ZenMoney support | Statement import (CSV/XLSX/PDF) | Notifications, manual balance |
| Deposits / savings | Manual balance snapshot (monthly) | Statement import |
| T-Bank brokerage | T-Invest API | Broker report import |
| Cash | Telegram bot / PWA quick-add | — |

### 3.2 Minimizing manual input (main goal)

Target: the family only ever types something for cash, and even that is optional.

**Ingestion** (pick the highest level that is acceptable):

| Level | Setup | Manual work per month |
|-------|-------|-----------------------|
| 1 | ZenMoney API sync for all supported banks | ~0 (re-link a bank when its session expires) |
| 2 | No ZenMoney, Android phones: push/SMS forwarding + statements e-mailed on a schedule, parsed from a mailbox | ~5–15 min (banks without scheduled e-mail statements) |
| 3 | No ZenMoney, iPhones: SMS forwarding where available + monthly statement upload | ~20–40 min |

Levels can be combined (e.g. ZenMoney + notifications for instant updates).

**Processing — no manual work after data arrives:**

- **Dedup:** notification, statement and API copies of the same payment are
  merged (match on account, amount, time ±1 day, merchant); the statement or
  API record wins.
- **Transfers between own accounts** are matched automatically, so they never
  count as income or expense.
- **Categorization:** 1) user rules (merchant → category), 2) MCC code from
  the statement, 3) an LLM for unknown merchants; anything categorized once is
  remembered as a rule. Manual review only for low-confidence items, shown
  as a weekly "check these 3 transactions" message in the bot.
- **Deposits:** balance is computed from rate and terms; interest notifications
  and statements confirm it.

**Cash with minimal input:**

- An ATM withdrawal is automatically a transfer card → owner's cash wallet.
- Logging individual cash purchases is optional.
- Once a week the bot asks "how much cash do you have?". The difference from the
  expected balance is booked as "cash spending (uncategorized)". One number a
  week keeps cash totals correct.

### 3.3 Our setup: Sber, T-Bank, Alfa · Android · no ZenMoney

**Building blocks**

- **Notification forwarder** on every family phone: an Android
  `NotificationListenerService` (small custom app, or MacroDroid/Tasker at first)
  sends bank push/SMS text to our webhook. Each phone sees only its owner's
  cards, so every member needs it. Battery-optimization whitelisting is
  required (especially on Xiaomi/Huawei/Samsung).
- **Notification parsers** per bank (Sber, T-Bank, Alfa): amount, currency,
  merchant, card/account, balance if present. Kept as regex templates with tests
  on real samples.
- **Statement importers** per bank: T-Bank (CSV/XLSX export), Sber (PDF),
  Alfa (PDF/XLSX). Statements requested in the bank app with "send to e-mail" go
  to a dedicated mailbox; the service picks them up via IMAP automatically.
- **Telegram bot** for cash and weekly review.

**Variants**

| Variant | Sources | Manual work | Risk |
|---------|---------|-------------|------|
| V1 (recommended) | Notifications + monthly statements via e-mail | ~5 min/month (3 statement requests) + weekly cash number | Low: read-only data |
| V2 | Notifications only | ~0 + cash | Low, less accurate: misses whatever banks don't notify (some fees, interest) and anything missed while a phone is off |
| V3 | V1 + automated login to bank web/mobile APIs | ~0 + cash | High: stored credentials give full access to money; 2FA; anti-fraud blocks; private APIs break |

V3 can be added later as an optional, isolated module if the family accepts
the risks (see §8).

## 4. Solution options

### Option A — Use an existing open-source tool (self-hosted)

| Tool | Strengths | Weaknesses |
|------|-----------|------------|
| **Firefly III** (+ Data Importer) | Mature, double-entry, multi-currency, rules, budgets, reports, REST API. Importer supports CSV and some aggregators. | PHP stack, UI is "accounting-like"; family sharing is weaker than a purpose-built app. |
| **Actual Budget** | Fast, local-first, great mobile web UI, envelope budgeting, bank sync via some aggregators. | Multi-currency support is limited. Budget-oriented rather than net-worth-oriented. |
| **Beancount + Fava** | Plain-text ledger, excellent multi-currency and reporting, everything in git. | Not family-friendly for non-technical members; cash entry from a phone needs extra tooling. |
| **Commercial apps** (e.g. ZenMoney, YNAB, Monarch, etc.) | Zero effort, many bank integrations already done. | Subscription, your data on their servers, bank coverage depends on region. |

**Effort:** 1–3 days to deploy and configure. **Risk:** low. **Fit:** good if the
tool's bank coverage matches your banks.

### Option B — Build a custom service

Build exactly what the family needs: a small web app (mobile-first PWA) +
Telegram bot, with pluggable bank importers.

**Effort:** MVP 4–8 weeks of part-time work (see §7). **Risk:** medium — mostly
in importers. **Fit:** best, and a good learning project.

### Option C — Hybrid (recommended starting point)

Use **Firefly III as the ledger/back-end** (accounts, transactions, currencies,
reports, API) and write only the parts that are missing for your situation:

- custom importers for your specific banks (statement parsers, aggregator sync);
- a Telegram bot / tiny PWA for 5-second cash entry;
- a simple family dashboard (total balance, net-worth chart) on top of its API.

**Effort:** working system in ~1 week, custom pieces added incrementally.
If Firefly III later feels limiting, the importers and bot are reusable for a
fully custom back-end (Option B).

### Recommendation for Russian banks

There is no open-banking aggregator for Russian banks, so pick based on how
much you want to depend on ZenMoney:

1. **Just ZenMoney.** It already covers bank sync, cash entry, categories and
   reports. Zero development. Good enough if its reports and family sharing suit you.
2. **ZenMoney as the data source + our own service (recommended).** A worker
   pulls everything via the ZenMoney `diff` API every ~15 min into our
   PostgreSQL DB. We build what ZenMoney lacks: a family dashboard and net-worth
   history, our own statistics, a Telegram bot, T-Invest API sync, custom rules.
   If ZenMoney ever goes away, our data is already stored locally and statement
   importers take over.
3. **Fully independent.** No third parties: statement importers + notification
   forwarding + manual entry (Option B or C). Most private, least automatic:
   about 15–30 minutes a month uploading statements.

## 5. Architecture (for Option B, or the custom parts of Option C)

```
                ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
                │  Web PWA     │   │ Telegram bot │   │ Statement    │
                │ (dashboard,  │   │ (cash entry, │   │ upload       │
                │  cash entry) │   │  summaries)  │   │ (CSV/PDF/OFX)│
                └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
                       └──────────┬───────┴──────────────────┘
                                  ▼
                        ┌───────────────────┐
                        │   API (backend)   │  auth, accounts, transactions,
                        │                   │  categories, rules, reports
                        └─────────┬─────────┘
                                  │
     ┌────────────────────────────┼─────────────────────────────┐
     ▼                            ▼                             ▼
┌───────────────┐       ┌───────────────────┐         ┌──────────────────┐
│  PostgreSQL   │       │ Scheduler/workers │         │ FX rates fetcher │
│  (ledger)     │       │ - aggregator sync │         │ (daily, e.g. ECB │
│               │       │ - dedup/transfer  │         │  or central bank)│
│               │       │   matching        │         └──────────────────┘
│               │       │ - categorization  │
└───────────────┘       └─────────┬─────────┘
                                  ▼
                       ┌──────────────────────┐
                       │ Bank importers       │  one plugin per bank /
                       │ (plugin interface)   │  aggregator / file format
                       └──────────────────────┘
```

Suggested stack (any mainstream stack works; this one is small and fast to build):

- **Backend:** Python + FastAPI (rich ecosystem for CSV/PDF parsing), or
  TypeScript + NestJS/Next.js if you prefer one language end-to-end.
- **DB:** PostgreSQL (SQLite is fine for a first prototype).
- **Frontend:** React/Next.js or SvelteKit as an installable PWA; charts via
  ECharts/Recharts.
- **Bot:** Telegram Bot API (aiogram / python-telegram-bot).
- **Deploy:** Docker Compose on a home server (NAS / Raspberry Pi / mini-PC) or a
  small VPS (~$5/month). Access via Tailscale/WireGuard or HTTPS + 2FA.

### Importer plugin interface

```python
class Importer(Protocol):
    id: str                      # "bank_x_csv", "enable_banking", ...
    def fetch(self, account: Account, since: date) -> list[RawTransaction]: ...
    # or, for file-based importers:
    def parse(self, file: bytes) -> list[RawTransaction]: ...
```

Every raw transaction gets a stable `external_id` (bank id, or a hash of
date + amount + description + running balance) so re-importing the same
statement is idempotent.

## 6. Data model (core)

```
member(id, name, telegram_id, role)
institution(id, name, country)
account(id, institution_id?, name, type[card|current|savings|deposit|cash|broker|loan],
        currency, owner_member_id?, is_shared, importer_id?, importer_config, archived)
category(id, parent_id?, name, kind[expense|income])
transaction(id, account_id, booked_at, amount, currency,      -- amount signed, in account currency
            description, counterparty, category_id?, member_id?,
            external_id, source[api|file|manual|notification], transfer_group_id?,
            created_by, created_at)
balance_snapshot(id, account_id, as_of, balance)               -- from bank or manual
fx_rate(date, base, quote, rate)
rule(id, match_field, pattern, category_id, priority)          -- auto-categorization
```

Key behaviours:

- **Balance** of an account = latest snapshot + transactions after it (or sum of
  all transactions for cash wallets). Mismatch between computed and
  bank-reported balance is shown as a reconciliation warning.
- **Transfers:** a transaction leaving account A and one arriving in account B
  (same amount ± FX, within a few days) are linked by `transfer_group_id` and
  excluded from income/expense stats. Matching is automatic with manual override.
- **Multi-currency:** store original currency; convert to the base currency on
  read using the rate of the transaction date (for stats) or today's rate (for
  current net worth).
- **Cash:** each person's wallet / "home safe" is an account of type `cash`.
  Withdrawing from an ATM = transfer card → cash wallet; spending cash = expense
  from the wallet.

## 7. Roadmap & estimate (one developer, part-time ~10 h/week)

| Phase | Scope | Estimate |
|-------|-------|----------|
| 0. Decide | List all banks/accounts/currencies; check which have aggregator coverage; pick option A/B/C. | 1–2 evenings |
| 1. Ledger MVP | Accounts, manual transactions, categories, total balance in base currency, FX rates, login for family members. | 1–2 weeks |
| 2. Import | Statement import with parsers for your 2–3 main banks; dedup; transfer matching. | 1–2 weeks (+1–3 days per extra bank) |
| 3. Cash entry | Mobile PWA quick-add + Telegram bot (`"coffee 4.5"` → expense from my wallet). | ~1 week |
| 4. Stats | Monthly dashboard: spending by category/person, income vs. expense, net-worth chart. | ~1 week |
| 5. Automation | Aggregator API sync for supported banks; categorization rules; optional LLM categorization of unknown merchants. | 1–3 weeks |
| 6. Extras | Budgets & alerts, deposits/investments, recurring payments, notification parsing. | ongoing |

**MVP (phases 1–4): ~4–8 weeks part-time.** With Option C, a usable system is
possible in the first week, because phases 1 and 4 come from Firefly III.

## 8. Security & privacy

- Self-host; the database never leaves your server. Encrypted backups (e.g.
  restic to cloud storage) — this is the data you least want to lose or leak.
- **Never store bank login passwords.** Use only read-only aggregator tokens or
  uploaded statements. Store aggregator secrets encrypted / in env secrets.
- HTTPS everywhere, 2FA for the web app, or expose it only over a VPN
  (Tailscale) — simplest and safest for a family.
- Telegram bot: allow-list family members' Telegram IDs; avoid sending full
  balances into chats if phones are shared.

## 9. Feasibility verdict

**Yes, this is very feasible.** Accounts, cash entry, multi-currency totals and
statistics are well-understood problems with existing open-source building
blocks. The only real uncertainty is **automatic bank sync**, which depends on
your banks' country:

- EU/UK/US banks: mostly automatable through an aggregator.
- Banks without open-banking access: fully workable with statement import
  (a few minutes per month) plus notification parsing — less automatic, but
  reliable.
- **Russian banks (our case):** automatic sync is realistic through ZenMoney's
  API. Without it, statement import plus notification forwarding works for
  every bank. The CBR Open API may become an option later.
  Revised MVP estimate for option 2 in §4: **~4–6 weeks part-time**
  (ZenMoney sync worker 1 week, dashboard/stats 1–2 weeks, Telegram bot 1 week,
  1–2 statement importers 1 week).

## 10. Open questions

1. ~~Which country?~~ Russia. Which banks exactly? (Needed to check ZenMoney
   coverage and statement formats.)
2. Which currencies besides RUB (USD/EUR/CNY accounts, cash in foreign currency)?
3. Is a paid ZenMoney subscription acceptable as the sync layer?
4. Phones: Android or iPhone? (Decides how notification forwarding works.)
5. How many family members will enter data, and do they need separate private
   accounts or is everything shared?
6. Are there investments, deposits, loans/mortgage to track, or just cash flow?
7. Preference: adopt an existing tool quickly, or build a custom app (also as a
   learning/hobby project)?
8. Where to host: home server or a Russian VPS?
