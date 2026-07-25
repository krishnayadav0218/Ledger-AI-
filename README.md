# Ledger AI — BI Dashboard (100% Python analysis, zero cloud API)

Single-file static site (`index.html`). No backend, no API keys, no
Supabase. Every bit of "AI" analysis — KPIs, forecasting, insight
narratives, SQL generation, chat — runs as **real Python**, in your
browser, via [Pyodide](https://pyodide.org) (Python compiled to
WebAssembly).

## How it works

- React, Recharts, PapaParse, SheetJS -> load from CDN, JSX compiled
  in-browser by Babel standalone. No build step.
- **Pyodide** loads a real CPython 3.12 runtime + `numpy`, `pandas`,
  `scipy` directly in the browser tab.
- All analysis -- column detection, KPIs, time series, customer/category
  breakdowns, revenue forecasting (`scipy.stats.linregress`), the
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

- **First load is heavy.** Pyodide + numpy/pandas/scipy is roughly
  15-25MB, downloaded once on first visit (cached by the browser after
  that). You'll see a "Python engine load ho raha hai..." screen for a
  few seconds on first load -- this is expected, not a bug.
- **SQL Generator and Chat are rule-based**, not LLM-flexible. Simple,
  direct phrasing works best: "top 5 customer by revenue", "total
  revenue", "trend kaisa hai". Unusual phrasing may not match -- the SQL
  generator says so explicitly rather than guessing wrong.
- **Forecasting uses linear regression** (`scipy.stats.linregress`) --
  deliberately not a heavier model like LightGBM/XGBoost, since those
  would add tens of MB more to the initial download for a dataset size
  where the extra sophistication rarely pays off. The forecast card
  shows the R2 so you can judge how much to trust it.

## Testing note

I validated this thoroughly but couldn't do one specific test: my
sandbox's network firewall blocks `cdn.jsdelivr.net` (where Pyodide's
package files live), so I couldn't run the actual numpy/pandas/scipy
package-loading step end-to-end there. What I did verify directly:
- The exact JS<->Python calling pattern this app uses (`loadPyodide` ->
  `runPython` -> `globals.get(fn)(jsonArgs)` -> `JSON.parse`) works
  correctly, tested live in a real headless browser.
- The embedded Python source parses with **zero syntax errors** under
  the real Pyodide/CPython 3.12 interpreter (confirmed the only failure
  locally is the expected "module not found" for the blocked packages).
- Every Python function's actual logic -- column detection, KPIs,
  forecasting, all 10 language narratives, SQL generation, chat -- was
  run and manually checked against real CPython with numpy/pandas/scipy
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
