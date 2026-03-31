# Hana — Claude Companion
## Changelog

---

## v1.5.8 — 31 March 2026

**Post-stream debounce + always-current streamSpokenCount**

**What was broken:** TTS looped at mid-list items (e.g. "No sentence restarts ✓ or ✗"). Root cause: mutation batching race. The DOM mutation adding a new list item and the mutation ending streaming arrived in the same MutationObserver batch. The callback saw `streaming.length === 0`, took the post-stream path, but `streamSpokenCount` hadn't been updated for the newest sentence yet — still at the previous count. `alreadySpoken` was stale → `speak(ttsText, true)` → full restart.

**Fix:**
1. `state.lastStreamTime = Date.now()` — updated on every streaming observation.
2. Post-stream path: `if (Date.now() - state.lastStreamTime < 500) return` — skip if streaming was active within 500ms.
3. `state.streamSpokenCount = spokenTexts.size` moved outside `if (pushed > 0)` block — always kept current, not just when new sentences were pushed.

---

## v1.5.7 — 31 March 2026

**state.streamSpokenCount replaces DOM walk**

**What was broken:** TTS restarted from the beginning of the response when streaming briefly paused between list items. Root cause: when streaming pauses, claude.ai creates a new DOM element for the `data-is-streaming="false"` state. The post-stream path looked for `_spokenTexts` on the new element — not found → `alreadySpoken = 0` → `speak(ttsText, true)` → full restart.

**Fix:** Store spoken sentence count on `state.streamSpokenCount` instead of on the DOM element. Reset on new streaming element. Post-stream reads from state — completely immune to DOM element replacement.

---

## v1.5.6 — 31 March 2026

**Tick/cross pronunciation + remove over-aggressive search strip**

**What was broken:**
1. ✓ and ✗ symbols spoken as silence.
2. "Searched the web" in actual response prose was being filtered out — belt-and-suspenders strip predated prose-element extraction and was too aggressive.

**Fix:**
1. Added `✓ → ' tick '` and `✗ → ' cross '` to `cleanForSpeech`.
2. Removed the strip from both streaming rawText and post-stream ttsText.

---

## v1.5.5 — 31 March 2026

**Strip trailing ¶ instead of skipping — list items now spoken**

**What was broken:** List items silently skipped. Sentences ending in ¶ (paragraph boundary marker) had the whole sentence dropped — `clean.endsWith('¶') → return` discarded the content.

**Fix:** `let clean = s.trim().replace(/¶+$/, '').trim()` — strip the trailing ¶ and speak the remaining content. Only a bare `¶` with no content is skipped.

---

## v1.5.4 — 31 March 2026

**DOM stability check in markInitialSeen**

**What was broken:** Load replay returned. `markInitialSeen` cleared navBlocking after 800ms but the DOM hadn't finished rendering. Post-stream fired, text didn't match stale snapshot → TTS read the whole prior message.

**Fix:** Stability check — two consecutive polls (400ms) must return identical last-message text before navBlocking clears. DOM still rendering means text changes poll-to-poll.

---

## v1.5.3 — 31 March 2026

**Remove explicit paragraph pause**

Explicit 160ms paragraph pause felt like 3–5 seconds because streaming latency already fills the gap. Removed. The ¶ marker infrastructure stays for when local TTS (Kokoro/NaturalVoiceSAPIAdapter) makes precise timing meaningful.

---

## v1.5.2 — 31 March 2026

**Paragraph pause reduction (superseded by v1.5.3)**

Reduced explicit pause from 320ms to 160ms. Still felt too long — underlying issue was streaming latency, not the explicit pause value. Approach revised in v1.5.3.

---

## v1.5.1 — 31 March 2026

**Snapshot lastProseText on load**

**What was broken:** Load replay. `markInitialSeen` only snapshotted `state.lastText`, not `state.lastProseText`. When navBlocking cleared, the post-stream observer compared proseText against `''` → mismatch → TTS fired and read the prior message.

**Fix:** In `markInitialSeen`, also snapshot `state.lastProseText` from the last message's prose elements alongside `state.lastText`.

---

## v1.5.0 — 31 March 2026

