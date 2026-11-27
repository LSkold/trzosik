# Trzosik

> **Trzosik – pilnuje Waszej kasy.**
> A private family budget review app. It reads our bank statements and shows facts that help us save: where the money goes, what we pay for regularly, and whether our savings could earn more.

This file is the single source of truth for the project. Claude Code reads it automatically, so keep it up to date whenever a decision changes.

---

## 1. Context

- **Users:** Łukasz and his wife only, both on iPhone. The app is private and never public.
- **Goal:** built for us first. It goes further (other families, modest passive income) only if it proves useful in our own home.
- **Approach:** one household budget, not a "fair split" between partners. We're looking for savings, not settling who pays what.
- **Tone:** computed facts only, e.g. "Zakupy online +12% vs październik". No AI-written commentary or advice in the UI.
- **Language:** UI in Polish. Code, commits, comments and docs in English.

## 2. Status

- [x] Idea, name, logo, palette, first mockups
- [ ] **Milestone 1: skeleton** ← next
- [ ] Milestones 2–7 (see §12)

---

## 3. Tech stack

| Layer | Choice |
|---|---|
| Backend | Java 21, Spring Boot 3.x, modular monolith (package by feature) |
| Database | PostgreSQL 16, Flyway migrations |
| Frontend | Angular (latest stable), standalone components and signals, PWA via `ng add @angular/pwa` |
| PDF parsing | Apache PDFBox, run locally on the server |
| Crypto | Google Tink (AEAD envelope encryption, HMAC) |
| Auth | Reused from Liberio (Łukasz's other project), plus 2FA. **Ask Łukasz for the Liberio auth module before you implement login.** |
| Tests | JUnit 5, Testcontainers (Postgres), Angular unit tests, Playwright for a few e2e checks |
| Runtime | Docker Compose on one small shared server (Hetzner Cloud, EU), one compose project per app |
| CI/CD | GitHub Actions; images in GHCR; deploy over Tailscale SSH on merge to `main` |

**On iPhone:** open the app in Safari, then "Add to Home Screen". It runs full screen with its own icon. Web push works for home-screen apps on iOS 16.4+, once the user allows it.
**Later, only if needed:** wrap the same Angular app with Capacitor and ship through TestFlight. That needs the $99/year Apple Developer account and a Mac.

### Proposed repo layout

```
trzosik/
├── CLAUDE.md
├── backend/            Spring Boot app
│   └── src/main/java/app/trzosik/
│       ├── household/      households, members, accounts
│       ├── importing/      fingerprints, recipes, CSV/PDF parsers
│       ├── transaction/    transactions, dedup, internal transfers
│       ├── category/       merchants, categories, rules
│       ├── recurring/      subscriptions, instalments, mortgage, yearly bills
│       ├── savings/        balances, real rate, offers check
│       ├── notification/   web push, calendar (.ics)
│       └── shared/         crypto, money, time, errors
├── frontend/           Angular PWA
├── deploy/             compose.yml, Tailscale serve config, nginx config, backup scripts, cloud-init
├── brand/              logo SVGs (mark, mark-bold, app icon, lockup)
└── .github/workflows/  ci.yml, deploy.yml
```

---

## 4. Product: MVP phases

The order goes from the big picture to growing what's left over:

1. **Spending (Wydatki):** where the money goes, the biggest and smallest categories, and where we spend more than we think.
2. **Subscriptions (Subskrypcje):** everything we pay for regularly, with a calendar event before each renewal. Instalments and the mortgage are covered here too.
3. **Savings (Oszczędności):** whether spare money could earn more.

Bottom nav, three tabs: **Wydatki · Subskrypcje · Oszczędności**.

### 4.1 Spending screen

- **Hero number:** what's left this month, e.g. `+1 000 zł` (green `--positive` when positive). It's not spending alone.
- Under it, smaller: income (*Wpływy*) and spending (*Wydatki*) for the month, plus a thin bar showing the % of income used ("Wydane 83% wpływów").
- One computed-fact line at most, e.g. "Zakupy online +12% względem października". Use `--warning` for an increase.
- **Na co poszło:** top categories as simple horizontal bars. The biggest is navy; the rest are muted blue.
- A subscriptions teaser card: monthly total, one fact, and a CTA to the Subscriptions tab.
- A month switcher pill in the header ("Listopad").
- Filter: whole household, or by person.
- Transfers between our own accounts **never** count as income or spending.

### 4.2 Subscriptions screen

- **Detection:** same counterparty, similar amount, regular cycle (monthly or yearly). The user confirms each one once, and it's remembered.
- **Top:** total monthly subscriptions, plus the yearly total.
- **Main list:** monthly subscriptions sorted by cost. Each row has name, amount, next renewal date and an "add to calendar" button (.ics download).
- **"Roczne" (yearly) section** below: real price and renewal date, e.g. "Ubezpieczenie, 1 200 zł, odnawia się w marcu". The monthly equivalent shows only in the details.
- **Duplicate and creeping charges** are flagged, e.g. "Apple pobiera 4 opłaty co miesiąc, razem 244 zł, w lutym było 67 zł".
- **Instalments (raty 0%)** have their own section, because they can't be cancelled and they end:
  - progress ("7 z 10 spłacone" with a bar), last payment date and amount remaining;
  - a computed fact: "W marcu Twoje miesięczne koszty spadną o 150 zł";
  - an instalment that disappeared is flagged ("Rata 224,92 zł chyba się skończyła").
- **Mortgage (kredyt hipoteczny)** has a separate section and isn't treated as a subscription. It shows the next due date, with a reminder before it.

### 4.3 Savings screen

- **Top:** what our savings actually earned. The real rate is worked out from interest payments in the statements and month-end balances.
- A comparison with today's rates, shown as facts, not advice.
- Promo alerts: "Twoja promocja kończy się 24-go".
- Safety cushion: how many months of spending our savings cover.
- **Weekly offers check:** our own banks first (Millennium, PKO BP, Pekao): official savings and deposit offers. Only if none of them is better, other options (e.g. government bonds).
  - Each rate is stored with its date, source URL and conditions (new money only, end date, limits).
  - **AI can make mistakes.** Code checks every result:
    - the rate is in a plausible range (e.g. 0–10%);
    - a big change vs last week is flagged as `SUSPECT`;
    - the source link is always shown;
    - the user verifies before acting.
  - Notification style: "Jest okazja, żeby zarobić więcej", with a link to check it.

### 4.4 Other detectors (from the first prototype; later)

Bank fees (e.g. SMS notifications), deferred payments (PayU / PayPo), irregular yearly payments (insurance, electricity settlements), and a calendar of big payments over the next 12 months with a suggested monthly amount to set aside. A score out of 100 with a "potential" score is a maybe for later; it isn't part of the MVP.

---

## 5. Import

**Accounts:**
- Millennium: CSV.
- PKO BP: monthly PDF.
- Pekao SA: the wife's account; format still to check.

**User flow:** choose a file → see a preview of a few rows → confirm.

**Pipeline:**

1. Compute the file **fingerprint**: for CSV, the normalized header row; for PDF, the bank name plus a layout signature from PDFBox text.
2. **Known fingerprint:** apply the saved **recipe** with plain Java.
   - CSV recipe: a column mapping (date, amount, counterparty, description, account), date format, decimal separator, encoding, rows to skip.
   - PDF recipe: a regex over the text that PDFBox extracts locally.
3. **New fingerprint** (Milestone 6): mask a small sample (names → `<NAME>`, account numbers → `<ACC>`), ask the AI **only for a recipe**, then test it on the whole file. The parsed totals must match the change in balance. Only then save it.
4. **The AI never sees a full statement.** It sees only one masked sample per bank format, once.
5. After parsing, **delete the uploaded file**. Keep only the transactions plus the file's SHA-256 (to block a second import of the same file).
6. **Dedup:** a stable key per row, `HMAC(account_id, booking_date, amount_minor, raw_key, ordinal_in_day)`. Re-importing an overlapping period must not duplicate rows.

**Known Millennium CSV issues** (the parser must handle them, with tests):
- Salary rows are shifted by one column.
- One export file can hold **two accounts**. Split by the account column.

**Test fixtures:** synthetic only, hand-written to mimic the real format. **Never commit real statements**; see §8.

---

## 6. Household: one budget, three accounts

- One `household`; Łukasz and his wife are its members.
- Accounts belong to the household, and each is tagged with its owner (view together or filter by person).
- All summaries are computed per household.
- **Internal transfers are excluded from spending and income.** Detect them in this order:
  1. **Strongest:** the counterparty account number matches one of the household's accounts. Compare HMACs only; account numbers are never stored in plain text.
  2. **Fallback:** same amount, opposite sign, within 2–3 days, both accounts in the household.
  3. **Unclear cases:** the app asks "Czy to przelew między Wami?", and one tap confirms.
  4. A confirmed case becomes a saved **rule** (e.g. title "na życie" between these two accounts = internal), applied automatically from then on.

---

## 7. Data model (draft)

All money is stored as `amount_minor BIGINT` (grosze) plus a `currency CHAR(3)`. Never use floating point for money.

| Table | Key columns |
|---|---|
| `household` | id, name, created_at |
| `app_user` | id, household_id, display_name, auth fields (from Liberio) |
| `account` | id, household_id, owner_user_id, bank, label, type (CHECKING / SAVINGS / CREDIT), currency, `number_hmac` |
| `import_recipe` | id, fingerprint, kind (CSV / PDF), definition JSONB, source (HAND / AI), validated_at |
| `import_batch` | id, account_id, recipe_id, file_sha256, period_from, period_to, row_count, status, created_at |
| `transaction` | id, account_id, import_batch_id, booking_date, amount_minor, currency, merchant_id, category_id, `counterparty_key` (HMAC), `raw_key` (HMAC of the raw description), `description_enc` (BYTEA, optional, encrypted), is_internal, transfer_pair_id, dedup_key (unique) |
| `merchant` | id, display_name, normalized_name, default_category_id, `is_sensitive` |
| `category` | id, name_pl, parent_id, sort |
| `rule` | id, household_id, kind, match JSONB, action JSONB, created_from (USER_CONFIRMED / SYSTEM) |
| `recurring_item` | id, household_id, merchant_id, kind (SUBSCRIPTION / YEARLY / INSTALMENT / MORTGAGE), amount_minor, cycle, next_date, confirmed, paid_count, total_count |
| `account_balance` | account_id, month, balance_minor (month-end) |
| `monthly_total` | household_id, month, category_id, amount_minor (derived; rebuildable) |
| `savings_offer` | id, bank, product, rate_bps, conditions JSONB, source_url, checked_at, status (OK / SUSPECT) |
| `finding` | id, household_id, kind, payload JSONB, created_at, dismissed_at |
| `push_subscription` | id, user_id, endpoint, keys_enc |

**Principles:**
- Transactions are stored **without raw descriptions** by default. Descriptions map to a known `merchant` via `raw_key` and rules.
- Merchants with `is_sensitive = true` (e.g. pharmacy, doctor) are shown only as their category, never by name, in notifications.
- `monthly_total` and `finding` are derived and can always be rebuilt from transactions.

---

## 8. Security (from day one)

These are hard rules. Claude Code must follow them in every PR.

**Network**
- No public ports. The server, both iPhones and the iPad are on **Tailscale**. HTTPS uses Tailscale's certificate on the private `*.ts.net` name (`tailscale serve`), which a PWA install needs.
- Only **Tailscale SSH**. OpenSSH is not exposed; the Hetzner Cloud Firewall has **no inbound rules** (Tailscale works over outbound connections only).
- Tailscale ACLs decide who reaches which app: family devices reach `trzosik` and the game, `tag:ci` reaches only SSH on `tag:server`. The iPad uses Łukasz's login for now; limiting it to the game later needs a separate login for the son.

**Data**
- Hosting in the EU only (Hetzner Falkenstein, Nuremberg or Helsinki).
- **Field-level encryption** with Tink AEAD, envelope style: one data key (DEK) per household, wrapped by a key-encryption key (KEK) kept outside the database (Docker secret / env file on the server, readable by root only). A copy of the KEK lives in Łukasz's password manager; without it the backups cannot be decrypted. It covers counterparty names, descriptions and push keys.
- Account numbers and counterparty identifiers are **HMACs** (Tink, separate key), never plain text.
- **Delete uploaded files** right after parsing.
- **No transaction data in logs.** No amounts, names, descriptions or account numbers, at any log level. Log IDs and counts only. Add a test or ArchUnit rule if practical.
- Backups: a nightly `pg_dump`, encrypted (age or restic) **before** it leaves the VM, stored **with a different provider than the server** (e.g. Backblaze B2 EU). Test a restore regularly.

**Auth**
- Liberio login plus 2FA (TOTP). Secure, HttpOnly, SameSite=Strict cookies; CSRF on.
- Every query is scoped by `household_id`, checked in the service layer, not just the controller.

**AI**
- Only masked samples (import recipes) and public bank offer pages (savings check). Never real transactions.
- Before using an AI provider, check its API terms on data retention and training.

**Repo**
- **Never commit real statements, exports or personal data.** `.gitignore` must cover `*.csv`, `*.pdf` and `*.xlsx` outside `**/test/resources/fixtures/`, plus `private/` and `.env*`.
- Secrets only in GitHub Actions secrets or on the VM, never in code.
- Dependabot or Renovate on; CI fails on known critical CVEs.

**Before opening to other families:** a GDPR review and a proper security audit.

---

## 9. Hosting and deploy

- **Server:** one small paid **Hetzner Cloud** VM in an EU location, shared by trzosik and Łukasz's other private apps (e.g. a game for his son on the iPad). Start with the smallest x86 shared-vCPU plan with 2 vCPU / 4 GB RAM / 40 GB (CX23 at the time of writing, roughly €4–6/month incl. IPv4; check the current price). Ubuntu 24.04 LTS. Upgrade in place (rescale) if RAM gets tight.
  - x86, not Arm: CI builds on standard GitHub runners without emulation.
  - Set up once with `deploy/cloud-init.yaml` (Docker, Tailscale with Tailscale SSH, unattended upgrades, no OpenSSH on the internet). Rebuilding the server = new VM + same cloud-init + restore backup.
  - Hetzner Cloud Firewall: no inbound rules at all.
- **Access model:** private to our tailnet, not "home Wi-Fi only". The phones keep working outside home; nothing is reachable from the internet. Devices need the Tailscale app.
- **Multi-app layout on the server:**

  ```
  /srv/apps/
  ├── trzosik/   compose.yml, .env (secrets, root-only)
  └── <game>/    compose.yml, .env
  ```

  - Each app is its own compose project with its own Docker network and its own Postgres (if it needs one). Apps never share a database.
  - Each app has a **Tailscale sidecar container** (`tailscale/tailscale`, `TS_SERVE_CONFIG`) that joins the tailnet as its own node, e.g. `trzosik.<tailnet>.ts.net`, and serves HTTPS to the app's `web` container. So every app gets its own origin (clean PWA scope and storage), its own certificate and its own ACL tag, and no shared reverse proxy is needed.
  - The host itself is a separate node `tag:server`, used only for SSH/deploy.
- **trzosik compose services:** `ts` (Tailscale sidecar), `web` (nginx serving the Angular build and proxying `/api`), `backend`, `postgres`.
- **Deploy:** on every merge to `main`, GitHub Actions builds the images, pushes them to GHCR, joins the tailnet with `tailscale/github-action` (OAuth client, ephemeral, `tag:ci`), and over Tailscale SSH runs `docker login ghcr.io` with the job's short-lived `GITHUB_TOKEN`, then `docker compose pull && docker compose up -d` in `/srv/apps/trzosik`. No long-lived registry token on the server.
- **Tailscale ACL (sketch):** `group:family` (Łukasz, wife) → `tag:trzosik:443`; `tag:ci` → `tag:server` SSH only; Łukasz → `tag:server` SSH.

## 10. Dev workflow (phone-first)

Goal: after the one-time setup, everything happens on the phone.

1. Plan and ask for work in the Claude app (project chat / Claude Code on the web), on this GitHub repo.
2. Claude Code opens a small PR. CI (`ci.yml`) runs backend and frontend tests on every PR.
3. Review in the GitHub mobile app (diff, CI status, comments) and merge there.
4. Merge to `main` deploys automatically (`deploy.yml`). A failed deploy leaves the previous containers running.
5. Test the PWA on the iPhone over Tailscale. Each PR description says what to tap to check it.
6. Dependency updates come as Renovate/Dependabot PRs, merged from the phone like any other.

No local dev environment is needed. If something must be run on the server, use Tailscale SSH from the phone (Termius, or the Tailscale app's SSH) only as a last resort; normal changes go through a PR.

The one-time setup (accounts, server, Tailscale, secrets) is listed in `deploy/SETUP.md`. Most of it works in a phone browser; a PC is just more comfortable for the Tailscale ACL editor.

**Rules for Claude Code in this repo:**
- Small PRs: one milestone step per PR, with tests and a short description of how to check it on the phone.
- Ask before adding a new dependency or changing a decision in this file. If a decision changes, update this file in the same PR.
- UI strings in Polish. Money formatted `1 234,56 zł` (`pl-PL`), with tabular numerals.
- Fixtures and seeds are synthetic and must look obviously fake.
- Migrations: Flyway, forward-only, never edit an applied migration.

---

## 11. Design

Minimalist: a white base with navy ink, light blue surfaces, and pink accents. Rounded, friendly, calm.

### Tokens

```css
:root {
  --ink:          #072D59; /* navy: text, logo, main bars, charts */
  --ink-hover:    #041B36;
  --text-2:       #3E5470; /* secondary text */
  --muted:        #5B6B80; /* labels, captions, inactive tabs */
  --bg:           #FFFFFF;
  --surface:      #EEF8FD; /* cards */
  --tint:         #DBF2FC; /* borders, tracks, pills */
  --bar-muted:    #8FB3D1; /* non-top category bars */
  --accent:       #FD6E88; /* primary CTA background (text on it is navy) */
  --attention:    #E84D6E; /* icons, active tab icon, new items */
  --attention-2:  #D63A5E; /* big upcoming payments */
  --accent-soft:  #FFE3E9; /* pink badge background */
  --positive:     #1F8A5B; /* "what's left" when positive */
  --warning:      #B4690E; /* increases vs last month */
}
```

**Meaning of the colors:**
- **Pink means "act" or "look here"**, never "loss" or "bad".
- Increases use `--warning`.
- Green is only for the positive "what's left" number.

### Type

- **Nunito**, weights 400, 600, 700 and 800. Use `font-variant-numeric: tabular-nums` for all amounts.
- Hero number: 46px / 800. Section headings: 16px / 800. Body: 14–15px. Captions: 12–13px.

### Shape and touch

- **Corners:** cards 20–24px, small tiles 14px, buttons and pills 999px.
- **Touch targets:** at least 44px high; tab items 48px.
- **Accessibility:** a visible `:focus-visible` outline (2px navy), and decorative icons `aria-hidden`.

### Logo

A drawstring money pouch with a coin bearing a "T". The source files are in `brand/`:
- `trzosik-mark.svg`
- `trzosik-mark-bold.svg`
- `trzosik-app-icon.svg`: white pouch on a navy rounded square, used for the PWA icons and `apple-touch-icon`.
- `trzosik-lockup.svg`: the mark plus "trzosik" in Nunito 800. Convert the text to outlines before using it as an asset.

Mark (viewBox `0 0 240 250`) for inline use:

```svg
<svg viewBox="0 0 240 250" aria-hidden="true">
  <path fill="#072D59" d="M66 18C79 14 94 22 120 21C146 22 161 14 174 18C183 21 183 31 178 39C173 47 164 55 155 64C176 76 196 94 207 116C224 149 220 181 198 204C180 222 153 231 120 231C87 231 60 222 42 204C20 181 16 149 33 116C44 94 64 76 85 64C76 55 67 47 62 39C57 31 57 21 66 18Z"/>
  <path d="M77 66C96 64 144 64 163 66" fill="none" stroke="#FFFFFF" stroke-linecap="round" stroke-width="9"/>
  <circle cx="120" cy="151" r="52.5" fill="#FFFFFF"/>
  <circle cx="120" cy="151" r="38" fill="none" stroke="#072D59" stroke-width="9"/>
  <path d="M103 135H137M120 135V169" fill="none" stroke="#072D59" stroke-linecap="round" stroke-width="9"/>
</svg>
```

In the header: the mark at 30px plus "trzosik" in lowercase, Nunito 800, 21px, letter-spacing −0.5px.

### Name

- **Trzosik** is the working name. A *trzos* is an old Polish money pouch worn on the belt.
- It always appears with the tagline "Trzosik – pilnuje Waszej kasy".
- The final market name is decided before opening to other families. The fallback is **Sakiewka**.

---

## 12. Build order

Each milestone should be usable on the phone before the next one starts.

1. **Skeleton:**
   - Spring Boot + Angular PWA, Docker Compose and Flyway.
   - Liberio login with 2FA.
   - Running on the Hetzner server behind Tailscale (sidecar), with CI and auto-deploy on merge.
   - "Add to Home Screen" works on iPhone.
2. **Millennium CSV import** with a hand-written mapping (no AI yet). Includes:
   - preview and confirm;
   - dedup;
   - file deletion;
   - tests for the shifted salary rows and the two-account file.
3. **Spending screen:** the first usable version (§4.1).
4. **Household:**
   - PKO BP PDF and Pekao import;
   - owner tags;
   - internal transfer detection and rules (§6).
5. **Subscriptions and instalments:**
   - detection plus confirmation;
   - yearly section;
   - mortgage;
   - .ics calendar events;
   - web push reminders.
6. **AI recipes for import:** masked sample → recipe → validation against the balance change. This replaces the hand-written mappings.
7. **Savings:**
   - month-end balances and the real rate;
   - the cushion;
   - promo alerts;
   - the weekly offers check with sanity checks.

---

## 13. Open questions

- Final market name: Trzosik or Sakiewka? (Decide before opening to others.)
- Domain?
- Maven or Gradle for the backend?
- Off-site backup target: Backblaze B2 (EU) or another provider outside Hetzner?
- Pekao export format: CSV or PDF?
- Which AI provider for import recipes and the offers check, and do its API terms fit?
- Bank data later: stay with CSV/PDF, or use an aggregator (Synci, Kontomatik, Salt Edge…)?
