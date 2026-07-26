# CharMemory (beta) — Baseline Audit

Local working notes only — not part of any PR, not committed upstream. Written 2026-07-26
against `beta` @ 2.3.0 (fork: Echo-Storm/sillytavern-character-memory).

**Status: full-file audit complete.** Every section of `index.js` (~9,800 lines) plus
`lib.js`, `editor.js`, and `consolidation.js` has been read — either directly or via a
background agent — and cross-checked for the same handful of recurring bug patterns
(popup-DOM-read-after-teardown, delete-before-confirmed-write, reentrancy locks set too
late, unvalidated LLM output persisted as data). Nothing below is guesswork extrapolated
from "similar code probably has similar bugs" — every finding traces to an actual read of
the specific lines cited.

## Scale

- `index.js`: ~9,800 lines — the entire extension (UI, event wiring, LLM calls, state).
- `lib.js`: ~670 lines — pure, side-effect-free helpers, fully unit-tested.
- `consolidation.js` (114), `editor.js` (111) — small, tested, extracted modules.
- `style.css`: ~2,350 lines.
- Total JS/CSS/HTML: ~13,100 lines. Whole repo incl. docs/locales: ~45,000 lines.
- Unit tests: 178 tests / 11 files, covering `lib.js`, `consolidation.js`, `editor.js` only.
  **`index.js` itself has ~0 direct unit coverage** — it's tightly coupled to jQuery/DOM/ST
  globals. This is where bugs hide; every finding below is from `index.js`.

## Architecture map (index.js section banners, in file order)

| Lines | Section | Notes |
|---|---|---|
| 1-95 | imports, module constants, global state | |
| 95-553 | Activity Log | small, low risk |
| 553-1732 | Structured Memory Helpers | serialize/parse, settings load, status display, `ensureMetadata`, provider dropdowns |
| 1732-1836 | Group Chat Helpers | `getGroupMembersDetailed`, `getMemoryTargets` |
| 1836-2003 | Per-Character Data Bank Operations | `readMemoriesForCharacter`, `writeMemoriesForCharacter` |
| 2003-2066 | Server API Helpers | ST chat-history endpoints |
| 2066-3344 | Provider API Helpers | multi-backend LLM client + `extractMemories` pipeline |
| 3344-3560 | Diagnostics | injection snapshot capture |
| 3560-3924 | Injection Health Score | advisory health checks, read-only |
| 3924-4746 | Settings Modal | |
| 4746-5252 | Prompts Modal | |
| 5252-6174 | Setup Wizard | |
| 6174-6897 | Troubleshooter Modal | health checks, Data Bank browser, diagnostic report |
| 6897-7043 | Memory Manager | block/bullet editor |
| 7043-7115 | Find & Replace | shared bar, reused across surfaces |
| 7115-7815 | Consolidation | presets, chunked map-reduce, block-selection override |
| 7815-8142 | Reformat Tool | |
| 8142-8176 | Slash Commands | |
| 8176-8405 | UI Setup | listener wiring |
| 8405-9282 | Per-Message Buttons & Indicators | pin / extract-here / set-last-extracted (ours) / view-injected |
| 9282-9501 | Batch Extraction | |
| 9501-end | Init | event hooks, drawer setup |

Key global state: `chat_metadata[MODULE_NAME]` (per-chat: `lastExtractedIndex`,
`messagesSinceExtraction`, `injectionData`) and `extension_settings[MODULE_NAME]`
(persistent config, `character_attachments`, `batchState`). `chat_metadata` does **not**
survive SillyTavern checkpoints/forks (confirmed against ST core `bookmarks.js`) —
new chat gets `{ main_chat: <parent id> }` only. No CharMemory code compensates for this;
`markChatAsFullyExtracted()` / our new per-message pointer-set button are the manual workaround.

## Findings, ranked by severity

### High — confirmed by direct code read