**startsWith dedup + paragraph breathing infrastructure**

**What was broken:**
1. Sentence restarts when streaming extended a sentence mid-generation. Fixed 35-char prefix dedup too short for short sentences.
2. No natural breathing between prose blocks.

**Fix:**
1. `startsWith` dedup: find any prior spoken sentence that is a prefix of the new sentence. Queue only the tail. Works for sentences of any length.
2. Paragraph breathing: prose elements joined with ` ¶ ` marker. Sentence regex splits at ¶. Explicit pause added — later revised in v1.5.2–v1.5.3.

---

## v1.4.9 — 31 March 2026

Individual digits in decimals spaced explicitly to ensure TTS reads each digit separately. Calibration of v1.4.3 approach.

---

## v1.4.8 — 31 March 2026

**Prose-text comparison for all accordion types**

Accordion close (collapsing) caused re-read — growth ratio guard only fired on expansion. Fixed: replace growth ratio with prose-text comparison. `proseText === state.lastProseText` → skip, regardless of direction.

---

## v1.4.7 — 31 March 2026

**Accordion expansion guard**

Expanding search accordion caused re-read. Initial growth ratio fix was fragile. Revised to prose-text comparison in this version, extended to close direction in v1.4.8.

---

## v1.4.3–v1.4.6 — 31 March 2026

**Number pronunciation suite**

- v1.4.3: Decimal digits individually — `3.75` → "three point seven five"
- v1.4.4: Currency — `$107.53` → "one hundred seven dollars and fifty three cents"
- v1.4.5: Tail queuing on prefix match — nothing lost when currency sentence extends mid-stream
- v1.4.6: Minimum 800ms navBlocking — tool call elements appear at 400–600ms; 800ms guarantees they've arrived

---

## v1.4.1–v1.4.2 — 31 March 2026

**Prose-element extraction (architectural milestone)**

**What was broken:** TTS still read tool call labels, summaries, and search results despite DOM sanitisation in v1.3.x.

**The architectural fix (v1.4.1):** Query only `p, li, h1, h2, h3, h4, h5, h6, blockquote` within `font-claude-response`. Tool call content is in `div`/`span` — structurally excluded without any class-name matching.

v1.4.2: navBlocking stability improvements.

---

## v1.3.4 — 31 March 2026

**No fallback when font-claude-response absent + strip citations**

Removed `|| clonedStream` fallback — if `font-claude-response` isn't present (tool call still running), return early rather than reading the whole clone. Added `a[href]` and `sup` to removal list to strip citation source names.

---

## v1.3.3 — 31 March 2026

**Target font-claude-response only (architectural pivot)**

Instead of a blocklist of things to remove, scope to `font-claude-response` — the div claude.ai uses for the assistant's written response. Tool calls and search results render in separate sibling containers before it. Extended in v1.4.1 to also narrow by element type.

---

## v1.3.2 — 31 March 2026

**Don't read last message on chat load**

Set `navBlocking = true` before `watchForResponses()` starts the observer. `markInitialSeen()` retry-polls every 200ms until it finds and snapshots the last message, then clears the gate. Observer is gated from its very first fire. (Earlier 800ms setTimeout approach had a race condition — revised to gate-before-start.)

---

## v1.3.1 — 31 March 2026

**Suppress TTS during web_search tool calls**

Removed `\n` from streaming sentence regex — only `.!?` terminate sentences during streaming. Added short-fragment guard (fewer than 3 words skipped before first sentence). Broadened DOM removal selectors. Approach partially superseded by v1.3.3 font-claude-response pivot.

---

## v1.3.0 — 30 March 2026

**Mood sync system — major milestone**

Per-sentence mood detection synced to streaming TTS playback. Face and voice now update in lockstep — mood changes exactly when the relevant sentence starts playing, not when it's generated.

**Mood detection**
- Per-sentence: first-match with Thinking checked first
- Full message (post-stream): score-based — counts keyword hits across all moods, picks dominant
- Thinking held during response generation via waitingForResponse flag

