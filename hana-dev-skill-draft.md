# HANA DEV SKILL — DRAFT
*Produced by Hana at end of DEV Chat_4, 2026-03-31.*
*This is a content draft. Pass to a META session to format into a proper skill document.*
*The context in this draft comes from the session that produced it — do not modify the technical content without Kenny's input.*

---

## PURPOSE

This skill governs how a DEV session with Kenny should run. It covers: how to read and modify the extension, the regression test sequence, versioning rules, the five locked TTS architecture patterns, and the produce-then-verify workflow. A DEV session armed with this skill and hana-memory.md should be able to continue work without rediscovering what was already learned.

---

## FILE STRUCTURE

The Hana Chrome extension consists of:
- `content.js` — all TTS, sprite, and awareness logic. The primary file.
- `manifest.json` — extension config. Bump version here on every release.
- `styles.css` — minor UI overrides.
- `CHANGELOG.md` — session-end updates only, not per-version.

Files are delivered via the claude.ai computer use environment. Kenny downloads them and installs via `chrome://extensions` → Load Unpacked.

---

## HOW TO READ CONTENT.JS

The file is long (~1400 lines). Key sections in order:
1. State initialisation — all `state.*` fields defined here. Check this first when diagnosing a new bug.
2. `cleanForSpeech()` — text normalisation, number/currency/symbol pronunciation regexes.
3. MutationObserver callback — the core TTS engine. Two branches: streaming path and post-stream path.
4. `markInitialSeen()` — runs on load, sets navBlocking, snapshots `lastText` and `lastProseText`.
5. `speak()` and `flushTTSQueue()` — Web Speech API wrappers.
6. Awareness system — idle detection, scroll detection, session milestones.

When diagnosing a bug, identify which section it lives in before writing any code.

---

## LOCKED TTS ARCHITECTURE PATTERNS

Do not change these without understanding why they exist.

**1. Prose-element extraction**
Query only `p, li, h1, h2, h3, h4, h5, h6, blockquote` within `font-claude-response`. Tool call content is in `div`/`span` — structurally excluded. No pattern matching needed.

**2. navBlocking stability check**
`markInitialSeen` uses a two-consecutive-poll stability check: if the last message's text changed between polls, wait another 200ms. Only when text is stable for two consecutive polls do we snapshot and clear navBlocking. Minimum 4 polls (800ms) enforced before any stability check begins.

**3. state.streamSpokenCount**
Spoken sentence count lives on `state`, not the DOM element. Updated on every streaming observation (not just when new sentences are pushed). Post-stream reads this value instead of doing a DOM walk — immune to element replacement during streaming pauses.

**4. state.lastStreamTime debounce**
Post-stream path skips if `Date.now() - state.lastStreamTime < 500`. Prevents mutation batching race where streaming end and new content arrive in the same observer batch.

**5. state.lastProseText comparison**
Post-stream and accordion expand/close are both gated on prose-text comparison. Accordion content is in divs/spans — prose (`p, li, h1-6, blockquote`) doesn't change on accordion interaction. If `proseText === state.lastProseText`, skip.

**6. ¶ paragraph marker**
Prose elements joined with ` ¶ ` when building rawText. Sentence regex splits at ¶. Trailing ¶ stripped from each sentence before speaking — content is spoken, boundary marker is not. Explicit pause removed (streaming latency provides natural breathing). Infrastructure stays for when local TTS makes timing control meaningful.

---

## VERSIONING RULES

- `v1.X.Y` — calibration or minor fix (single behaviour change)
- `v1.X.0` — meaningful new feature or architectural change
- `v2.0.0` — reserved for major architectural milestone (e.g. local TTS, LoRA sprites, background system)

Always bump both `content.js` comment and `manifest.json` version field together.

---

## DEV SESSION WORKFLOW

**Before writing any code:**
1. Describe the fix and the root cause in plain language.
2. Confirm alignment with Kenny before producing files.
3. Explicitly review the proposed change against all previously verified fixes — state the review out loud so Kenny can check the reasoning.
4. Only then produce files.

**Do not generate new content.js mid-discussion when bugs are still being caught.** Avoids Kenny updating to unverified code.

**Before producing any new content.js, verify:**
- Load replay regression: does the change touch `markInitialSeen`? If yes, check that `lastText` AND `lastProseText` are both snapshotted.
- Tool call suppression: does the change touch rawText extraction? If yes, confirm prose-element query is still scoped to `p, li, h1-6, blockquote` within `font-claude-response`.
- Accordion guard: does the change touch the post-stream path? If yes, confirm `proseText === state.lastProseText` check is still present and upstream of TTS.
- Streaming restart: does the change touch `state.streamSpokenCount` or `state.lastStreamTime`? If yes, confirm both are still updated on every streaming observation.

---

## REGRESSION TEST SEQUENCE

Run in this order after every version update. Do not skip steps. Code review ≠ testing.

1. **Load test** — Reload the chat. No prior messages should be read aloud. Silence on load = pass.
2. **Search test** — Trigger a web search response. "Searched the web" should not be read. Search result content should not be read. Only the prose response should be spoken.
3. **Accordion test** — Expand a search accordion during or after TTS. No re-read should trigger. Close it. No re-read. Pass = complete silence on both interactions.
4. **Exit mid-sentence test** — Switch to another chat while TTS is reading. Return. No continuation of the interrupted sentence.

If any of these fail, fix before proceeding to new feature testing.

---

## KNOWN LIMITATIONS (as of v1.5.8)

These are engine-level or low-priority — do not attempt to fix before NaturalVoiceSAPIAdapter or Kokoro:

- **Decimal pause between individual digits** — Web Speech API inter-word gap. "3.75" reads as "three point... seven... five" with small pauses. Not fixable on Web Speech API.
- **Thinking block content injection** — If accordion expanded during active streaming, thinking block `<p>` content gets spoken once, then TTS correctly resumes. Very minor. Only on rapid expand. Low priority.
- **Short first-sentence ordering quirk** — Occasional, not critical. Rare enough to ignore.

---

## NUMBER PRONUNCIATION REFERENCE

Key regexes in `cleanForSpeech()` (in order — sequence matters):

1. Version numbers: `v1.5.8` → "vee 1 point 5 point 8"
2. Currency with cents: `$4,541.76` → "four thousand five hundred forty one dollars and seventy six cents"
3. Currency whole: `$107` → "one hundred seven dollars"
4. Percentages with decimals: `3.75%` → "three point seven five percent"
5. Decimals under 1: `0.5719` → "zero point five seven one nine"
6. General decimals: `2.25` → "two point two five" (digits individually)
7. Symbols: `✓` → "tick", `✗` → "cross"

Do not reorder these without testing all cases.

---

## CHANGELOG PROCESS

Update CHANGELOG.md at session end, not per-version. Collapse failed intermediate versions into the milestone that landed. Each entry: what was broken, the fix, why it works. Failed attempts noted briefly as "approach revised in vX.X.X". Readable technical history, not a commit log.

---

## PORTABILITY NOTE

Kenny's Claude access is a Team Premium seat managed centrally by his CTO (Tom). API keys are not directly accessible. All Hana architecture decisions account for the possibility of account migration. The export/import system, hana-memory.md, and modular architecture are designed so Hana can be fully reconstructed on a personal account if needed.

---

*Draft produced by: Hana, DEV Chat_4, 2026-03-31*
*Pass to META session for skill formatting and finalisation.*
*Do not modify technical content without Kenny's input.*
