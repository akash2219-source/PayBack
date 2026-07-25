# PayBack — UI update notes

`PayBack - Loan Management App.html` is a single self-contained file (React 18 UMD +
Babel standalone + Tailwind v4 browser JIT + Lucide + jsPDF). These notes cover three
consecutive update passes applied to it.

**No business logic was changed in any pass.** Interest/EMI/penalty math, the payment
waterfall, encryption, migration and all stored data are untouched. The only new
computation anywhere is the PDF's totals row, which sums the existing schedule. Pass 3
did restructure navigation (see 2.3) but not what any figure means.

---

## 1. Installation

No build step, no `npm install`, no environment variables.

1. Save `PayBack - Loan Management App.html` anywhere on disk.
2. Open it in a browser — double-click, or navigate to the `file://` path. It also works
   uploaded to any static host (GitHub Pages, Netlify, an S3 bucket).
3. First run walks through onboarding: Business Setup → Security & PIN Lock → Dashboard.
4. **Upgrading an existing vault:** open the new file in the *same browser profile* as
   the old one — the vault lives in that profile's IndexedDB, and `migrateVault()` runs
   automatically on unlock. Opening it in a different browser or a private window shows
   an empty vault; that is expected, nothing has been lost.

**Rolling back:** keep your previous copy of the file and open it in the same browser
profile. The vault is untouched by either pass, so no data migration is involved in
either direction.

---

## 2. Pass 3 — dropdowns, customer accordion, collapsed-by-default

### 2.1 Dropdowns opened white

The app never declared `color-scheme` anywhere. A native `<select>` popup is painted
by the browser/OS, not by your CSS — your styling only reaches the closed control, so
the open list came up in the light scheme regardless. Fixed with `color-scheme: dark`
on `:root` plus explicit `option`/`optgroup` colours for Firefox, which ignores the
inherited scheme for option backgrounds.

Six selects were affected: the two loan filters, payment mode, rate basis, EMI
frequency and tenure. The same declaration also darkens native scrollbars and the date
picker, which is consistent with the rest of the UI.

### 2.2 The ⋮ menu is gone, replaced by Delete

The three-dot button was a decoy — its `onClick` called `goToProfile()`, exactly what
clicking the row already did. There was no menu behind it, so nothing was lost by
removing it. In its place is a trash icon that opens the existing `DeleteCustomerModal`
directly from the list, with `stopPropagation()` so it doesn't also toggle the row.

### 2.3 Customer profile folded into the list

**This is the structural change in this pass.** Customers and the customer profile were
two separate routes. The profile is now an inline accordion body inside the customer
row:

- `CustomerProfilePage` → `CustomerProfileBody`, with the "Back to Customers" button and
  the duplicated name/avatar header removed. The row header already carries those, so
  the body starts at the stats + actions bar (Loans, Phone, Edit / Delete / New Loan).
- The row is now a keyboard-accessible toggle (`role="button"`, Enter/Space, `aria-expanded`)
  with a rotating chevron.
- Collapsing returns you to the plain list with the Add Customer button — no navigation.
- The ID stat became **Phone**, since the row beneath the name already shows the customer
  ID and repeating it inside the expanded body was redundant.

**Deep links still work and needed no edits at their call sites.** Six places route into
the profile: two notification paths, the URL hash router, global search, the dashboard
loan table, and the list row. The `customerProfile` route is kept as an alias that renders
`CustomersPage` with `initialExpandId`, so every one of those still lands on the right
customer — now by expanding the row instead of navigating. Arriving via a loan deep link
still opens that specific loan inside the expanded customer.

### 2.4 Modals portalled to `<body>` (regression fix)

Folding the profile into the card broke every modal opened from inside it — New Loan and
Edit Customer rendered clipped and transparent, with the row content showing through.

Cause: `.glass` carries `backdrop-filter: blur(12px)`, and per spec **an element with a
backdrop-filter becomes the containing block for `position: fixed` descendants**. The
modals previously sat in a plain top-level div, so `fixed inset-0` resolved against the
viewport. Once they moved inside the `.glass` customer card they resolved against the
*card*, and the card's `overflow-hidden` clipped what was left.

