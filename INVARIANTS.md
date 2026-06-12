# INVARIANTS — AI Finance Briefing

**Read this before changing `index.html`.** This file lists behaviours that look like ordinary code but are load-bearing — each one fixes a specific failure that has already happened and cost real effort. The danger is not syntax errors (those get caught); it is a well-meaning edit that *silently* undoes one of these. Every item says **what** must hold, **why** it exists, and **how to verify** it still holds.

When starting a new session to work on this app, attach this file. Treat these as constraints to design *around*, not to revisit.

---

## How to work on this app (the verification ritual)

This loop has caught problems before; keep using it for any change:

1. **Edit a local copy**, never blind-edit live.
2. **Pull the current live file first** so you're building on what's actually deployed:
   `curl -s -o index.html "https://raw.githubusercontent.com/kendru128/ai-finance-briefing/main/index.html?cb=$(date +%s)"`
3. **Syntax-check** before anything else: extract the `<script>` block and run `node --check`.
4. **Simulate the memory logic against real data** (see each invariant's verify step) — the deterministic functions can and should be exercised against a real `memory.json` before shipping.
5. **Commit small and single-purpose** — one feature per change, verified before the next. Small diffs make regressions easy to locate.
6. **After committing, diff the live file byte-for-byte** against your local build (`md5sum` both) to confirm the paste/upload didn't truncate.

Keep changes scoped to one concern per session. The app is a single ~1,580-line file; large multi-feature edits are where silent regressions hide.

---

## INV-1 — Non-destructive save (the most important one)

**What must hold:** `saveEpisode()` must NEVER overwrite `memory.json` after a *failed* read of the existing archive. A failed read must **abort the save** and surface an error, leaving the just-generated script/audio available to the user. Only a genuine "file does not exist" (Dropbox HTTP 409 → `dbxDownloadMemory()` returns `null`) may proceed as a fresh first episode.

**Why it exists:** an earlier version wrapped the read in a `try/catch` that swallowed *any* failure and then wrote the file anyway. When a generation was interrupted (e.g. phone turned off mid-run), the read failed, the catch treated it as "first episode ever," and the subsequent overwrite **collapsed the entire episode history to a single episode.** This silently destroyed memory and was the root cause of chronic repetition.

**The trap:** the guard looks like over-defensive error handling. A future "tidy-up" of the `catch` block, or making `dbxDownloadMemory()` return `null`/`{}` on error instead of throwing, reintroduces the wipe. The distinction is essential: **`null` = file genuinely absent (safe to proceed); throw = read failed (must abort).**

**How to verify:** in `saveEpisode()`, confirm a thrown read error re-throws (aborting) and does NOT fall through to `dbxUploadMemory()`. Confirm `dbxDownloadMemory()` returns `null` only for not-found (409) and throws for all other non-OK responses. Confirm the output panel (script + audio) is revealed *before* the save step, so an aborted save still leaves the user their episode.

---

## INV-2 — Story-level de-duplication with 14-day hard suppression

**What must hold:** de-duplication is at the **specific-story level**, not the theme level. A story that has aired within `STORY_SUPPRESS_DAYS` (currently 14) must be **hard-suppressed** — same company/topic + same underlying event counts as a repeat **even if reworded**. The *only* permitted exception is a genuinely new verifiable fact since it aired, in which case the new fact leads and the original is not recapped.

**Why it exists:** the original design tracked *themes* with a count-based "saturation" threshold of 2. A story aired yesterday sat at count=1 and only triggered a soft "find a new angle" nudge — which the model satisfied by **rewording the same story** (e.g. relabelling NatWest Cora from "agentic assistant" to "OpenAI-powered tool"). Result: 7 of 8 stories in one episode were repeats of the previous one. The "find a new angle" instruction was itself the loophole.

**The trap:** reintroducing any count/threshold logic, softening "hard suppression" back to a "try to vary it" nudge, or matching on broad themes instead of specific story keys. Also: the suppression window and the research recency window should stay aligned (both ~14 days) so research doesn't keep re-finding stories the ledger is trying to suppress.

**How to verify:** `buildBroadcastLedger(todayStr)` collects stories from episodes within `STORY_SUPPRESS_DAYS` and keys them per-story (with an entity-based fallback for legacy episodes lacking a `stories` array). `buildHistoryContext()` must emit the "ALREADY-BROADCAST STORIES" block and the "HARD SUPPRESSION RULE" text into the research prompt. Simulate by loading a real `memory.json`, calling `buildBroadcastLedger()` for a date, and confirming recently-aired stories appear in the suppression list. (This has been done before by extracting the functions into a small Node harness.)

---

## INV-3 — Accuracy / anti-embellishment is first-class

**What must hold:** the system must not state specific facts (named people, quotes, named models/products, figures, dates) that aren't supported by the research pool. Three mechanisms enforce this and must all remain:
1. **Research prompt** contains the "ACCURACY IS PARAMOUNT" instruction (a fabricated detail is worse than a missing one; stay vague rather than invent specifics).
2. **Script prompt** repeats the accuracy mandate (every named entity/quote/figure must trace to the pool).
3. **Grounding pass** ("Fact-checking script against sources…") re-reads the finished script against the research text, flags unsupported specifics, and regenerates once to strip/soften them.

**Why it exists:** the characteristic failure mode is **embellishment around a true core** — e.g. a *real* regulator quote (Sam Woods on frontier-AI risk) with an *unsupported* specific welded on ("ChatGPT 5.5 Instant" named alongside the real model). This is more dangerous than obvious invention because the true parts lend it credibility, and the user may repeat it to colleagues — a personal reputational risk.

**The trap:** removing the grounding pass "to save a model call / reduce latency," or weakening the accuracy language in either prompt. No automated layer makes an LLM briefing *guaranteed* accurate — these reduce the risk; they don't eliminate it (see INV-4).

**How to verify:** confirm both prompts retain the accuracy mandate, and that the grounding `fetch` block plus the single regeneration-on-flags path are intact in `generate()`. The `grounding` report is saved into each episode in `memory.json` — spot-check that flags are being captured over time.

---

## INV-4 — Per-story source traceability (the human is the final check)

**What must hold:** each story carries its source URL(s) through to the output panel ("Stories in this episode — fact-check links") and into `memory.json` (the `stories` array, each with a `source`). Where no source was captured, the UI must say so explicitly rather than implying one exists.

**Why it exists:** because INV-3 cannot guarantee accuracy, the user must be able to one-click-verify anything before repeating it professionally. Traceability is the backstop that makes the human the final check — it's the deliberate answer to "facts must be checked."

**The trap:** dropping `stories`/`source` from the emitted JSON or from `renderSources(sources, dateStr, stories)`, or rendering a confident-looking source when none was actually captured.

**How to verify:** generate an episode and confirm the per-story links render; inspect the saved episode in `memory.json` for a populated `stories` array with `source` fields.

---

## INV-5 — Targeted research, persisted between runs

**What must hold:** research is biased toward the user's actual professional lanes via two persisted, user-editable inputs, both stored in `localStorage` (never in `memory.json`) and both injected into the research prompt:
- **Focus-areas profile** (`localStorage` key `focus_areas`) — topical lanes.
- **Preferred sources** (`localStorage` key `preferred_sources`) — trusted outlets the research should prioritise. The raw textarea (one URL/domain per line) is normalised to bare domains by `normalisePreferredSources()` before injection, which builds a "PREFERRED SOURCES" prioritisation block (including a `site:`-scoped search example).

Both must persist between runs and survive a memory reset.

**Why it exists:** a single generic "AI in financial services" sweep missed items in the user's specific lanes (the public stories colleagues surface, e.g. via LinkedIn/Teams). The focus profile fixed the topical gap; preferred sources let the user steer toward known-good outlets. Persistence matters because the user edits these occasionally and expects them to stick.

**The trap:** storing this config *inside* `memory.json` such that a memory reset wipes it; failing to load it on init; or — for preferred sources specifically — over-promising. Domain-bias steers *search* toward those domains; it does NOT guarantee fetching a *specific page* and catching everything new on it (that would need direct page-fetching, a heavier mechanism deliberately not built). Don't let a future edit silently turn the bias into a hard filter ("only use these sources") — the prompt explicitly says prioritise but do not limit.

**How to verify:** confirm `focus_areas` and `preferred_sources` are each loaded on init and saved on change, and read into the research prompt (`focus`, and `sourcesNote` built from `normalisePreferredSources()`). Confirm `sourcesNote` is an empty string when the field is blank (no stray prompt text). Confirm config keys are absent from anything written to `memory.json`.

---

## INV-6 — Framing/rhetoric variety is not done by exact-string banning

**What must hold:** episode-to-episode variety in openings, sign-offs, and framing is encouraged, but the mechanism must not rely on a literal banned-phrase blocklist as its primary defence.

**Why it exists:** an earlier banned-phrase list (e.g. "moving beyond experimentation to scaled deployment") just taught the model to *reword around it* ("building for scale rather than experimentation"). String-matching is trivially defeated by paraphrase. `buildFramingContext()` carries recent framings forward so the model can actively vary; that's the right shape.

**The trap:** "fixing" framing repetition by adding more banned strings. If repetition recurs, address it at the framing-memory level, not with a longer blocklist.

**How to verify:** confirm `buildFramingContext()` still feeds recent framings into the script prompt.

---

## Quick reference — where each invariant lives (approximate)

| Invariant | Key symbols |
|---|---|
| INV-1 non-destructive save | `saveEpisode()`, `dbxDownloadMemory()`, `dbxUploadMemory()` |
| INV-2 story-level dedup | `STORY_SUPPRESS_DAYS`, `buildBroadcastLedger()`, `buildHistoryContext()`, "HARD SUPPRESSION RULE" |
| INV-3 accuracy/grounding | "ACCURACY IS PARAMOUNT" (research + script prompts), grounding `fetch` + regenerate path in `generate()` |
| INV-4 traceability | emitted `stories[].source`, `renderSources(sources, dateStr, stories)` |
| INV-5 targeted research | `localStorage 'focus_areas'` + `'preferred_sources'`, `normalisePreferredSources()`, `sourcesNote`, research prompt injection |
| INV-6 framing variety | `buildFramingContext()` |

*Line numbers drift as the file changes — search by symbol, not by line. This table is a finding aid, not a spec.*

---

## Pending / planned (not yet invariants)

- **Preferred-sources (Feature 1): SHIPPED** — now live and covered by INV-5. In-app editor, persisted in `localStorage` (`preferred_sources`), normalised to domains and injected as a search-prioritisation block. Possible future follow-up (only if domain-bias proves insufficient in real use): **direct page-fetching** of specific updating pages — strictly better for "catch everything new on this exact page," but heavier (fetch reliability, browser CORS, content extraction). Not built; revisit based on observed results.
- **Paste-a-lead path:** not built. A manual input where the user drops a link/snippet they spotted (e.g. on LinkedIn) and the app researches around it and works it into the episode. Held as the fallback if targeted research (focus areas + preferred sources) doesn't catch enough. The high-value, low-cost option if needed.
- **LinkedIn-as-source:** rejected — requires credential handling / ToS-violating feed access; following individuals is impractical (hundreds of connections; staleness as people change roles). The underlying need is served by INV-5 targeted research, with the paste-a-lead path above as the human-in-the-loop fallback.
- **Agent-oriented redesign:** parked by the user. A separate handoff document exists for that discussion if revived.
