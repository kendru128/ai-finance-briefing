# AI Finance Briefing — Handoff Document

**Last updated:** 13 November 2026 (morning — model-aware thinking config)
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
│                  Browser (index.html, ~1,850 lines)          │
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

**All five Claude calls in the app** (the four stages below plus trend analysis) turn extended thinking off via the model-aware `thinkingForModel()` helper. See section 6.1 for why and how. All text extraction from Claude responses filters by `type === 'text'` and joins — never assumes `.content[0].text`, because newer models may emit tool-use or thinking blocks first.

### Stage 1 — Research (Claude Haiku + web search)

A single Claude call with the `web_search_20250305` tool. Prompt asks for a broad, well-sourced pool of 10–12 candidate stories from the last 7 days, covering four fixed angles (AI in UK/EU financial services, regulatory activity, research breakthroughs, agentic AI deployments) plus **targeted searches for every item in the user's Focus Areas list**. The focus-areas pass is critical — the generic sweep misses most practitioner-level news.

Each candidate must carry: headline, what is genuinely new, a short canonical tag, source publication AND full URL, publication date, and any hard deadline within 7 days.

Hard rules in the prompt: don't fabricate figures, dates, names, products or quotes; cite every claim; suppress stories already aired (passed in as `historyNote`) unless there's genuinely new information.

**Fail-fast guard:** after the research call, the code counts text blocks and `web_search_tool_result` blocks. If the text returned is under 500 characters, the pipeline aborts with a clear error and tells the user to pin a specific Haiku model rather than generating a holding-script apology downstream. The full response object is logged to the browser console (`[Research] model, stop_reason, text blocks, web_search results, usage, full response`) so any future weirdness can be diagnosed from DevTools.

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

### OAuth 2.0 authorisation-code flow (with offline access)

Dropbox no longer offers permanent "generated" tokens for new apps — all access tokens are short-lived (~4 hours) unless obtained via the OAuth flow with `token_access_type=offline`, which returns a long-lived **refresh token** that doesn't expire.

The app implements a two-step authorisation-code flow using the App Key + App Secret (classic OAuth 2.0 — not PKCE):

1. In the Configure panel the user pastes their Dropbox **App Key** and **App Secret** (these are stored in localStorage, see section 7) and clicks **1. Authorise with Dropbox →**. The app opens Dropbox's `/oauth2/authorize` in a new tab, with `client_id=<App Key>`, `response_type=code`, and `token_access_type=offline`.
2. The user approves in Dropbox, then copies the authorisation code Dropbox shows them.
3. The user pastes the code into the **Authorisation Code** field and clicks **2. Connect**.
4. The app POSTs to Dropbox's `/oauth2/token` with `grant_type=authorization_code`, `code`, `client_id` and `client_secret`, and receives an `access_token`, a `refresh_token`, and `expires_in`. All three are persisted to localStorage.
5. Subsequent API calls use `getAccessToken()`, which returns the cached access token if it's still valid, or hits `/oauth2/token` with `grant_type=refresh_token` to mint a new one. This refresh is automatic and transparent to the rest of the app.