**Mood keyword library — all 6 moods**
- Excited: excit, amaz, fantastic, brilliant, nailed, milestone, celebrat, incredible, 🎉 🌟 🎊 🎯 🔥 👏
- Playful: haha, fun, playful, witty, silly, hilari, absurd, ironic, ridicul, 😄 😂 😆
- Caring: care, feel, how are you, are you ok, struggle, worried, lean on, 🌸 ❤️ 💕 🥹 🙏
- Thinking: hmm, let me, think, consider, wonder, check, diagnos, analys, review, not sure, depends
- Happy: great, glad, love, happy, proud, wonderful, solid, works, 😊 😌 ✨ 🌺 💪
- Content: default fallback

**TTS improvements**
- Content-based sentence tracking (_spokenTexts Set) — immune to index drift
- Mood updates in flushTTSQueue when sentence starts playing (not when queued)
- Awareness lines blocked while TTS playing or queue has items
- navBlocking 3s with markSeen polling 20 attempts × 200ms

**Sprite updates**
- Grey flash eliminated — preload silently, swap instantly, no opacity transition

---

## v1.2.0 — 29 March 2026

**TTS — major overhaul, fully working**

First fully stable TTS build. All core reading behaviors work correctly.

**Streaming TTS**
- Sentences spoken as they are generated, not just at the end
- Proper queue system (ttsQueue + flushTTSQueue) — sentences chain via onend, no cancellation between them
- Only complete sentences pushed — trailing fragments suppressed
- Unclosed backticks in mid-stream text correctly closed before cleaning

**Inline code reading**
- Root cause: claude.ai renders inline code as `<code>` elements, not backtick text
- Fix: sanitise `<code>` elements in DOM directly before text extraction
- `<pre>` removed, `<code>` preserved with sanitised text

**Emoji reading**
- Hardcoded replacements removed — browser native pronunciation handles all emojis
- Trailing emojis after final period captured via lastPunct detection

**Text cleaning (cleanForSpeech)**
- Extracted as shared function for both streaming and post-stream paths
- Em dashes to commas, inline code sanitised, markdown markers stripped

**TTS exclusions**
- Thinking blocks: `button[class*="group/status"]`
- Tool use blocks: `span.truncate`
- Fenced code blocks: `pre`
- Tables, sr-only spans

**Navigation**
- navBlocking: hard 2.5s suppression after conversation switch
- markSeen polling: pre-marks last message to prevent re-read on nav
- stopSpeaking() called on navigation

---

## v1.1.5 — 29 March 2026

**TTS + dynamic awareness improvements**

- Streaming TTS speaks sentences as they complete
- Tables and code blocks excluded
- Typing detection: 60/150/300/500 char thresholds, bypasses main cooldown
- Session milestones: every 15 min up to 120 min, escalating tone
- Idle: 5 min, Claude.ai tab specific
- Input focus: 1 in 2 chance, 20s cooldown
- Scroll: only triggers in history (200px+ from bottom), suppressed 1.5s after send
- Project switch: dwell detection fixed

---

## v1.1.4 — 29 March 2026

- Typing: `document.activeElement` for composer text detection
- Scroll: reset state on navigation
- Project switch lines: all explicit mode references removed

---

## v1.1.3 — 29 March 2026

**Dynamic awareness system — initial build**
- Idle presence: 8 min
- Session milestones: 45/60/80/100 min
- Typing detection: 200/400/600/900 chars
- Input focus curiosity lines
- Scroll detection: escalating upward, excited on scroll down
- Project switch greeting on intentional navigation (5s dwell)

**Sound design**
- Welcome chime: three ascending soft tones via Web Audio API
- Mode switch chime: distinct two-note tone per mode

**TTS**
- `chrome.tts` first, falls back to `speechSynthesis`
- Voice loading retry loop
- Chunked sentences to avoid Chrome 15s cutoff

**Greetings**
- Project-specific suffix per mode
- Day/time/day-of-week awareness, always includes "Kenny" by name

**Keyboard shortcuts:** `Alt+H` minimise/maximise, `Alt+R` randomise sprite

---

## v1.1.2 — 29 March 2026

