# AI Finance Briefing — Handoff Document

**Last updated:** 12 November 2026
**Owner:** Rufus Kendall
**Status:** Live and in daily use

---

## 1. Purpose

A personal, single-page web app that generates a daily spoken-word podcast briefing on AI developments in financial services. Each run produces a 3–8 minute MP3 that the owner listens to on his commute.

The app exists because no commercial podcast covers the specific intersection of AI, UK/European financial services, and the owner's professional focus areas (CRM, agentic AI, regulatory moves, major UK banks and insurers). Rather than scan dozens of newsletters, the app does the scanning, synthesises the week's news into a tightly-written script, reads it aloud, and keeps a searchable archive of every episode so the same story is never covered twice.

It is not a product for anyone else — it is a personal tool, deliberately minimal, and intentionally hand-rolled in a single HTML file so the owner can edit it live from any device.

---

## 2. Live URLs & Repository

| Resource | URL |
| --- | --- |
| **Live app** | https://kendru128.github.io/ai-finance-briefing/ |
| **GitHub repository** | https://github.com/kendru128/ai-finance-briefing |
| **GitHub editor (quick-edit `index.html`)** | https://github.com/kendru128/ai-finance-briefing/edit/main/index.html |
| **Dropbox app console** | https://www.dropbox.com/developers/apps/info/c8qhhuxt5z26zbn |
| **Dropbox app folder** | `Dropbox/Apps/AI_Podcast_Briefing/` |

Deployment is GitHub Pages, serving the `main` branch root. A commit to `main` goes live within about 60 seconds. There is no build step, no bundler, no framework — the entire app is one `index.html` file.

---

## 3. Architecture at a glance

```
┌──────────────────────────────────────────────────────────────┐
│                  Browser (index.html, ~1,800 lines)          │
│                                                              │
│  UI layer:  single-page vanilla HTML + CSS + JS              │
│                                                              │
│  State:     localStorage (keys, model prefs, focus areas)    │
│             + in-memory arrays (episodeHistory, currentScript)│
└──────────┬───────────────┬─────────────────┬─────────────────┘
           │               │                 │
           ▼               ▼                 ▼
    ┌───────────┐    ┌───────────┐    ┌──────────────┐
    │ Anthropic │    │  OpenAI   │    │   Dropbox    │
    │  Claude   │    │    TTS    │    │  App Folder  │
    └───────────┘    └───────────┘    └──────────────┘
      research +        audio             episode archive
      script +         (MP3)              (JSON, MP3, TXT)
      fact-check
```

**No backend of its own.** Every API call runs from the browser directly to the relevant third party. Credentials live only in the user's own browser (localStorage). This is why the Anthropic calls use the `anthropic-dangerous-direct-browser-access: true` header — Anthropic normally discourages direct-from-browser calls, but for a single-user personal app with the user's own key, it's fine.

---

## 4. Generation pipeline

The **Generate** button runs a five-stage pipeline. Each stage shows progress in the UI.

### Stage 1 — Research (Claude Haiku + web search)

A single Claude call with the `web_search_20250305` tool. Prompt asks for a broad, well-sourced pool of 10–12 candidate stories from the last 7 days, covering four fixed angles (AI in UK/EU financial services, regulatory activity, research breakthroughs, agentic AI deployments) plus **targeted searches for every item in the user's Focus Areas list**. The focus-areas pass is critical — the generic sweep misses most practitioner-level news.

Each candidate must carry: headline, what is genuinely new, a short canonical tag, source publication AND full URL, publication date, and any hard deadline within 7 days.

Hard rules in the prompt: don't fabricate figures, dates, names, products or quotes; cite every claim; suppress stories already aired (passed in as `historyNote`) unless there's genuinely new information.

### Stage 2 — Script writing (Claude Sonnet)

A 10-second pause for rate limits, then Claude Sonnet is handed the research pool and asked to curate it down to what fits the chosen duration (3/5/8 minutes = 450/750/1200 words) and write a podcast script in the voice of the host, "Neo".

The prompt enforces: accuracy over completeness, lead with new facts, warm professional intro, smooth transitions, no stage directions or sound effects, no markdown.

Three additional context blocks are built in JS and injected into the prompt:
- **Entity cross-reference** — companies/regulators/products mentioned recently
- **Framing context** — recurring narratives
- **Pending topics** — follow-up angles from previous days

### Stage 2b — Grounding check (fact verification)

A second Sonnet call reads the finished script against the research pool and emits a JSON list of claims not supported by the pool — specifically named people, direct quotes, named AI models/products, named companies, statistics, monetary figures, dates. Common-knowledge background is ignored.

### Stage 2c — Correction pass (conditional)

