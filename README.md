# Ledger AI — BI Dashboard (100% Python analysis, zero cloud API)

Single-file static site (`index.html`). No backend, no API keys, no
Supabase. Every bit of "AI" analysis — KPIs, forecasting, insight
narratives, SQL generation, chat — runs as **real Python**, in your
browser, via [Pyodide](https://pyodide.org) (Python compiled to
WebAssembly).

## How it works

- React, Recharts, PapaParse, SheetJS -> load from CDN, JSX compiled
  in-browser by Babel standalone. No build step.
- **Pyodide** loads a real CPython 3.12 runtime + `numpy` and `pandas`
  directly in the browser tab.
- All analysis -- column detection, KPIs, time series, customer/category
  breakdowns, revenue forecasting (numpy least-squares fit), the
  multi-language insight/forecast narratives, the SQL generator, and the
  chat responder -- is Python code (`py_engine`, embedded in `index.html`)
  that JS calls into via `pyodide.globals.get(fnName)(...)`.
- **No LLM anywhere.** Narratives are hand-written templates (10
  languages) filled in with numbers Python actually computed -- not
  generated text. SQL generation and chat are rule-based pattern matching
  over your schema, not an LLM either.

## Deploy to Vercel

**Option A - dashboard:** Add New -> Project -> "Deploy without a Git
repository" (or drag-and-drop this folder/zip) -> Framework: **Other**
(static, auto-detected) -> Deploy.

**Option B - CLI:**
```bash
npm i -g vercel
cd this-folder
vercel --prod
```
`index.html` at the root is served as-is -- nothing else to configure.

## Do you need Supabase?

**No.** Nothing here talks to a server at all -- not even for the "AI"
parts, which is the whole point of this version. Supabase (or any
backend) would only matter if you later want data shared across
devices/users, logins, or a real database instead of "the file I
uploaded in this browser tab" (currently kept in `localStorage`, per
browser).

## Honest tradeoffs of this approach

- **First load is heavy.** Pyodide + numpy/pandas is roughly
  15-25MB, downloaded once on first visit (cached by the browser after
  that). You'll see a "Python engine load ho raha hai..." screen for a
  few seconds on first load -- this is expected, not a bug.
- **SQL Generator and Chat are rule-based**, not LLM-flexible. Simple,
  direct phrasing works best: "top 5 customer by revenue", "total
  revenue", "trend kaisa hai". Unusual phrasing may not match -- the SQL
  generator says so explicitly rather than guessing wrong.
- **Forecasting uses a plain numpy least-squares line fit** --
  deliberately not scipy or a heavier model like LightGBM/XGBoost, to
  keep the initial download as small as possible (scipy's own wheel
  plus its BLAS dependency added real weight for a feature that's just
  a straight-line trend). The forecast card shows the R2 so you can
  judge how much to trust it.

## Testing note

I validated this thoroughly but couldn't do one specific test: my
sandbox's network firewall blocks `cdn.jsdelivr.net` (where Pyodide's
package files live), so I couldn't run the actual numpy/pandas
package-loading step end-to-end there. What I did verify directly:
- The exact JS<->Python calling pattern this app uses (`loadPyodide` ->
  `runPython` -> `globals.get(fn)(jsonArgs)` -> `JSON.parse`) works
  correctly, tested live in a real headless browser.
- The embedded Python source parses with **zero syntax errors** under
  the real Pyodide/CPython 3.12 interpreter (confirmed the only failure
  locally is the expected "module not found" for the blocked packages).
- Every Python function's actual logic -- column detection, KPIs,
  forecasting, all 10 language narratives, SQL generation, chat -- was
  run and manually checked against real CPython with numpy/pandas
  installed, with correct output.
- The app boots cleanly, shows the loading screen, and fails **gracefully**
  (friendly error message, not a crash) when package loading fails --
  confirmed live.

I'd still recommend one quick real-world check after you deploy (or by
opening `index.html` locally): open it, wait for "Python engine load ho
raha hai..." to finish, and click through the AI Insight / Forecast /
SQL / Chat buttons once. If anything's off, tell me what you see and
I'll fix it fast -- that's the one path I couldn't fully close the loop
on myself.