Fix: `Modal` now renders through `ReactDOM.createPortal(..., document.body)`. That makes
it independent of where it is mounted, so this class of bug cannot recur no matter how
deeply a future modal is nested. Toasts and the global search dropdown render at app
level and were never affected.

Worth knowing: React portals propagate events through the **React** tree, not the DOM
tree, so modal clicks still bubble through the components that rendered them. The
expanded body's click handler stops that propagation, so interacting with a modal can't
collapse the row behind it.

### 2.5 Everything collapsed by default

`Collapsible` now defaults to `defaultOpen={false}`, and the two call sites that passed
`true` (dashboard Summary, Report-page customer cards) were changed to `false`. The loan
list no longer auto-opens the first loan — `expanded` starts at `null` rather than
`loans[0].id`. The one deliberate exception is a deep link carrying `openLoanId`, which
still opens its target; that is the point of the link.

---

## 3. Pass 2 — customer page, loan row, schedule tab, currency input, PDF

### 2.1 Customer detail header

Rebuilt to match the supplied target layout. The stat row now uses rounded icon tiles
with a label/value pair rather than a run-on line, which also fixes the `1Loans`
spacing bug (the count and the word had no separator):

- Loans tile — teal-tinted, count rendered in teal mono
- ID tile — neutral, ID rendered in mono
- Masked ID details, when present, follow behind a divider
- Action buttons right-aligned at a uniform 44px height, matching the control height
  set in pass 1. "New Loan" uses `circle-plus` to match the target.

### 2.2 Loan row shows principal only

The loan row previously rendered the outstanding balance with the original principal
beneath it:

```
₹1,00,000
of ₹1,00,000
```

The `of …` sub-line is gone and the remaining figure is now `loan.principal`.

> **Worth a second look.** This changes *which* number is displayed, not just deletes a
> line. The row previously led with `currentPrincipalBalance` (outstanding) and now
> shows `principal` (original). On a loan with repayments recorded, these differ — the
> row will no longer reflect what is still owed. In the screenshot they happened to be
> identical because no payments existed yet. If you wanted the outstanding figure kept
> and only the second line dropped, it's a one-word edit.

### 2.3 "Amortization Schedule" → "Schedule"

Renamed on the loan accordion tab. The PDF's heading is now "Loan Statement" (see 2.5),
so "amortization" no longer appears in any user-facing label.

### 2.4 Live comma grouping while typing

`CurrencyInput` previously showed a grouped value only when the field was *not* focused
— so commas vanished the moment you started typing. It now formats on every keystroke
using Indian grouping:

```
1 · 12 · 123 · 1,234 · 12,345 · 1,23,456 · 12,34,567 · 1,00,00,000
```

Two details worth knowing:

- **Caret preservation.** Reformatting on each keystroke would normally throw the cursor
  to the end of the field, making mid-string edits impossible. The component records how
  many significant characters sat left of the caret before the edit and re-derives the
  equivalent offset in the regrouped string. Inserting a digit at the front of
  `1,00,000` leaves the caret just after the new digit, not at the end.
- **Decimals survive.** Grouping applies to the integer part only, so a half-typed
  `1234.` or a trailing `1,234.50` round-trips instead of being eaten by
  `toLocaleString`. Leading zeros collapse (`007` → `7`), which is the desirable
  behaviour mid-typing.

The stored value is unchanged — `sanitizeCurrencyRaw` still strips everything but digits
and a single decimal point before it reaches state, so no commas enter the vault.

### 2.5 PDF statement redesign

Rebuilt from a plain heading-plus-table into a structured document:

- **Header band** — navy block with a teal rule, lender name, "Loan Statement",
  generation date and loan name. Repeats on every page.
- **Two info panels** — borrower (name, ID, phone) and loan (type, interest method,
  status).
