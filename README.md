# JP Pricing Tool

A single-file, zero-backend web app — themed as a Japanese bamboo forest — that
reads **MHLW drug-price publication PDFs** (in Japanese), translates them to
English, and produces a structured summary table of each drug's pricing details.

## What it does

- Upload one or more Japanese MHLW drug-price PDFs (button or drag & drop).
- Each PDF is sent **directly from your browser** to the Anthropic API
  (`claude-opus-4-8`) as a base64 `document` block, using structured-output
  extraction. No server of ours is involved.
- Results render (translated to English) in a table with an expandable per-drug
  drop-down: drug/generic name, date priced, category, manufacturer, pricing
  method (算定方式), comparator drug (最類似薬), premiums earned (補正加算 with %),
  pre/post-premium prices, foreign price adjustment (外国平均価格調整), foreign
  comparator prices, PMP eligibility (新薬創出加算), and notes.
- Export the results to a real Excel `.xlsx` (via SheetJS).

## Using it

1. Open the deployed page (or `index.html` locally).
2. Paste your **Anthropic API key** in step 1 and click **Save**. The key is
   stored only in your browser's `localStorage` and sent only to Anthropic from
   your device — it is never committed or sent anywhere else. Get a key at
   <https://console.anthropic.com/settings/keys>.
3. Upload PDFs, press **Analyze**, expand rows for full detail, and **Export to
   Excel**.

The **Load sample row** button seeds a demo drug so you can try the table,
drop-down detail, and Excel export without a key.

## Architecture

`index.html` is intentionally a **single self-contained file** — all CSS and JS
are inlined; the bamboo forest, komorebi light, and dust motes are pure CSS plus
a JS-drawn inline `<svg>`. The only external dependency is **SheetJS** (loaded
from an absolute CDN URL) for `.xlsx` export; the page degrades gracefully if the
CDN is blocked (only export is affected). There is no build step.

## Deploying

Because `index.html` is at the repo root, any static host serves it at `/` with
no configuration. On Vercel: import this repo as a new project and deploy — no
`vercel.json` needed.

> AI-extracted output should be verified against the source documents for any
> regulatory or commercial use.