If Stage 2b flagged anything, a third Sonnet call rewrites the script with those claims either deleted or softened to only what the pool supports. Nothing else is changed.

### Stage 3 — Text-to-speech (OpenAI)

The script is chunked (OpenAI TTS has a 4000-character limit), each chunk sent to the OpenAI TTS endpoint with the chosen voice (default: Fable — British, warm), and the returned MP3 buffers are concatenated into a single Blob.

### Stage 4 — Save to Dropbox

Script, sources and metadata written to `/memory.json` (one row per episode). The MP3 and script text can also be pushed to `/audio/` and `/scripts/` subfolders via the "Save to Dropbox" button in the Audio and Save tabs.

---

## 5. Dropbox integration

### App registration

- **App name:** `Claude_FinancePodcast_Briefing`
- **App key (client ID):** `c8qhhuxt5z26zbn` (hardcoded in the app — public, by design)
- **Access type:** Scoped access, App folder (limited scope — the app can only read/write inside its own folder, not the whole Dropbox)
- **Permissions:** `files.content.read`, `files.content.write`, `account_info.read`
- **Redirect URI:** `https://kendru128.github.io/ai-finance-briefing/`

### OAuth 2.0 PKCE flow

Dropbox no longer offers permanent "generated" tokens for new apps — all tokens are short-lived (~4 hours) unless obtained via the OAuth flow with `token_access_type=offline`, which returns a **refresh token** that never expires.

The app implements the full PKCE (Proof Key for Code Exchange) flow:

1. User clicks **Connect Dropbox** in the Configure panel.
2. JS generates a cryptographically random `code_verifier`, derives a SHA-256 `code_challenge`, stores the verifier in localStorage, and redirects to Dropbox's `/oauth2/authorize` endpoint with the challenge.
3. User approves in Dropbox; Dropbox redirects back to the app's URL with a `?code=` query parameter.
4. On page load, the app detects the `code` parameter, exchanges it with Dropbox's `/oauth2/token` endpoint (using the stored verifier), and receives an `access_token` plus a `refresh_token`.
5. Both are stored in localStorage. The URL is cleaned up via `history.replaceState` so the code doesn't linger.
6. When any Dropbox API call returns `401`, the app silently uses the refresh token to get a new access token and retries the call once. This happens transparently.

The refresh token is the only truly permanent credential — losing it means reconnecting Dropbox. Everything else can be re-derived.

### Folder structure

```
Dropbox/Apps/AI_Podcast_Briefing/
├── memory.json          ← full episode archive (scripts, sources, metadata)
├── audio/
│   └── ai-finance-briefing-YYYY-MM-DD.mp3
└── scripts/
    └── ai-finance-briefing-YYYY-MM-DD.txt
```

`memory.json` schema:

```json
{
  "episodes": [
    {
      "date": "2026-11-12",
      "topics": "concatenated 1-liners for the top 3 stories",
      "script": "full spoken script text",
      "sources": [
        { "url": "https://...", "title": "..." }
      ],
      "savedAt": "2026-11-12T08:34:12.000Z"
    }
  ]
}
```

Dropbox keeps 180 days of version history on every file, so accidental overwrites and deletions are recoverable.

---

## 6. Claude model selection (dynamic)

As of November 2026 the app no longer hardcodes model IDs. Instead:

- Two dropdowns in Configure — "Research Model" and "Script Model" — default to "Auto"
- On page load (if the Anthropic API key is present), the app calls `GET https://api.anthropic.com/v1/models` and populates both dropdowns with every model Anthropic currently serves, grouped by line (Haiku → Sonnet → Opus → Fable → Mythos), newest-first within each group
- "Auto" resolves at generation time to the newest model in the appropriate line (Haiku for research, Sonnet for script)
- Specific picks are saved to localStorage and honoured on subsequent runs
- Fallback constants (`claude-haiku-5-5`, `claude-sonnet-5-5`) are used if `/v1/models` is unreachable

**Design intent:** when Anthropic releases a new model, it appears in the dropdown automatically and "Auto" silently switches to it. No code changes required.

The script-writing stage, the grounding check, the correction pass, and the trend-analysis feature all share the same "Script Model" setting.

---

## 7. State & configuration (localStorage keys)

Everything in the browser's localStorage. Nothing server-side.

| Key | Purpose |
| --- | --- |
| `anthropic_key` | Anthropic API key |
| `openai_key` | OpenAI API key |
| `dbx_refresh_token` | Dropbox OAuth refresh token (permanent) |
| `dbx_access_token` | Dropbox OAuth access token (short-lived; auto-refreshed) |
| `dbx_pkce_verifier` | Transient — only during OAuth round-trip |
| `research_model` | `"auto"` or a specific model ID |
| `script_model` | `"auto"` or a specific model ID |
| `focus_areas` | The user's editable focus-areas text |
| `preferred_sources` | List of preferred source URLs/domains |