## What's new in this pass
- Cloud AI API (Anthropic) removed entirely -- replaced by the Pyodide
  Python engine described above
- Multi-language selector (10 languages) on Insight, Forecast Narrative,
  and Chat
- Forecast card now shows the model's R2 for honest confidence signaling
- Everything from the previous polish pass still applies: localStorage
  persistence, multi-sheet Excel support, CSV export, Load Sample Data,
  mobile-responsive layout

## Fixed after the first deployed version got stuck loading

If you deployed an earlier copy and saw the loading screen hang forever
showing literal text like `\u2014` and `\u2026` instead of proper
punctuation -- both are fixed now:

1. **Literal `\u2014`/`\u2026` text** -- these were unicode escapes
   written directly as JSX text content, which (unlike JS string
   literals) doesn't interpret `\u` escapes. Replaced with the actual
   typed characters throughout.
2. **Removed `scipy`** (and, as a side effect, its `openblas`
   dependency) from the packages Pyodide loads -- confirmed via local
   testing that this drops the resolved package list from
   `numpy, openblas, pandas, python-dateutil, pytz, scipy, six` down to
   just `numpy, pandas, python-dateutil, pytz, six`. Smaller download,
   fewer things that can fail to fetch. The forecast math (a straight
   trend line + R²) is now done with plain `numpy.polyfit` instead of
   `scipy.stats.linregress` -- same result, lighter dependency.
3. **Added real loading progress + diagnostics** -- the loading screen
   now shows which stage it's on (runtime -> packages -> engine) instead
   of one static message, logs each stage to the browser console, and
   after 15 seconds shows a hint to check the console if it's still
   stuck. If it ever hangs again, open DevTools (F12) -> Console and
   whatever error is there will point at the real cause.

## New features added

- **Date range filter** -- a date picker in the top bar (appears once a
  date column is detected) that scopes every KPI, chart, forecast, and
  customer breakdown to the selected range. Clear it to go back to the
  full dataset.
- **Data Quality panel** -- on the Upload tab: total rows, duplicate row
  count, and per-column null% / unique-value count / detected type
  (numeric vs text), computed with pandas.
- **Anomaly detection** -- on the AI Dashboard: flags unusually high or
  low periods in the revenue trend using a numpy z-score check
  (|z| >= 2), shown as a callout list with direction and the actual
  z-score, so you can judge severity yourself.

## Design polish (animated KPIs & charts)

No animation library was added (keeps this a zero-build single file) --
instead:

- **KPI numbers count up** from their previous value to the new one
  (ease-out curve, ~700ms) whenever the data changes -- filtering by
  date range, uploading a new file, or switching sheets.
- **Cards fade-and-rise in** on load, staggered slightly so they cascade
  rather than popping in all at once.
- **Hover lift** on every card -- subtle elevation + shadow.
- **Chart animations** are now explicit (easing + duration) across all
  trend/forecast/customer charts, with polished tooltips (rounded
  corners, soft shadow, a highlighted cursor line/bar on hover).
- **Tab switches fade** instead of snapping.

All of this is plain CSS keyframes + a small `requestAnimationFrame`
counter component + Recharts' own animation props -- no new dependency,
no extra download weight.

## Better forecasting & data understanding

**Smarter forecasting (still pure numpy, no scipy/sklearn):**
- **Model selection** -- fits both a straight-line and a curved (quadratic)
  trend, and automatically picks the curved one only when it meaningfully
  improves the fit (avoids overfitting on small datasets). The chosen
  model shows as a badge above the chart.
- **Seasonality detection** -- with at least a year of history, it works
  out a per-calendar-month seasonal index (e.g. "November/December
  consistently run higher") and factors that into the future projection,
  instead of a flat trend line.
- **Confidence band** -- the forecast chart now shows a shaded ~90%
  prediction interval (based on historical residual spread, widening the
  further out the projection goes), so you get a realistic range instead
  of one falsely-precise number.
- **Period-over-period growth table** -- every period's % change vs the
  previous one, not just a single start-vs-end comparison.

**Data understanding:**
- **Correlation matrix** -- on the AI Dashboard, a color-graded table
  showing how your numeric columns move together (pandas `.corr()`),
  green for positive correlation, red for negative.