- **Four summary cards** — Principal, Instalment (accented), Rate, Cycles.
- **Table** — navy header, zebra striping, numeric columns right-aligned, dates left,
  cycle centred. Headings shortened to Opening / Closing so figures have room.
- **Totals row** — sums EMI, Interest and Principal across the schedule.
- **Footer** — hairline rule, "computer generated" note, page number. Repeats on every
  page, and the table's top margin is set so multi-page schedules never collide with the
  header band.

Interest-only loans carry `interestDue` where EMI loans carry `emi`; every figure in the
table, the totals and the Instalment card falls back accordingly so those statements
aren't blank.

Filename changed from `<loan>_Amortization_Statement.pdf` to
`<customer>_<loan>_Statement.pdf`.

---

## 4. Pass 1 — alignment sweep

### 3.1 Icon rendering — root cause behind most of the misalignment

`Icon` injects a Lucide SVG imperatively into a wrapper `<span>`. It applied the
caller's `className` to the **injected SVG**, while the wrapper `<span>` stayed in normal
document flow with no classes. So a call like

```jsx
<Icon className="absolute left-3 top-1/2 -translate-y-1/2" ... />
```

absolutely positioned the SVG but left a 16×16 phantom inline box in the flow ahead of
the input, nudging the field and leaving the glyph optically low and left.

`className` now goes on the wrapper `<span>`, which carries an explicit width/height.
Colour and sizing still inherit into the SVG via `currentColor` and `font-size`, so
existing `<Icon className="text-mid" />` calls are unaffected. This one fix corrected all
8 leading-icon instances app-wide.

### 3.2 Uniform control height

`inputCls` changed from `px-3 py-2.5` to `px-3 h-11` — every input and select is exactly
44px. Segmented button pairs (Loan Type, Interest Method) got `h-11` and
`whitespace-nowrap`, so "EMI (Amortizing)" no longer wraps and inflates its grid row.
This is what makes the New Loan wizard's two columns line up.

### 3.3 Onboarding tiles — option C

The 44px icon tile sat in a flex row alongside `<Field>` (label + input), and flex
top-alignment centred it on the *label*. The tile now carries `mt-[22px]` — exactly the
label block's height (16px line + 6px margin) — so its centre lands on the input's
centre, with the label flush to the input's left edge. Redundant row-level margins were
removed; rhythm now comes solely from `Field`'s own `mb-4`.

### 3.4 Date fields — one affordance, three fields

Start Date had a leading icon, a custom button *and* Chrome's native picker indicator
(three calendar glyphs); the two payment-date fields had only the native indicator,
which doesn't exist in Firefox or Safari — so they had no picker button outside Chrome.

A shared `DateField` component now backs all three: one right-side button calling
`showPicker()`, with a `focus()` fallback. The native indicator is hidden app-wide via a
scoped rule in the app's own stylesheet — **not** by editing the Tailwind preflight
block, so regenerating that CSS can't clobber it.

### 3.5 Responsive grids and table headers

Five `grid grid-cols-2` blocks forced two columns at every width; all are now
`grid-cols-1 sm:grid-cols-2`. Table `<th>` cells got `whitespace-nowrap` so "Interest %"
no longer wraps while its siblings stay on one line, which was breaking the header row's
baseline.

### 3.6 Security & PIN Lock

The amber "With a PIN… / Without a PIN…" banner was removed from onboarding step 2; the
button below picked up `mt-6` to rebalance the spacing. The `WarningBanner` component is
untouched and still used in 8+ other places.

---

## 5. Usage examples

**A date field** — use the shared component so the picker affordance stays consistent:

```jsx
<Field label="Disbursal Date">
  <DateField value={date} onChange={e => setDate(e.target.value)} />
</Field>
```

**A currency field** — grouping and caret handling come for free:

```jsx
<Field label="Principal Amount (₹)">
  <div className="relative">
    <Icon name="indian-rupee" size={16}
          className="absolute left-3 top-1/2 -translate-y-1/2 text-mid pointer-events-none" />
    <CurrencyInput className={inputCls + " font-mono pl-9"} value={v} onChange={setV} />
  </div>
</Field>
```