Episode history itself is **not** cached in localStorage. It is read fresh from Dropbox on every page load, so every device sees the same archive.

---

## 8. UI structure

Single page with three main zones:

1. **Masthead** — title and today's date
2. **Configuration panel** — collapsible, auto-expands when any credential is missing. Contains the two API-key fields, the Dropbox connection, the two model dropdowns, duration, voice, focus areas, preferred sources, and the big **Generate Today's Briefing** button
3. **Output panel** — four tabs (Audio / Script / Sources / Save), populated after a successful generation
4. **Archive panel** — scrollable list of every past episode, read from Dropbox, each row expandable to read the script and copy source URLs
5. **Trend Analysis panel** — ad-hoc analytical query against the full archive (uses the Script Model)

Styling: newspaper-inspired (serif headings, mono labels, generous whitespace, subdued colour palette with a single accent). Large text throughout, for accessibility.

---

## 9. Key design decisions and why

| Decision | Rationale |
| --- | --- |
| **Single HTML file, no framework** | Edits are instant, portable, forkable. No build step, no dependencies to go stale. |
| **No backend** | Zero running cost, no secrets to protect server-side, no deploy pipeline. The user's keys stay in their own browser. |
| **GitHub Pages hosting** | Free, HTTPS by default, auto-deploys on commit, globally cached. |
| **Dropbox App Folder (not Full Dropbox)** | Security — the app can never see or touch anything outside its own folder, even if the token is compromised. |
| **Single `memory.json` file (not one file per episode)** | Simpler. Atomic read/write. Fine up to thousands of episodes. |
| **OAuth PKCE (not short-lived tokens)** | Refresh tokens never expire; access tokens auto-refresh on 401. Connect once per device, forever. |
| **Dynamic model selection via `/v1/models`** | New Anthropic releases roll in automatically. No code changes to stay current. |
| **Hardcoded fallback model constants** | If `/v1/models` is unreachable, the app still works. |
| **Fact-check and correction passes** | The user may repeat these items to colleagues — a confidently stated wrong detail damages credibility. Costs one extra Sonnet call per run; worth it. |
| **Focus areas drive targeted searches** | Generic sweeps miss practitioner-level news. Targeted per-topic searches catch what a single generic pass would overlook. |
| **No emojis in generated content** | Spoken aloud, they add nothing; written, they clutter. |
| **Large body text, newspaper typography** | Owner's eyesight — content must be legible on a phone screen. |

---

## 10. Making changes

The project has no build step. To make a change:

1. Clone or open https://github.com/kendru128/ai-finance-briefing/edit/main/index.html
2. Edit `index.html` directly
3. Commit to `main`
4. Wait ~60 seconds, reload the live URL

For local testing: `python3 -m http.server 8080` in the repo directory, then `http://localhost:8080/`. The OAuth redirect URI is set to the GitHub Pages URL, so Dropbox connection needs to be done via the live app; local testing works for everything else.

If a change is substantial (adding a feature, restructuring), do it on a branch and open a pull request — otherwise commit directly to `main`.

A fresh Claude Code session can pick up this project by reading:
- the GitHub repo (current state of `index.html`)
- this handoff document
- the conversation summary in Claude's memory under `/areas/ai-finance-briefing.md`

---

## 11. Known limitations and ideas not yet implemented

- **Rate limits** — the pipeline has a hardcoded 10-second pause between the research call and the script call to stay under Anthropic's per-minute token limits. Rarely an issue in practice but can bite on back-to-back regenerations.
- **No scheduling** — the app runs only when the owner opens it and clicks Generate. A scheduled run (e.g. 7am daily) would need either a GitHub Action or a Claude scheduled task that drives the browser.
- **No multi-user support** — by design. Everything assumes a single owner.
- **No cost tracking** — the app does not display estimated token usage or cost per run. A rough mental model: one full run with Haiku + Sonnet + grounding + correction + TTS is under £0.20.
- **No offline mode** — requires live internet access to Anthropic, OpenAI, and Dropbox.

---

## 12. Credential inventory (what to have on hand)

To operate the app from a new device:

1. **Anthropic API key** — from https://console.anthropic.com/
2. **OpenAI API key** — from https://platform.openai.com/api-keys
3. **Dropbox** — nothing to copy; click "Connect Dropbox" and complete the OAuth flow once
4. (Optional) A specific preferred model to pin via the dropdowns, otherwise leave on Auto

All three credentials are stored only in the device's own browser localStorage. Clearing site data on https://kendru128.github.io forgets everything and requires reconnection; the Dropbox archive itself is untouched.

---

*End of handoff document.*
