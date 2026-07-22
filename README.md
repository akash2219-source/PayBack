# PayBack — Personal Loan & Lending Tracker

PayBack is an offline-first web app for individuals and small lenders to
track loans they've given out — EMI loans and Interest-Only loans — along
with customers, payments, penalties, foreclosures, write-offs, and reports.

It ships as a **single self-contained HTML file**. There is no server, no
build step, no npm install, and no internet connection required to use it.
Every library it needs (React, Tailwind, jsPDF, icons) is bundled directly
inside the file.

---

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [First-Time Setup](#first-time-setup)
- [Usage Guide](#usage-guide)
  - [Adding a Customer](#adding-a-customer)
  - [Creating an EMI Loan](#creating-an-emi-loan)
  - [Creating an Interest-Only Loan](#creating-an-interest-only-loan)
  - [Recording a Payment](#recording-a-payment)
  - [Foreclosing a Loan](#foreclosing-a-loan)
  - [Editing or Deleting](#editing-or-deleting)
  - [Locking the App](#locking-the-app)
  - [Reports & Exports](#reports--exports)
- [Dashboard Reference](#dashboard-reference)
- [Data & Security](#data--security)
- [Known Limitations](#known-limitations)
- [Troubleshooting](#troubleshooting)

---

## Features

- **Dashboard** — live KPI cards: Total Principal Outstanding, Expected
  Income (this month), Collected EMI/Interest (this month), Pending
  EMI/Interest (this month), Foreclosure, Write-off, plus a per-loan
  breakdown table with Status/Type filters.
- **Customers** — add, edit, and delete customers (Aadhaar/PAN stored
  encrypted, phone number validated and mandatory, optional "Browse
  Contact" picker on supported Android browsers).
- **Loans** — two loan types per customer:
  - **EMI loans**: Reducing or Flat interest method, Monthly/Yearly rate
    entry, 3/6/12/24/48/60-month tenure, Monthly or Weekly collection
    frequency, full amortization schedule.
  - **Interest-Only loans**: recurring interest-only payments against the
    outstanding principal, Monthly or Weekly frequency.
- **Payments** — record full or partial payments, with an automatic
  waterfall allocation (Penalty → Interest → Principal → Excess), comma-
  formatted currency input, and a live Pending calculation.
- **Foreclosure** — pay off a loan's full remaining balance in one
  transaction; closes the loan and is tracked separately on the Dashboard.
- **Write-off** — mark an unrecoverable loan's principal as written off.
- **PIN Lock & Encryption** — optional 6-digit PIN, AES-256-GCM encrypted
  local storage, 10-minute auto-lock with a 60-second warning, and a
  guided in-app flow to set a PIN if you lock the app before one exists.
- **SMS Notification** — one-tap SMS reminder per loan with the current
  EMI/interest amount pre-filled.
- **Reports** — per-customer, per-loan CSV export and a PDF statement
  (EMI loans) including the full amortization schedule.
- **Factory Reset** — PIN-gated full data wipe, for starting over cleanly.

---

## Installation

Since PayBack is a single HTML file, there's no traditional "install."
Pick whichever of these fits how you want to use it:

### Option A — Just open it (simplest)
1. Download `PayBack_-_Loan_Management_App.html`.
2. Double-click it, or drag it into any modern browser (Chrome, Edge,
   Safari, Firefox).
3. That's it — the app runs entirely in that browser tab.

> Your data is tied to **that specific browser** on **that specific
> device**. Opening the file in a different browser, or in Incognito/
> Private mode, starts a fresh, empty vault.

### Option B — Keep it handy on your device
- **Desktop**: Save the file somewhere permanent (e.g. Documents), then
  bookmark the opened tab, or create a desktop shortcut to the file.
- **Android**: Open the file in Chrome, then use the browser menu →
  "Add to Home screen" for an app-like icon.
- **iOS**: Open the file in Safari, then use the Share sheet → "Add to
  Home Screen."

### Option C — Serve it over a local web server (optional)
If your browser restricts certain features on `file://` pages, serve the
folder locally instead:

```bash
# Python 3
python3 -m http.server 8000

# Node.js (via npx)
npx serve .
```

Then open `http://localhost:8000/PayBack_-_Loan_Management_App.html` (or
whatever port/tool you used).

**Do not** re-host the file on a public server without adding your own
authentication in front of it — the app protects data with a local PIN,
not a login system, and is designed for single-device personal use.

---

## First-Time Setup

On first launch you'll walk through a 3-step onboarding flow:

1. **Business Setup** — enter your Lender/Business Name and Phone Number
   (required), and Address (optional). Tap **Register** to continue.
2. **Security & PIN Lock** — create a 6-digit PIN, or tap **Skip for Now**
   to use the app without one. You can add, change, or remove a PIN later
   any time from **Settings → Security**.
   - *With a PIN*: your data is zero-knowledge encrypted using that PIN.
     There is no recovery if you forget it.
   - *Without a PIN*: data is still encrypted at rest with a built-in key,
     but anyone with access to the device can open the app.
3. **Dashboard** — setup is complete and you land on the main Dashboard.

---

## Usage Guide

### Adding a Customer
Go to **Customers → Add Customer**. Enter Full Name, Phone Number
(mandatory, 10-digit validated), ID Details (Aadhaar/PAN — stored
encrypted), and optionally use the contact picker icon to pull details
from your device's contacts (Chrome on Android only).

### Creating an EMI Loan
From a customer's profile, tap **New Loan** and choose **EMI**:
1. **Core Terms** — Principal Amount, Interest Rate (with a Monthly/Yearly
   toggle), EMI Collection Frequency (Monthly or Weekly), Tenure.
2. **Engine Rules** — Reducing or Flat interest method, Start Date, grace
   period/penalty settings.
3. **Schedule Preview** — review the generated amortization schedule
   before confirming.
4. **Confirm** — creates the loan and its first due date.

### Creating an Interest-Only Loan
Same flow, but choose **Interest-Only**. There's no fixed tenure or
amortization schedule — you set the Principal, Interest Rate, and
Interest Collection Frequency (Monthly or Weekly), and the loan continues
until you foreclose, write it off, or close it manually.

### Recording a Payment
Open the loan, tap **Record Payment**, enter the amount (comma-formatted
as you type) and payment mode. Partial payments are supported — the app
tracks the shortfall and reflects it as Pending on the Dashboard and in
the loan's Transactions table.

### Foreclosing a Loan
From the loan's detail view, choose **Foreclose** to pay off the entire
remaining balance in one transaction. This closes the loan and records
the payoff amount separately under the Dashboard's Foreclosure card for
that month.

### Editing or Deleting
- **Customers**: edit or delete from the customer profile (deleting a
  customer cascades to their loans and history).
- **Loans**: delete an individual loan via the accordion header's delete
  icon, or use "Select Loans" in the customer profile to bulk-delete
  specific loans without touching the customer record.
- **Transactions**: edit or void individual transactions; editing
  recalculates partial-payment and shortfall figures automatically.

### Locking the App
Tap the lock icon in the header any time to lock the app immediately.
- **If a PIN is already set**: you're taken straight to the PIN-entry
  screen.
- **If no PIN is set yet**: you're prompted to set one first (with a
  Cancel option to back out and keep using the app unlocked). Once a PIN
  is set, the app locks immediately using that new PIN.

The app also auto-locks after 10 minutes of inactivity (with a warning at
the 9-minute mark) whenever a PIN is configured.

### Reports & Exports
Go to **Report Download → Customer Statements**. Pick a customer, then a
specific loan, to download:
- A **CSV** of that loan's full transaction history.
- A **PDF statement** (EMI loans) including the complete amortization
  schedule, available as soon as the loan is created — even before any
  payments are made.

---

## Dashboard Reference

All figures except Total Principal Outstanding are scoped to the
**current calendar month** (there is no date-range picker):

| Card | Meaning |
|---|---|
| **Total Principal Outstanding** | Live snapshot of outstanding principal across all ACTIVE/OVERDUE/PARTIALLY_SETTLED loans, as of today. |
| **Expected Income** | Sum of each active loan's fixed scheduled EMI/interest amount for the current month — Monthly loans count once, Weekly loans count as 4× the per-cycle amount. This is independent of whether the installment has already been paid. |
| **Collected EMI/Interest** | Total payment amount actually collected this month. |
| **Pending EMI/Interest** | Expected Income minus Collected, floored at zero. |
| **Foreclosure** | Total foreclosure payoff amount collected this month. |
| **Write-off** | Total principal written off this month. |

A loan that is foreclosed or written off no longer contributes to Expected
Income for that month (only loans currently ACTIVE, OVERDUE, or
PARTIALLY_SETTLED are counted).

---

## Data & Security

- **Storage**: all data lives in your browser's IndexedDB (with a
  localStorage fallback), scoped to the specific browser + device you
  opened the file in. Nothing is sent anywhere over a network.
- **Encryption**: the entire data vault is encrypted at rest with
  AES-256-GCM. The encryption key is derived via PBKDF2-SHA256 (310,000
  iterations) from either your PIN, or — if you skip PIN setup — a
  built-in passphrase baked into the app (which allows automatic
  unlocking, but means the data is only as protected as physical access
  to the device).
- **PIN recovery**: if you set a PIN and forget it, there is **no
  recovery path** by design (that's what "zero-knowledge" means here) —
  your only option is Factory Reset, which erases all data.
- **Factory Reset**: available in Settings, gated behind your current PIN
  (or a typed "DELETE" confirmation if no PIN is set), and wipes the
  vault completely.

---

## Known Limitations

- **Single device/browser only** — there is no sync, backup, or
  multi-device access built in. Use the CSV/PDF exports as your backup
  strategy.
- **No login system / multi-user support** — this is a personal, single-
  operator tool, not a multi-tenant SaaS product.
- **Contact picker** is Chrome-on-Android only (a browser API
  limitation, not a bug).
- **No cloud backup** — clearing your browser's site data, or switching
  browsers/devices, means starting over unless you've exported reports.

---

## Troubleshooting

- **"I forgot my PIN"** — there's no recovery; you'll need to Factory
  Reset (Settings) and start fresh.
- **Blank screen on load** — try a hard refresh; if it persists, check
  the browser console for errors and confirm you're using a modern,
  up-to-date browser (Chrome, Edge, Safari, or Firefox).
- **Data seems to have disappeared** — you're likely opening the file in
  a different browser, a different device, or an Incognito/Private
  window than the one you originally set it up in. Data is local to the
  exact browser profile it was created in.
- **PDF export shows "Rs." instead of ₹** — this is intentional: the PDF
  engine's built-in font can't render the ₹ glyph reliably, so exports
  use "Rs." for safety while the live app UI continues to show ₹
  everywhere.
