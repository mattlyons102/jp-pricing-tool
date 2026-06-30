# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## Overview

**JP Pricing Tool** — a single-file, zero-backend web app themed as a realistic
Japanese bamboo forest with soft light filtering in (komorebi). It reads
**MHLW (厚生労働省 / 中医協) drug-price publication PDFs** (in Japanese, often with
abbreviations), translates them to English, and produces a structured summary
table of each drug's pricing details, with an Excel (`.xlsx`) export.

The whole app is **`index.html`** at the repo root. There is no build step, no
package manager, and no test suite.

### History / context
This started life inside the `mattlyons102/pomodoro-pizzeria` monorepo but was
extracted to its **own repo** (`mattlyons102/jp-pricing-tool`, public) so it
deploys as its own Vercel project. It shares no code with that repo. Keep it
standalone.

## Running

It's a static file — open `index.html` directly, or serve it:
```bash
python3 -m http.server 5502 --directory .
```
For Claude Code preview, add a `.claude/launch.json` (`runtimeExecutable`
`python3`, args `-m http.server <port> --directory .`) and use `preview_start`.

## Deployment

Its own **Vercel project**, imported from this GitHub repo. Because `index.html`
is at the **repo root**, Vercel serves it at `/` with **no configuration** — no
`vercel.json`, no rewrites, framework preset "Other", default build/output.
Pushing to `main` auto-redeploys.

**Deployment Protection caveat (important):** the owner's Vercel team
(`lumbertones`) appears to have **Vercel Authentication / Deployment Protection**
enabled, which makes `*.vercel.app` URLs redirect to a Vercel login (this looked
like "an error" / a broken link during setup). If the production URL shows a
Vercel login instead of the app, turn it off in **Project → Settings →
Deployment Protection** (set Vercel Authentication to *Disabled* or *Preview
only*), or attach a custom domain. Note: `pomodoro-pizzeria.vercel.app` (the bare
subdomain) is squatted by an **unrelated** project — not ours; ignore it.

## Architecture notes

**`index.html` is intentionally a single self-contained file** — all CSS
(`<style>`) and JS (`<script>`) are inlined; the bamboo forest, komorebi light
shafts, and dust motes are pure CSS plus a JS-drawn inline `<svg>` (`drawBamboo()`,
deterministic pseudo-random so the scene is stable across reloads). **One
deliberate exception:** SheetJS for `.xlsx` export is loaded from an absolute CDN
URL (`https://cdn.sheetjs.com/...`). An absolute URL is immune to the
relative-path-404 problem that motivates inlining everything else, and the page
degrades gracefully if the CDN is blocked (only export is affected). Do **not**
re-split into separate `.css`/`.js` files.

**LLM extraction — browser-direct to the Anthropic API.** On Analyze, each PDF is
sent straight from the browser via `fetch` to `POST https://api.anthropic.com/v1/messages`
as a base64 `document` block (`buildBody()`), so there is **no OCR / pdf.js layer**
— Claude reads the PDF (incl. scanned pages and Japanese) natively. Model:
**`claude-opus-4-8`**. Required headers: `x-api-key`, `anthropic-version: 2023-06-01`,
and **`anthropic-dangerous-direct-browser-access: true`** (this is what enables
browser CORS — without it the call fails).

**Structured output is prompt-based, NOT `output_config`.** We intentionally do
**not** use the `output_config`/`json_schema` structured-output parameter: the raw
browser API call can reject it (it can require a beta header), which silently
broke every analyze during early testing. Instead the prompt (`PROMPT`) instructs
the model to return a single JSON object only, and `stripJson()` tolerantly parses
it (strips ``` fences / surrounding prose, slices from first `{` to last `}`). If
you ever reintroduce `output_config`, verify it works from a real browser request
first. (`DRUG_SCHEMA` / `nul()` remain in the file as a shape reference but are no
longer sent.)

**API key handling.** The Anthropic key lives **only** in the browser's
`localStorage` under `jp_pricing_api_key` — never committed, never sent anywhere
but Anthropic. Each visitor supplies their own key.

**Error handling (learned the hard way).** Per-file failures show the **actual
reason inline in red** under each file (`.ferror`) AND in the banner. Do not let a
generic summary banner overwrite the specific reason — an earlier bug did exactly
that, so errors appeared with no description. `analyzeFile()` throws descriptive
errors for: non-OK HTTP (uses `data.error.message`), `stop_reason === "refusal"`,
`max_tokens` truncation, and non-JSON responses (includes the first ~140 chars).

**Common runtime errors the user may hit** (all now surfaced inline):
- `invalid x-api-key` — wrong/empty key.
- `Your credit balance is too low...` — a new key with no billing/credit set up;
  fix in the Anthropic console.
- Large/too-many-page PDFs — the API caps PDF size/pages per request.

## Extraction fields (one object per drug; English; `null` if not stated)

`drug_name`, `generic_name`, `strength_formulation`, `date_priced` (薬価収載日),
`drug_category` (区分), `manufacturer`, `pricing_method` (算定方式),
`comparator_drug` (最類似薬), `premiums` (補正加算 — `[{name, value}]`, e.g.
有用性/画期性/市場性/小児/先駆加算 with %), `price_pre_premium`,
`price_post_premium`, `foreign_price_adjustment` (外国平均価格調整 —
`{received, direction, adjustment}`), `foreign_comparator_prices`
(`[{country, price}]`, US/UK/DE/FR), `pmp_eligibility` (新薬創出加算), `notes`
(abbreviation expansions, caveats). `source_file` is attached client-side.

## UI

Bamboo-themed cards: (1) API key, (2) upload (drag-drop + file list with status
chips: queued/analyzing/done/error), (3) results — a master table with an
expandable per-drug **drop-down** detail (`detailHtml()`), a file filter
`<select>`, and an **Export to Excel (.xlsx)** button (`XLSX.json_to_sheet` →
`XLSX.writeFile`, downloads locally). Files are processed **sequentially** to
respect rate limits.

## Verifying changes

The **"Load sample row"** button (`#sampleBtn`, calls into the same render path)
seeds a demo drug so you can verify the table, the expandable detail, and the
`.xlsx` export **without an API key**. To verify the request/error path without a
real key, set a bad key + a fake `File` and call `analyzeAll()` via `preview_eval`
— it should reach the API and surface a clean error inline. **Real end-to-end
extraction requires a valid Anthropic API key and a real MHLW PDF** and can't be
fully exercised in-session. Key globals are page-scoped and reachable from
`preview_eval`: `files`, `analyzeAll`, `addFiles`, `buildBody`, `stripJson`,
`renderResults`, `PROMPT`.
