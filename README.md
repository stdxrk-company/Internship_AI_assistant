# 🏁 F1 2026 AI Assistant

An AI assistant that answers questions about the **2026 Formula 1 season** using
live data from public APIs. Built for AI Homework 1.

The model never answers from memory. Every question triggers a real HTTP request
to a Formula 1 data API, and the answer is written from that JSON.

The whole app is a **single self-contained file** — [`static/index.html`](static/index.html).
No build step, no server, no `npm install`. Open it in a browser and it runs:
HTML, CSS, JavaScript, the embedded F1 fonts and the F1 logo are all inlined, and
the page talks to the data and AI APIs directly from the browser.

## How it works

```
  User question
        │
        ▼
  ┌──────────────────┐   1. An OpenRouter model reads the question and
  │  AI (OpenRouter,  │      picks one of 7 tools + its arguments
  │  free models,     │      (OpenAI-style function calling).
  │  auto-fallback)   │      If a model is rate-limited it falls back
  └────────┬──────────┘      to the next free model automatically.
           │  get_driver_standings({ season: "2026" })
           ▼
  ┌──────────────────┐   2. The browser executes the call against
  │  F1Client (JS)    │      the live public API.
  └────────┬──────────┘
           │  GET https://api.jolpi.ca/ergast/f1/2026/driverStandings/
           ▼
  ┌──────────────────┐   3. Raw JSON is normalised and fed
  │  Live JSON        │      back to the model.
  └────────┬──────────┘
           ▼
  ┌──────────────────┐   4. The model analyses the data — computes
  │  AI (analysis)    │      gaps, compares teams, summarises —
  └────────┬──────────┘      and writes the final answer.
           ▼
   Answer + a live, timestamped pipeline log on the page
```

## AI: OpenRouter with automatic model fallback

The assistant calls [OpenRouter](https://openrouter.ai) (OpenAI-compatible API)
using **free models only**. Free models have small daily quotas, so the app walks
a chain of models and automatically switches to the next one whenever a model is
rate-limited (`429`), out of quota (`402`), unavailable (`404`) or overloaded
(`500`/`503`). Every switch is shown in the live log.

Model chain (edit `MODEL_CHAIN` near the top of the `<script>` to reorder):

1. `openai/gpt-oss-20b:free` — primary, reliable tool-calling
2. `nvidia/nemotron-3-super-120b-a12b:free`
3. `inclusionai/ling-3.0-flash:free`
4. `nvidia/nemotron-3-ultra-550b-a55b:free`
5. `nvidia/nemotron-nano-9b-v2:free`

## Data sources

Both are public, free, and require **no API key**. They are called straight from
the browser.

| Source | Used for | Endpoint |
|---|---|---|
| [Jolpica-F1](https://github.com/jolpica/jolpica-f1) | standings, race & qualifying results, calendar, next race | `api.jolpi.ca/ergast/f1/…` |
| [OpenF1](https://openf1.org) | trackside weather during a race | `api.openf1.org/v1/…` |

> Jolpica-F1 is the community-maintained successor to the Ergast API, which shut
> down after the 2024 season. It keeps the same response format and is updated
> throughout the current season.

## Tools available to the model

| Tool | Returns |
|---|---|
| `get_driver_standings` | driver championship table |
| `get_constructor_standings` | team championship table |
| `get_race_results` | full classification of one race |
| `get_qualifying_results` | qualifying with Q1/Q2/Q3 times |
| `get_season_schedule` | full season calendar |
| `get_next_race` | next race + session times (UTC) |
| `get_race_weather` | air/track temperature, humidity, wind, rain |

## Run

No installation. No Node.js. Just:

1. Open [`static/index.html`](static/index.html) in any modern browser
   (double-click it, or drag it into a browser tab).
2. Ask a question.

An OpenRouter API key is already embedded in the file, so it works out of the box.
To use your own key, get a free one at <https://openrouter.ai/keys> and replace the
`OPENROUTER_API_KEY` constant near the top of the `<script>` in `index.html`.

> ⚠️ **Note on the key.** Because the app is a single client-side file, the
> OpenRouter key is visible to anyone who opens `index.html`. That is fine for a
> local / classroom project, but do **not** publish the file with a real key in it.

### CORS

The browser calls `api.jolpi.ca`, `api.openf1.org` and `openrouter.ai` directly.
These normally allow cross-origin requests, so the page works when opened from a
file. If a network or extension blocks one of them, that data simply won't load.

## Example questions

- Who leads the 2026 championship and by how much?
- Compare Ferrari and Mercedes this season
- Summarise the last race
- When is the next race, and what was the weather at the last one?

## Live pipeline log

While a question is processed, the page streams a timestamped console (in English)
showing exactly what happens, step by step:

```
[00:00.000] System ready. Data: Jolpica-F1 + OpenF1. AI: OpenRouter (free models).
[00:00.014] 🎙 Question received: "Who leads the 2026 championship?"
[00:00.028] 🧠 Step 1 — AI (openai/gpt-oss-20b:free) is deciding which API request is needed…
[00:00.711] ⚠ openai/gpt-oss-20b:free: HTTP 429 … — switching to another model…
[00:06.098] 🧠 AI (nvidia/nemotron-3-super-120b-a12b:free) chose tool: get_driver_standings(season="2026")
[00:06.104] 📡 Step 2 — fetching live data from the F1 API…
[00:06.108] ↳ GET https://api.jolpi.ca/ergast/f1/2026/driverStandings/?format=json
[00:06.500] 📦 Data received (records: 22).   [show JSON (2.7 KB)]
[00:06.503] 🧠 Step 3 — AI is analysing the data and writing the answer…
[00:14.492] ✅ Done — answer produced by nvidia/nemotron-3-super-120b-a12b:free.
```

Each data line has a collapsible **show JSON** toggle with syntax highlighting.

## Interface

The page follows the [formula1.com](https://www.formula1.com) design language and
uses the **real Formula1 brand font** (embedded from the official site) plus the
**official F1 logo** (inline SVG, in the header and hero). The look: `#15151E`
carbon background, `#E10600` brand red, `#27F4D2` accent, heavy uppercase headings
and skewed racing accents. Driver cards are tinted with their team colour. The
whole interface — including the live log — is in English, and the assistant answers
in English regardless of the language of the question.

> The embedded Formula1 font and F1 logo are the property of Formula 1. They are
> used here only for a local, non-published student project.

## Project layout

| File | Role |
|---|---|
| `static/index.html` | **the entire app** — HTML, inlined CSS, inlined JS (data layer + AI loop + UI), embedded fonts and F1 logo |
| `.env.example`, `package.json`, `package-lock.json` | leftovers from the original Node/Gemini version — not used by the current single-file app |

> Earlier versions split the app into `f1.js`, `gemini.js`, `server.js`,
> `static/style.css` and `static/app.js` and ran on a Node server with Google
> Gemini. Everything has since been consolidated into `static/index.html` and
> moved to OpenRouter, so those files were removed.