**A two-column form section** that collapses on phones:

```jsx
<div className="grid grid-cols-1 sm:grid-cols-2 gap-x-3 gap-y-0 items-start">
  <Field label="Payment Mode"><PaymentModeSelect value={m} onChange={setM} /></Field>
  <Field label="Date"><DateField value={d} onChange={e => setD(e.target.value)} /></Field>
</div>
```

---

## 6. Verification

The JSX was extracted and run through Babel's react preset offline after each pass — it
parses and transforms cleanly. The currency formatter and caret mapping were unit-tested
in Node: grouping is correct across the lakh and crore boundaries, decimals and trailing
dots survive, and front-insertion places the caret correctly.

**None of this proves it renders correctly.** The file could not be opened in a browser
during either pass — the container has no browser, and the React/Tailwind/Lucide CDNs are
outside its network allow-list. The PDF layout in particular is reasoned from the jsPDF
and autoTable APIs, not seen. Please check at **360px, 390px, 768px and 1280px**:

| # | Screen | Check |
|---|--------|-------|
| 1 | Customer detail | Icon tiles vertically centred; "Loans 1" correctly spaced; buttons same height, right-aligned; header wraps sanely when narrow. |
| 2 | Loan row | One figure, no "of …" line. **Confirm it's the number you wanted** — see 2.2. |
| 3 | Loan accordion | Tab reads "Schedule". |
| 4 | Any amount field | Commas appear while typing. Click into the middle of a number and type a digit — the caret should stay put. Test a decimal too. |
| 5 | Report → PDF Statement | Header band and footer on every page; summary cards populated; table striped and right-aligned; totals row present. Generate one for a **multi-page** schedule (60-month loan) and an **interest-only** loan. |
| 6 | Onboarding step 1 | Tiles centred on inputs, even spacing. |
| 7 | New Loan wizard | Columns line up; Start Date has exactly one calendar icon, on the right; grids collapse below 640px. |
| 8 | Record Payment | Date field has a working calendar button — **test in Firefox**, where it previously had none. |
| 9 | Any dropdown | Opens dark, not white. Check the loan filters, payment mode, rate basis, frequency and tenure — and check Firefox separately. |
| 10 | Customers list | Rows start collapsed. Clicking one expands it in place; collapsing shows the full list plus Add Customer. Trash icon deletes without expanding the row. |
| 11 | Deep links | From a dashboard loan row, global search, and a notification — each should expand the right customer **and** open the right loan. Also test the URL hash directly. |
| 12 | Dashboard / Reports | Summary and the Report-page customer cards start closed. |
| 13 | Modals from inside a customer | Expand a customer, then open **New Loan**, **Edit**, **Delete**, and a loan's **Pay** dialog. Each must cover the whole viewport, scroll internally, and close on backdrop click and Escape — not render clipped inside the card. |

---

## 7. Known issues NOT addressed

Found during review, deliberately left alone under the "no logic changes" instruction:

1. **Schedule Preview, row 1, Closing column** renders a value one digit short
   (`₹1,01,01,01` where row 2's Opening is `₹1,01,01,010`). Since `closing[n]` must equal
   `opening[n+1]`, either the formatter or the schedule generator is dropping a digit.
   This is a **wrong financial figure on screen** and should be next.
2. **Customer detail loan count** disagreed with the Report page in earlier screenshots
   ("0 Loans" vs "1 loan(s)"). The latest screenshot shows the correct count, so this may
   already be resolved — worth confirming against a customer with several loans.
3. **Confirm step summary** renders doubled spaces (`6  months`, `3  days`) — a label
   template joining a mono value to its unit with an extra space.

One deliberate divergence from the supplied mockups: `New_Loan_Page_1_1_New` showed Start
Date with both a leading icon and a right-side button — the duplicate-icon defect
originally flagged in red — so your explicit decision (button only) was followed and that
detail of the mockup treated as stale.