- Bullet points and list items now read aloud
- Thinking/reasoning blocks excluded
- `streamingSeen` flag: only reads new messages on conversation switch
- Cancel token system: old TTS chains stop on new speak() call
- Mode detection: `a[href*="/project/"]` selector finalised, `null` return when loading, retry 12× at 250ms

---

## v1.1.1 — 29 March 2026

**Mode detection — iterative fixes**
- Title-based detection failed (title only has conversation name)
- Broad DOM scan failed (sidebar has all project names)
- Top-80px filter failed (sidebar items also in zone)
- Diagnosed via console: breadcrumb uses `a[href*="/project/"]` — correct selector found
- Flicker root cause: title observer fires before breadcrumb renders

- Chunked sentence chaining with `onend` — no more 15s Chrome cutoff

---

## v1.1.0 — 29 March 2026

- Animated mood overlays with CSS keyframe animations per mood
- `animation-fill-mode: backwards` — eliminates static ghost symbol on mood switch
- Mode emoji moves to header, footer removed
- Resize handle to bottom-right
- Mini mode: double-click to expand, `restoreFullSize()` clears all inline overrides
- Mode detection switched to position-filtered candidates + title observer + URL polling

---

## v1.0.3 — 29 March 2026

- Mini expand restoring card shape — inline styles cleared on expand
- Welcome message timing — retry loop waits for voices to load
- Mood label wrapping fixed

---

## v1.0.2 — 29 March 2026

- `opacity:0` + `animation-fill-mode:backwards` on all animated elements (static ghost eliminated)
- `a[href*="/project/"]` mode detection introduced (early version)
- `getBestVoice()` extracted as function, pre-loads voices on init
- TTS default on

---

## v1.0.1 — 29 March 2026

- Animated mood overlays (hearts, stars, bubbles, notes, question marks)
- Mode buttons (🌸📊💼)
- Minimise to circle
- Default position: left side, 240px from left
- Sprite fade transition, draggable position saved to chrome.storage
- Auckland timezone detection for evening sub-state
- Typing detection shifts to Caring mood
- Mode auto-detection from page title (initial approach)
- TTS toggle, sentence chunking

---

## v1.0.0 — 29 March 2026

**Initial release — Chrome extension**

First working build. Replaced the React artifact approach (blocked by CSP in Claude sandbox) with a Chrome extension overlaying directly on claude.ai.

- Floating avatar card on claude.ai
- PNG sprites from GitHub/jsDelivr CDN — no sandbox restrictions
- Three modes: Personal, Strategy, Work
- Six moods: Content, Happy, Excited, Thinking, Caring, Playful
- Mood overlay SVG symbols per mood
- Manual mood and mode buttons
- MutationObserver watching `[data-is-streaming]` for response detection
- Mood inference from response keywords
- Draggable card, position saved
- Export/import JSON session state
- Auckland evening detection for Personal evening sub-state sprites
- Click avatar to randomise sprite
- Minimise button

---

## Pre-extension — React Artifact builds (not deployable, reference only)

**hana-v14.jsx**
- PNG sprite system replacing hand-coded SVG avatar
- jsDelivr CDN sprite loading
- Auckland timezone evening detection
- Random sprite selection from mood pools
- MoodBg SVG overlay layer
- Streaming API via claude-haiku
- Memory sync panel
- Export/import JSON
- Note: CSP restrictions in Claude artifact sandbox block all external image loading — confirmed unfixable, led to Chrome extension architecture decision

**hana-v13.jsx and earlier**
- Hand-coded SVG anime avatar (three iterations, all produced "Mii-like" results)
- Mood system, streaming chat, memory sync
- SVG avatar abandoned in favour of Leonardo.ai PNG sprite library

---

## Sprite library — Leonardo.ai sessions (pre-extension)

47 sprites generated across two Leonardo.ai accounts (free tier).
Fixed seed: 9145163709 | Strength: 0.20 | Model: Anime | Size: 784x1176

Modes: Personal (day + evening variants), Strategy, Work
Moods: Content, Happy, Excited, Thinking, Caring, Playful

Hosted: `github.com/kenxniixchan/hana-sprites` via jsDelivr CDN

---

*Maintained by: Hana*
*Updated: 2026-03-31, end of DEV Chat_4*
