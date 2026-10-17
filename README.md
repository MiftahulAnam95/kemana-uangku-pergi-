# WHERE DID MY MONEY GO?

> **Your salary came. Your salary left. Nobody knows why.**
>
> *Gajian masuk. Duit menghilang. Tersangka masih diselidiki.*

A mobile-first, bilingual personal spending tracker that behaves like a small
financial detective agency: you file the evidence (transactions), it shows you
the pattern — with charts, budgets, rule-based findings, and a healthy sense of
humour. It never shames you, and it never talks to a bank.

---

## Contents

- [What it does](#what-it-does)
- [Tech stack](#tech-stack)
- [Run it locally](#run-it-locally)
- [Deploy to GitHub Pages](#deploy-to-github-pages)
- [Data model & storage](#data-model--storage)
- [CSV format (import / export)](#csv-format--import--export)
- [How the numbers are calculated](#how-the-numbers-are-calculated)
- [Humour policy](#humour-policy)
- [Accessibility](#accessibility)
- [Privacy](#privacy)
- [Project structure](#project-structure)

---

## What it does

| Area | Details |
| --- | --- |
| **Overview** | Month income, expenses, remaining balance, savings rate, average daily spend, and a comparison against the previous month (only when the previous month actually has entries). |
| **Add / edit / delete** | Dialog with name, amount (tolerant parser: `25.000`, `25,000`, `Rp 25000`), type, category, date and optional note. Deleting always asks for confirmation first. |
| **Visualisation** | Hand-built SVG: category donut (hover / focus / tap highlights the legend and filters the ledger), daily **or** weekly spending bars, and a 3-month income-vs-expense comparison. Every chart has a textual equivalent (`The numbers` → real `<table>`) plus an `aria-label` summary. |
| **Insights** | Rule-based findings that cite their own evidence (top-3 concentration, biggest category, month-over-month change, per-category change, small purchases adding up, weekend share, quiet days, salary-week effect, budget pressure). If the data is too thin, it says so instead of inventing an opinion. |
| **Budget planner** | Per-category monthly limits with limit / spent / left, a progress track, configurable **nudge** and **alarm** thresholds, and a total-budget bar. |
| **Reality check** | Pick categories + a redirected share (0–60 %) and it computes the hypothetical cushion, a 12-month extrapolation of *your own* pace, and what the amount equals in days of your average spend. Labelled hypothetical everywhere; no return promises. |
| **Transaction management** | Search, category filter, type filter, period (selected month / all time / custom range), 5 sort orders, edit, delete, CSV export, CSV import with per-line validation errors. |
| **Monthly report** | Choose any month with data, see income / expenses / net / savings rate, category distribution, budget performance and key findings. Print or save as PDF, or copy a plain-text summary. |
| **Data** | JSON export, JSON restore (validated), demo case file (explicitly labelled fake data), delete-everything with confirmation, and a live storage readout. |
| **Language** | Bahasa Indonesia is the primary/default language. The `🇮🇩 ID \| EN` switch in the header changes everything instantly (labels, charts, filters, insights, dialogs, empty states, and toasts). An explicit choice persists; older stores migrate to Indonesian once. |

## Tech stack

The complete application, including its CSS and vanilla JavaScript, is in
**`index.html`**. No build step, bundler, JavaScript framework, external financial
API, backend, or analytics is used. Tailwind's browser CDN is included; the
existing editorial design uses the inline application stylesheet.

- Charts: hand-authored inline SVG (no chart library on purpose).
- Numbers: `Intl.NumberFormat` — rupiah locale `id-ID` in ID mode; in EN mode a
  clearly labelled selected currency (`IDR 1,234,567`, `USD 1,234.56`, `SGD 1,234.56`).
- Icons: Lucide via CDN, purely decorative, with graceful degradation if it fails to load.
- Fonts: Bricolage Grotesque (display), Instrument Sans (body), Space Mono (figures).
- Storage: `localStorage` only.

## Run it locally

```bash
git clone <your-repo-url> && cd <repo>
# then just open index.html, or serve it:
python3 -m http.server 8080      # http://localhost:8080
npx serve .                      # if you prefer Node
```

`localStorage` needs a normal origin, so `file://` works in most browsers but a
tiny local server is the safest way to test import/export.

## Deploy to GitHub Pages

1. Push `index.html` and `README.md` to a repository (e.g. on `main`).
2. **Settings → Pages → Source:** *Deploy from a branch* → `main` / `/ (root)`.
3. Save. The app is live at `https://<user>.github.io/<repo>/`.

All application code is inline, so the app works from a project subdirectory
without path configuration. Fonts, decorative icons, and Tailwind are loaded
from their CDNs. No build or additional application files are required.

## Data model & storage

Key: **`wdmg.store.v1`**, schema version stored alongside the payload.

```jsonc
{
  "version": 1,
  "transactions": [
    {
      "id": "tx_k1a2b3c4",
      "name": "Kopi susu",
      "amount": 18000,          // positive number, 2dp max
      "type": "expense",        // "expense" | "income"
      "category": "food",       // one of the 15 known ids
      "date": "2026-05-14",     // validated YYYY-MM-DD, local (never shifted)
      "note": "Pagi-pagi",      // ≤ 180 chars
      "createdAt": 1747200000000
    }
  ],
  "budgets": { "food": 900000 },   // catId → monthly limit
  "prefs": {
    "lang": "id", "currency": "IDR", "month": "2026-05", "trend": "daily",
    "warn": 80, "max": 100,
    "realityCats": ["food", "fun", "shopping"], "realityShare": 20
  }
}
```

`Store.normalize()` validates locally stored version 1 data. The existing
`wdmg.store.v1` key and transaction schema are retained, so valid existing entries
remain usable without importing or resetting them. Invalid individual records
are normalized or omitted from the in-memory view. An unreadable store opens an
empty view with a notice rather than automatically overwriting the stored file.

JSON restores are stricter: the application identifier, version, transaction
fields, unique IDs, and category budgets must validate before confirmation.
Valid empty backups are supported. Exports from this app are valid restore files.

Categories — expenses: `food, transport, shopping, bills, education, fun,
health, savings, family, other`. Income: `salary, side, gift, refund, invest`.

## CSV format

Header line (exact names, order free):

```csv
date,name,category,type,amount,note
2026-05-14,"Kopi susu, large",food,expense,18000,"Pagi-pagi"
2026-05-01,"Gaji Mei",salary,income,4200000,
```

- `date`: a real calendar date in `YYYY-MM-DD` format.
- `name`: required; up to 60 characters are retained.
- `category`: one of the IDs above, matching the transaction type.
- `type`: `expense` or `income`.
- `amount`: a positive number with at most two decimal places; grouping separators are accepted.
- `note`: optional content, with a required header column; up to 180 characters are retained.
- Quoted commas, quotes, newlines, and a UTF-8 BOM are supported.
- Invalid rows are skipped and reported before confirmation; dates and categories are not silently substituted on CSV import.

**Export** writes the rows currently matching your filters — filter first if you
only want one month. **Import** appends and never overwrites; rows that fail
validation are listed with their line numbers instead of silently dropped.
Use the **Template** button to download a ready-made file.

## How the numbers are calculated

There is a single source of truth. Screens never re-add totals themselves:

```
Store.data.transactions
   └─ Calc.monthTxs(month)      → filtered rows
        ├─ Calc.totals()        → income, expense, net, savingsRate
        ├─ Calc.categoryBreakdown() → donut, legend, accessible table
        ├─ Calc.byCategory()    → category amounts, counts, shares, report
        ├─ Calc.daily/weeklySeries() → trend chart
        ├─ Calc.budgetOverview()→ budget cards + warnings
        ├─ Calc.reality()       → hypotheticals
        └─ Calc.insights()      → findings (each carries its own proof chips)
```

Savings rate = `(income − expenses) / income`; it shows `—` and explains itself
when no income has been logged for the month. Comparisons need at least one
expense in each compared month; otherwise the UI explains that more entries
are needed. Totals and category aggregation add integer minor units rather
than repeatedly adding fractional currency values.

## Category Breakdown Fix

- The donut's rotation is on its SVG group. Hover and focus only adjust stroke
  width and opacity, so highlighting no longer rotates or displaces a slice.
- Slice sizes use the actual category shares. Small categories are never
  artificially enlarged, and a single category fills the ring.
- The chart includes only expenses in the selected month. Income-only and
  empty months show a neutral ring and an add-expense action, not invented data.
- Full currency amounts, matching category colors, localized percentages,
  and a total row are available in the legend and accessible data table.
- Percentage labels use one shared formatter. Small positive shares display
  `<0.1%` in English and `<0,1%` in Indonesian; a note explains rounding.
- The panel adapts to its own width. Labels can wrap, amounts are not truncated,
  and the legend stacks below the donut when side-by-side content would be cramped.
- Selecting a slice or legend row opens that category's expenses for the selected
  month. Old search/date filters are cleared, and dropdowns stay synchronized
  with the filter state, including after switching language.
- SVG slice controls and legend buttons support keyboard activation. Interactive
  slices are no longer hidden from the accessibility tree.

## Verification

Read-only regression checks are included in `index.html`. Open the app and run
`runCategoryChecks()` in the browser console. The checks use temporary in-memory
fixtures and detached SVG elements; they do not write test transactions or alter
localStorage. Results are printed as a table, and any failed check throws an error.

Coverage includes month filtering, excluding income, 75/25 proportions, decimal
addition, invalid amounts, every expense category, empty and single-category
charts, tiny shares, EN/ID formatting, keyboard roles, stable SVG rotation,
category edits/deletions, filter state, translation coverage, and data preservation.

For visual and interaction checks, use a separate browser profile:

1. Add expenses of 75,000 in Food and 25,000 in Transportation in the same month.
2. Add income in that month and an expense in another month. The chart should
   still total 100,000, with 75% and 25% shares.
3. Hover or keyboard-focus both slices. Their positions must stay fixed.
4. Set a search and date filter, then select a category in the chart. The ledger
   should show that month's category expenses with matching dropdown values.
5. Switch EN/ID, change the selected month, edit a category, and delete the last
   expense. Check that all chart values, table rows, and empty states update.
6. Check narrow mobile, tablet, and desktop widths, plus reduced-motion mode.

Source review was performed during the fix. These browser checks were added but
could not be executed in the editing environment; visual behavior and browser
execution still need verification.

## Humour policy

The joke is always at the situation, never at the person. Copy is written so
that (a) it is funny with the numbers hidden, (b) it never states a fact the app
has not measured, and (c) no message implies the user is careless, lazy or bad
with money. Indonesian mode is casual Gen-Z (`lu`, `jajan receh`,
`evaluasi tipis-tipis`) without drowning in slang. Money-related instructions —
budgets, thresholds, reality check — are neutral and literal.

Nothing in the app claims bank access: the header, the receipt, the report
footer and the privacy panel all repeat that it is hand-entered data.

## Accessibility

- Mobile-first, one column, thumb-height inputs, no hover-only features.
- Every dialog is keyboard-operable: focus moves in, `Tab` is trapped, `Esc`
  closes, focus returns to the trigger. `role="dialog"` / `role="alertdialog"`.
- Visible focus rings, `aria-pressed` on the language and trend toggles,
  `role="progressbar"` with values on budget tracks.
- The interactive category SVG uses `role="group"` with labelled slice buttons.
  Noninteractive chart summaries use `role="img"`. Real data tables are provided
  under **The numbers**.
- Toasts use `aria-live="polite"`; donut slices are focusable buttons that filter
  the ledger with the keyboard.
- `prefers-reduced-motion` disables the count-ups, reveals, drifting background
  and chart transitions. Nothing animates to communicate meaning.
- The print stylesheet isolates the monthly report for an ink-friendly printout;
  page count depends on the report contents and printer settings.

## Privacy

- Data lives in `localStorage` in one browser. No accounts, no network requests
  containing your numbers, no cookies, no third-party analytics.
- The application has no bank, e-wallet, or card connection and sends no financial
  records over the network. CDN requests load styling, fonts, and decorative icons.
- Clearing site data / private browsing deletes records: export a JSON backup.
- Multi-device use = export on one, import on the other.
- Not financial, investment or therapeutic advice.

## Project structure

```
.
├── index.html      # complete app: markup, styles, logic, and read-only checks
└── README.md
```

Inside the inline application script in `index.html`, the layers stay separate:

| Layer | Responsibility |
| --- | --- |
| `STR` / `I18n` | every visible string, in both languages, with `{placeholder}` interpolation |
| `Store` | read, validate, normalize, save synchronously, clear |
| `Money` | the only place a number becomes text (and back) |
| `Calc` | all arithmetic: totals, aggregation, series, budgets, insights |
| `Charts` | SVG generation from `Calc` output only |
| `UI` | toasts, modals, focus trap, animated counters |
| `render*` | DOM painting, one function per section |
| `bindEvents` | every listener in one place |
| `runCategoryChecks` | read-only category regression checks |

**Known limits (by design):** no cloud sync, no recurring transactions, no
multi-currency conversion (the currency setting only changes formatting, and the
UI says so), and one browser per dataset.

## License

MIT — do whatever, credit appreciated. Built so you can actually read the source.
