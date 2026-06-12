# Session Handoff — AI Finance Briefing

**For:** the next working session on this app. Attach this plus `INVARIANTS.md` (and the live `index.html` is the source of truth on GitHub `main`).

**One-line state:** the single-file app is healthy and live; the memory/dedup/accuracy fixes and two research-targeting features (focus areas + preferred sources) are all shipped and verified. No known regressions or open bugs.

---

## Where things stand

**Live & working (do not re-derive — see `INVARIANTS.md` for the why and the verify steps):**
- **Non-destructive save** (INV-1) — a failed read aborts the save instead of wiping history. This was the original root-cause bug.
- **Story-level dedup, 14-day hard suppression** (INV-2) — replaced the old weak theme-count nudge that let stories be reworded and repeated.
- **Accuracy / grounding stack** (INV-3) — accuracy mandates in both prompts + a post-script fact-check pass that strips unsupported specifics.
- **Per-story source traceability** (INV-4) — fact-check links in the output and in `memory.json`; the user is the final check.
- **Targeted research, persisted** (INV-5) — focus-areas profile AND, newly shipped this session, **preferred sources**: an in-app editor (one URL/domain per line) normalised to domains and injected as a search-prioritisation block. Both persist in `localStorage`, survive a memory reset.
- **Framing variety** (INV-6) — recent-framings memory, NOT an exact-string blocklist.

**Current file:** `index.html`, ~1,620 lines, single file, GitHub Pages. Architecture unchanged from the original (BYO API keys in `localStorage`; Dropbox via OAuth PKCE for `memory.json` + scripts + audio; pipeline = research[Haiku+web_search] → script[Sonnet] → grounding[Sonnet] → TTS[OpenAI] → non-destructive save).

## How to work on it (carry this discipline forward)

The verification ritual is documented at the top of `INVARIANTS.md` and has reliably caught problems. In short: pull the live file first → edit a local copy → `node --check` the script block → simulate the deterministic memory logic against a real `memory.json` → commit small and single-purpose → after commit, byte-diff (md5) the live file against the build. **Commit mechanism:** the user pastes the file content over `index.html` in the GitHub web editor and commits (the browser-extension file-upload path is blocked; paste-over works cleanly). The assistant then verifies the live raw file matches.

**Session-management approach (decided this session):** the *repo* is the trunk; build sessions are disposable and sequential, briefed from the repo + these docs. Hand off to a fresh session (with an updated handoff) when context gets full or a topic deserves its own thread. Don't try to keep one long "primary" thread alive — durable state lives in the repo's documents, not in any conversation. Reach for **chat, not Cowork**, for this work: it's deliberative and the valuable context lives in the conversation, not in a delegable file-heavy task.

## Open threads / what's likely next

1. **Validate preferred-sources in real use (immediate).** The user will add trusted sites and run a few episodes. The test: do stories actually surface from those domains (check the output panel's source links)? This is domain-bias only — it steers search, it does not fetch specific pages. Decision point it feeds:
2. **Possible follow-up: direct page-fetching** of specific updating pages — only if domain-bias proves insufficient. Strictly better for "catch everything new on this exact page" but heavier (fetch reliability, browser CORS, content extraction). Not built; gated on (1).
3. **Possible: paste-a-lead path** — manual input for a link/snippet the user spotted (e.g. LinkedIn), researched-around and woven into the episode. The human-in-the-loop fallback if targeted research doesn't catch enough. Low-cost, high-value if needed. (LinkedIn-as-direct-source was rejected — credentials/ToS; see `INVARIANTS.md` pending section.)
4. **Tuning the dedup, from live use** — the 14-day `STORY_SUPPRESS_DAYS` window and the story-key matching are the things most likely to need adjustment as real episodes accumulate. If repeats recur, check whether the repeated story's `key` matches what was stored (inconsistent keys are the main way a repeat slips through), and tune key-normalisation before touching the window.
5. **Parked: agent-oriented redesign.** The user paused this. A separate, fuller handoff document exists for that discussion (covers the proposed Coverage / Topic-Checker / Explainability / Longitudinal agents, the hybrid feasibility assessment, and the datastore question). Pick that up only if the user revives it — it's a design discussion, not a build.

## Things to be careful of

- **Treat `INVARIANTS.md` as constraints to design around.** Several invariants look like ordinary code (esp. INV-1's error handling) and are easy to "tidy" into a regression. Read the "trap" line for each before editing nearby.
- **Keep user config out of `memory.json`.** Focus areas, preferred sources, and API keys belong in `localStorage` so a memory reset doesn't wipe them.
- **Don't reach for exact-string blocklists** to fix framing/quality repetition — it just trains the model to paraphrase around them (INV-6).
- **Accuracy has personal stakes** for the user (he may repeat items to colleagues). Don't weaken the grounding pass or the source traceability to save a call or latency.
- **One feature per session, verified before the next.** Small diffs are the regression net.

## Pointers

- Live code: GitHub `kendru128/ai-finance-briefing`, `main`, `index.html` (raw URL is the verification source of truth).
- Memory file: Dropbox `Apps/AI_Podcast_Briefing/memory.json` — the user can upload it on request for grounding analysis (has been done before; very useful for diagnosing dedup/accuracy questions against real data).
- The personal-podcast app repo also stores `/scripts/*.txt` and `/audio/*.mp3` per episode.