The refresh token is the only truly permanent credential — losing it means reconnecting Dropbox. The App Key and App Secret are also permanent but are less sensitive on their own (they can't access the account without the refresh token).

**Why this rather than PKCE:** an earlier version used PKCE, which avoids storing the App Secret in the browser. In practice the refresh step kept breaking because the token endpoint rejected the refresh without a client_secret, so we switched to the standard flow. Since every credential is already stored only in the user's own localStorage on their own device, holding the App Secret there as well is an acceptable trade-off for this single-user app. If this ever becomes a shared app, revisit.

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

### 6.1 Extended thinking (model-aware configuration)

Newer Claude models (Haiku 5.5 / Sonnet 5.5 and beyond) default to running an internal "thinking" reasoning phase. Those thinking tokens come out of the same `max_tokens` budget as the actual answer, and **turning thinking off requires different values for different model families** — a fact we learned the hard way across two days.

**What went wrong on 12 Nov 2026:** after the Auto-upgrade to Haiku 5.5, the research call began returning empty text. Direct testing showed Haiku 5.5, given this app's research prompt with `max_tokens: 3500` and `web_search_20250305`, was running 12 searches, then spending **3,739 tokens on an internal thinking block** before starting to write text — hitting `stop_reason: max_tokens` with zero text blocks produced. We set `thinking: { type: 'disabled' }` on all five Claude calls and the research call recovered.

**What went wrong on 13 Nov 2026:** the fix broke the SCRIPT call. Sonnet 5.5 began returning a 400 error: *"To turn thinking off on this model, send `thinking: {'type': 'between_tools'}` instead of `{'type': 'disabled'}`. The model does not think before responding. The short updates it writes between tool calls come back as thinking blocks."* So the off-equivalent differs per model family.

**Verified test matrix (direct API calls, 13 Nov 2026):**

| Model | `{type:'disabled'}` | `{type:'between_tools'}` | Omitted |
| --- | --- | --- | --- |
| **claude-haiku-5-5** | ✅ works, 0 thinking tokens | ❌ 400 "not supported" | works but burns ~500 thinking tokens |
| **claude-sonnet-5-5** | ❌ 400 "send between_tools instead" | ✅ works, no thinking emitted | works but emits a thinking block |

**Fix:** a model-aware helper function picks the right value per model family:

```js
function thinkingForModel(modelId) {
  if (/^claude-haiku-[5-9]/.test(modelId))                         return { type: 'disabled' };
  if (/^claude-(sonnet|opus|fable|mythos)-[5-9]/.test(modelId))    return { type: 'between_tools' };
  return undefined;  // pre-5.x models — field omitted by JSON.stringify
}
function thinkingForResearch() { return thinkingForModel(resolveResearchModel()); }
function thinkingForScript()   { return thinkingForModel(resolveScriptModel()); }
```

All five Claude calls pass through `thinkingForResearch()` or `thinkingForScript()`, so a user pinning any specific model from the dropdown (or a future Auto-upgrade) gets the right value automatically. `JSON.stringify` drops keys whose value is `undefined`, which cleanly handles pre-5.x models that predate the parameter entirely.

**End-to-end verification before deployment** (13 Nov 2026, same prompts the app actually uses):

- Research (Haiku 5.5, `disabled`): 6,782 chars in 17.9 s, `end_turn`
- Script (Sonnet 5.5, `between_tools`): 4,064 chars in 11.1 s, `end_turn`

**Why disabling thinking is the right trade-off for this app:**

- *Research:* a search-and-report task with highly prescriptive instructions — thinking adds nothing
- *Script:* the prompt already encodes curation criteria, tone, accuracy rules, output format — thinking would mostly duplicate what the prompt dictates
- *Grounding / correction / trend:* mechanical or well-scoped tasks — no observed quality cost

**If future quality issues suggest re-enabling it selectively** (most likely script-writing or trend-analysis), note that `{type:'enabled', budget_tokens:N}` is **also rejected** by 5.x models — they want `{type:'adaptive'}` with `output_config.effort`. Test the new combination against both Haiku and Sonnet directly before shipping. Do NOT re-enable thinking on the research call (the one that uses `web_search`) without very carefully raising `max_tokens`.

---

## 7. State & configuration (localStorage keys)

Everything in the browser's localStorage. Nothing server-side.

| Key | Purpose |
| --- | --- |
| `anthropic_key` | Anthropic API key |
| `openai_key` | OpenAI API key |
| `dbx_app_key` | Dropbox App Key (public client ID) |
| `dbx_app_secret` | Dropbox App Secret (used in the token-refresh call) |
| `dbx_refresh_token` | Dropbox OAuth refresh token (permanent) |
| `dbx_access_token` | Dropbox OAuth access token (short-lived; auto-refreshed) |
| `dbx_token_expiry` | ms-since-epoch timestamp for when the access token stops being valid |
| `research_model` | `"auto"` or a specific model ID |
| `script_model` | `"auto"` or a specific model ID |
| `focus_areas` | The user's editable focus-areas text |
| `preferred_sources` | List of preferred source URLs/domains |

Episode history itself is **not** cached in localStorage. It is read fresh from Dropbox on every page load, so every device sees the same archive.

---

## 8. UI structure

Single page with five main zones:

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
| **OAuth refresh-token flow (not short-lived generated tokens)** | The offline-access refresh token doesn't expire; access tokens are minted on demand from it and cached until `expires_in` runs out. Connect once per device, forever. Uses App Key + App Secret (classic auth-code flow) rather than PKCE — see section 5 for why. |
| **Dynamic model selection via `/v1/models`** | New Anthropic releases roll in automatically. No code changes to stay current. |
| **Hardcoded fallback model constants** | If `/v1/models` is unreachable, the app still works. |
| **Fact-check and correction passes** | The user may repeat these items to colleagues — a confidently stated wrong detail damages credibility. Costs one extra Sonnet call per run; worth it. |
| **Focus areas drive targeted searches** | Generic sweeps miss practitioner-level news. Targeted per-topic searches catch what a single generic pass would overlook. |
| **Extended thinking disabled on all Claude calls, via a model-aware helper** | Newer models (5.5+) default to internal reasoning that eats `max_tokens`. Different model families want different values to turn it off (Haiku: `disabled`; Sonnet/Opus/Fable/Mythos: `between_tools`). A single `thinkingForModel(id)` helper picks the right one per call. See section 6.1. |
| **Safe text-block extraction (filter by type)** | Response `.content[0]` is not guaranteed to be a text block in newer models. All text extraction uses `.filter(b => b.type === 'text').map(b => b.text).join('\n')`. |
| **Fail-fast if research pool is empty** | Rather than silently generating a holding-script apology when research returns <500 chars, the pipeline stops with a clear error and tells the user how to recover. |
| **Console diagnostics on research call** | Every research call logs model, `stop_reason`, text-block count, web-search-result count, token usage, and the full response object. Future weirdness can be diagnosed from DevTools without redeploying. |
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
3. **Dropbox App Key AND App Secret** — from the app's page in the [Dropbox App Console](https://www.dropbox.com/developers/apps/info/c8qhhuxt5z26zbn). After pasting both into the Configure panel, click **1. Authorise with Dropbox →**, approve in the tab that opens, copy the authorisation code back into the app's **Authorisation Code** field, and click **2. Connect**. Section 5 has the full flow.
4. (Optional) A specific preferred model to pin via the dropdowns, otherwise leave on Auto

All credentials are stored only in the device's own browser localStorage. Clearing site data on https://kendru128.github.io forgets everything and requires the full Dropbox authorisation again; the Dropbox archive itself is untouched.

---

## 13. Troubleshooting notes

A short field guide to the kinds of things that have broken this app in the past, and how to diagnose them.

### "Holding script" output (12 Nov 2026 incident)

**Symptom:** script begins with an apology about the candidate pool being empty, no real news covered.

**Root cause:** the research step is producing no text (even though the HTTP call succeeds). Caused by newer Claude models burning the `max_tokens` budget on internal thinking before writing any output. See section 6.1.

**How to confirm:** open the browser DevTools console before clicking Generate. The research call logs `text blocks: 0` and `stop_reason: max_tokens`, with high `thinking_tokens` in the usage object.

**Current protection:** the fail-fast guard aborts with a clear error, so a holding script should no longer be produced silently. If the research call ever starts failing again:
- Open Configure, pin a specific older Haiku (e.g. `claude-haiku-4-5-20251001`) from the Research Model dropdown
- Or re-check that `thinkingForResearch()` is still being called in the research request body, and that `thinkingForModel()`'s regex still matches the current model's ID
- Or raise `max_tokens` on the research call (currently 3,500)

### "undefined is not an object (evaluating 'rawOutput.match')" (12 Nov 2026 incident)

**Root cause:** newer models can emit a non-text block first in the response (thinking, tool-use), so `d.content[0].text` is undefined.

**Protection:** every text extraction site now filters by `type === 'text'` and joins. If a similar error shows up on a different field, apply the same pattern.

### "thinking.type.X is not supported for this model" (13 Nov 2026 incident)

**Symptom:** a Claude call fails with a 400 error complaining about the `thinking.type` value — either "`between_tools` is not supported" (Haiku) or "send `between_tools` instead of `disabled`" (Sonnet).

**Root cause:** different Claude model families accept different `thinking.type` values. There is no universally-accepted value for turning thinking off.

**Protection:** the `thinkingForModel(modelId)` helper (section 6.1) returns the right value per model family. When adding a NEW model family (Fable, Mythos, a hypothetical new line), check its 400 error messages against both `disabled` and `between_tools` first, then extend the helper's regex.

**Lesson learned — the methodology:** when a shared Anthropic parameter changes semantics across models, always test the full matrix (every value × every model you care about) BEFORE deploying a fix. Yesterday's blanket `disabled` worked only because yesterday's isolated test happened to be against Haiku — Sonnet would have 400'd on the same value immediately.

### Dropbox token expired (earlier in Nov 2026)

**Symptom:** Dropbox calls fail with 401.

**Root cause:** Dropbox tokens expire after ~4 hours.

**Current protection:** OAuth offline-access flow grants a refresh token that doesn't expire; access tokens are refreshed automatically. If this chain breaks (refresh token revoked, say), reconnect via Configure (section 5 has the full re-authorisation flow).

### Script truncated or missing end

**Possible cause:** `max_tokens` on the script call (currently 4,000) is too tight for a long briefing plus the JSON metadata block, especially at the 8-minute duration.

**Fix:** raise `max_tokens` on the script call from 4,000 to (say) 6,000. Each duration preset (3/5/8 min) targets 450/750/1200 words, which in tokens is roughly 600/1000/1600 — plus the JSON block (~300 tokens) and some headroom. If you change this, also raise `max_tokens` on the correction pass (also 4,000) by the same amount, since it rewrites the full script.

### Diagnosing any new Claude failure

1. Open DevTools Console before clicking Generate
2. Watch the `[Research]` log line that appears after the research call — it has `stop_reason`, text-block count, search-result count, token usage
3. The full response object is attached to that log line — expand it to inspect every content block
4. If the failure is in a later stage, add a similar `console.log` near the relevant `await r.json()` and redeploy

---

## 14. Change log highlights

Only the changes with lasting architectural impact. Small refinements aren't listed.

| Date | Change |
| --- | --- |
| 2026-11-13 (morning) | Made the `thinking` parameter model-aware via a new `thinkingForModel(id)` helper. Haiku takes `disabled`, Sonnet/Opus/Fable/Mythos take `between_tools`, pre-5.x omit. Fixes the Sonnet 5.5 regression where `disabled` is rejected. All 5 Claude calls updated. |
| 2026-11-12 (evening) | Disabled extended thinking on all Claude calls (initially with a blanket `disabled` — later refined); added fail-fast guard for empty research; added console diagnostics; switched to safe text-block extraction (filter by `type`). |
| 2026-11-12 (morning) | Added dynamic model selection via `/v1/models` — "Auto" picks the newest Haiku/Sonnet without code changes. |
| 2026-11 (earlier) | Added "Save to Dropbox" button for MP3 + script to `/audio/` and `/scripts/` folders. Added Configure collapsible panel. |
| 2026-10 (approx) | Switched Dropbox auth from short-lived generated tokens to OAuth 2.0 authorisation-code flow with a refresh token (initially PKCE; later moved to App Key + App Secret because the refresh step was failing without `client_secret`). Connect-once-per-device. |
| 2026-10 (approx) | Initial deploy on GitHub Pages at https://kendru128.github.io/ai-finance-briefing/ |

---

*End of handoff document.*