1. **Block-selection override picker is completely non-functional** (`showBlockSelectionModal`,
   [index.js:7403](index.js#L7403), used by `consolidateMemories` at
   [index.js:7539](index.js#L7539)). Reads `$('.charMemory_blockPickerCheck:checked')`
   *after* `await callGenericPopup(...)` resolves — but per the codebase's own established
   precedent (issue #14: "the selected radio button was read after callGenericPopup closed
   and destroyed the popup DOM, so the selector always returned undefined"), the popup DOM
   is already gone by then. Result: `showBlockSelectionModal` always returns an empty `Set`
   when confirmed. Caller sees `eligible.length === 0` and shows "All memories appear to be
   already consolidated — nothing new to do" — a silent, misleading no-op. This is the
   "Change selection…" feature shipped in 2.3.0's changelog for selective consolidation;
   it doesn't work at all, regardless of what the user checks/unchecks.
   **Fix shape**: track a live `Set` via the existing `change.blockPicker` handler (already
   fires on every checkbox change) instead of re-querying DOM after teardown — same pattern
   already used correctly by `showMemoryManager` (~6927), `consolidateMemories`'s own
   character picker (~7494), `reformatMemories` (~8032), and the pin-memory picker (~8614).

2. **Delete-then-upload data-loss window** (`writeMemoriesForCharacter`,
   [index.js:1941](index.js#L1941)). Overwriting a file deletes the old one first, then
   uploads new content; if the upload fails after the delete succeeds, the old memories are
   gone and the new ones never landed. No rollback, no backup. The outer extraction
   try/catch does surface an error toast (not silent), but the data is still lost.
   **Fix shape**: upload new content first, confirm success, then delete the old file.

3. **Duplicate-extraction race on manual confirm** (`extractMemories`, confirm popup at
   ~2914, `inApiCall = true` not set until ~2937). The `inApiCall` reentrancy guard only
   protects calls made *after* that assignment — while the confirm dialog is open, a second
   manual-extraction trigger can pass the guard too. Both get confirmed → concurrent
   extraction runs → duplicate memory writes, unpredictable final `lastExtractedIndex`.
   **Fix shape**: set `inApiCall = true` (or a separate lighter lock) before showing the
   confirm popup, not after.

### Medium — confirmed by code read, lower likelihood or impact

4. ~~No message-deletion handling anywhere~~ — **FIXED**, `fix/deletion-pointer-clamp`.
   No `MESSAGE_DELETED` listener existed anywhere; clamped `lastExtractedIndex` to
   `chat.length - 1` inside `ensureMetadata()`, which every read/write path already calls.

5. ~~LLM refusal/off-topic response can be saved as a memory verbatim~~ — **FIXED**,
   `fix/reject-non-memory-extraction-fallback`. Un-bulleted entries are now skipped and
   logged instead of saved raw.

6. **`abortSignal` never reaches the actual `fetch()` calls** (provider generator functions,
   ~2186-2451). Checked only between chunks/targets, so "Stop extraction" can't cancel a
   single slow/hung LLM call — it only takes effect after that call finally resolves.
   Not yet fixed.

7. **Silent shape-mismatch masking** (`generateOpenAICompatibleResponse` /
   `generateAnthropicResponse`, ~2242/2279/2344). A 200 response with an unexpected body
   shape (some providers do this) returns `''` rather than throwing — surfaces upstream as
   "No new memories found," masking a real API/provider problem. Not yet fixed.

8. **Chunked consolidation drops failed chunks silently** (~7581-7591, acknowledged in a
   code comment). `runConsolidationLLM` catches its own errors and returns `null`, so the
   orchestrator's retry-on-throw path never fires — a transient network blip during a large
   chunked consolidation just drops that chunk's memories rather than retrying. Documented
   limitation, not hidden, but still a data-quality gap. Not yet fixed.

9. **`previewConversion` destination-merge round-trip** (index.js ~1046-1053). When
   converting into an existing CharMemory file, the flow does
   `parseMemories(existingContent)` then overwrites with
   `serializeMemories([...existingBlocks, ...newBlocks])`. If the existing file has any
   content `parseMemories` can't fully round-trip (hand edits, slightly malformed
   `<memory>` tags), that content is silently dropped and then the file is atomically
   overwritten — permanent, no rollback since this is a confirmed-write, not the
   delete-then-upload race. Depends on `parseMemories`'s real-world tolerance; worth
   verifying live with a hand-edited file rather than assuming.

10. **`getCharacterName()` fallback mismatch** (index.js ~1740-1743). Bails on
    `context.characterId === undefined` but then reads name from the global
    `characters[this_chid]` rather than `characters[context.characterId]`. If these two
    ever diverge — e.g. a rapid character switch mid-async-operation, a hazard this same
    file calls out elsewhere (`savedCharId`/`savedChatId` checks in `extractMemories`) —
    the name baked into an LLM conversion prompt could belong to a different character
    than the Data Bank file actually being written. Worth verifying live.

### Low / worth verifying live, not confirmed

- Verbose-mode error logging (`logActivity` of `JSON.stringify(errorBody)` on provider
  errors) — worth checking whether any provider ever echoes request headers/key fragments
  in its error body, which would leak into the in-app activity log.
- `fetchProviderModels`/`fetchNanoGptModels` (~2083-2104): only the subscription sub-fetch
  is `.catch()`-guarded; a network-level rejection on the primary fetch may propagate
  uncaught — didn't trace the UI caller far enough to confirm it's swallowed.
- `previewConversion` re-run swallows failure explanation (~971-981): the warnings toast
  only fires when the re-run returns ≥1 block. A re-run that legitimately returns 0 blocks
  (e.g. heuristic parse of empty/freeform content) shows nothing — dialog just looks like
  it did nothing, no explanation.
- `convertWithLLM` bullet-salvage fallback (~706) only recognizes `"- "` bullets when no
  `<memory>` tags are present; a response using `"* "` bullets is discarded entirely with
  "No memories could be extracted," even though valid content existed.
- `getFilteredNanoGptModels` (~1254) assumes every model object has a `capabilities` array
  (`m.capabilities.includes('reasoning')`). If any live NanoGPT API response omits it, this
  throws and `populateProviderModels`'s catch rethrows, breaking the whole model dropdown
  whenever the "reasoning" filter is checked. Depends on NanoGPT's actual API shape — worth
  verifying live rather than assuming.

## Already fixed in beta (don't re-do)

Full list of previously-filed issues and their status is in the conversation history, not
repeated here. Summary: #9, #13, #14, #17/#18 all have confirmed fixes in beta's current
code. #16's asks (extraction lag, generation-aware deferral, manual pointer-set, mobile UX)
are shipped except draggable memories (deliberately deferred by maintainer) and manual
vectorize button (maintainer explicitly declined). #21 is not a CharMemory bug (traced to
SillyTavern core / local-transformers backend). #19 is spam, not a real issue.

## editor.js / consolidation.js

Both fully read, both clean. `editor.js`'s `createMemoryEditor` is pure state management
(clone-on-read, undo stack, editing-set tracking) with no DOM access and no bugs found.
`consolidation.js`'s `runChunkedConsolidation` is a pure map-reduce orchestrator with
per-chunk retry and cancellation checks — also clean. No findings, no action needed.

## Already done this session

All committed and unit-tested (178/178 passing), see `tools/PROJECT-NOTES.md` for the
full branch map and live-testing status:

- **#20** — manual extraction-pointer control, `feature/issue-20-manual-extraction-pointer`
- **Finding #1** (block-selection picker) — `fix/block-selection-picker-dom-race`
- **Finding #2** (delete-then-upload data loss) — `fix/data-bank-write-order-data-loss`
- **Finding #3** (double-extraction race) — `fix/manual-extraction-double-trigger-race`
- **Finding #4** (deletion-pointer clamp) — `fix/deletion-pointer-clamp`
- **Finding #5** (refusal-as-memory) — `fix/reject-non-memory-extraction-fallback`
- **New feature**, not from the original issue list — auto-inherit extraction pointer on
  SillyTavern checkpoint/branch — `feature/inherit-pointer-on-checkpoint`
  (**unverified live** as of this writing)

All merged together into `testing/all-fixes`, currently checked out live in the user's
SillyTavern install for testing. Nothing pushed as individual PR branches or opened
against upstream yet.

Findings #6-10 and the four "low / worth verifying live" items remain open, not yet fixed.
