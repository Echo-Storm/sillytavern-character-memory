# CharMemory fork — project notes

Living notes for this fork effort. Lives on the orphan `tools` branch so it never
shows up in diffs against upstream or in any PR we send them. Update this as things
change — it's meant to let a session pick up cold.

## Purpose

Standard contributor flow: fork `bal-spec/sillytavern-character-memory`, find/fix bugs,
PR them back upstream. **Not** trying to take over or fork-and-diverge the project.

## Remotes / branches (in the local clone at `E:/sillytavern-character-memory`)

- `origin` → `https://github.com/Echo-Storm/sillytavern-character-memory.git` (our fork)
- `upstream` → `https://github.com/bal-spec/sillytavern-character-memory.git` (their repo)

We work off **`beta`**, not `master`. Reasoning: `beta` is 70 commits ahead of `master`
with 0 behind (pure superset), `master` hasn't been pushed to since 2026-04-28 (~3
months stale as of this writing), and every bug we've investigated that's already fixed
is fixed in `beta`, not `master`. Historically the two merged PRs against this repo
(#1, #12) targeted `master`, but given how stale master is, `beta` is the right base
until we hear otherwise from the maintainer.

### Branch map

| Branch | Contents | Status |
|---|---|---|
| `feature/issue-20-manual-extraction-pointer` | #20 — per-message "set as last-extracted point" button | committed, not pushed/PR'd |
| `fix/block-selection-picker-dom-race` | Consolidation's "Change selection…" picker was completely non-functional (DOM-read-after-popup-teardown, same class as issue #14) | committed, not pushed/PR'd |
| `fix/data-bank-write-order-data-loss` | `writeMemoriesForCharacter` deleted old file before confirming new upload succeeded | committed, not pushed/PR'd |
| `fix/manual-extraction-double-trigger-race` | `inApiCall` lock wasn't set until after the confirm popup await | committed, not pushed/PR'd |
| `fix/deletion-pointer-clamp` | No `MESSAGE_DELETED` handling anywhere; clamp `lastExtractedIndex` in `ensureMetadata()` | committed, not pushed/PR'd |
| `fix/reject-non-memory-extraction-fallback` | Un-bulleted LLM output (refusals, etc.) was saved as a memory verbatim | committed, not pushed/PR'd |
| `feature/inherit-pointer-on-checkpoint` | Auto-inherit `lastExtractedIndex` from parent chat on ST checkpoint/branch | committed, **UNVERIFIED LIVE** — built from reading ST core source, not yet checkpoint-tested |
| `fix/thread-abort-signal-to-fetch` | 2 commits: thread `abortSignal` into extraction's `fetch()` calls; throw on unexpected 200-but-malformed provider response shapes instead of returning `''` | committed, not pushed/PR'd |
| `fix/chunked-consolidation-retry-failed-chunks` | `runConsolidationLLM` gets `{ rethrow: true }` for the chunked path so the orchestrator's existing retry logic actually engages; added a failure toast when chunked consolidation exhausts retries (previously silent) | committed, not pushed/PR'd |
| `fix/getCharacterName-characterId-mismatch` | Fallback now reads `characters[context.characterId]` instead of the module-level `this_chid` global | committed, not pushed/PR'd |
| `fix/conversion-flow-edge-cases` | 4 fixes: `"* "` bullets recognized in the LLM-fallback parse; re-run warnings no longer gated behind block count; destination-merge confirms before silently dropping unparseable existing content; `getFilteredNanoGptModels` no longer assumes `capabilities` exists | committed, not pushed/PR'd |
| `testing/all-fixes` | All 11 branches above merged together (clean merges throughout, zero conflicts) | **pushed to origin**, checked out live in the user's SillyTavern install (`D:\Applications\SillyTavern\...\sillytavern-character-memory`) — needs a hard-refresh in browser to pick up this latest batch |
| `tools` | This orphan branch — notes/scratch only, no code history shared with the rest | pushed to origin |

All individual `fix/*` and `feature/*` branches are based on `beta` and kept separate
so each can become its own focused upstream PR. `testing/all-fixes` is the merge-them-
all-together branch purely for local live testing convenience — don't PR that one as-is.

Every fix has: syntax-checked (`node --check index.js`), full unit suite passing
(178/178, `npm test`), and a descriptive commit message. None have been pushed to
`origin` individually or opened as PRs yet — holding for the user's live testing pass
before anything goes to GitHub or upstream.

## Live SillyTavern test setup

- Install: `D:\Applications\SillyTavern`, served at `http://127.0.0.1:8000/`
- Extension path: `D:\Applications\SillyTavern\public\scripts\extensions\third-party\sillytavern-character-memory`
- That install was a plain clone of `upstream/master` (stale, pre-beta-fixes) before this
  session. Added remote `echostorm-fork` → our fork there, fetched, and checked out
  `testing/all-fixes` as a new local branch tracking `echostorm-fork/testing/all-fixes`.
- To pick up further changes: `git pull` (or re-fetch + reset) in that directory, then
  hard-refresh the SillyTavern page in browser (Ctrl+Shift+R) — no server restart needed,
  it's loaded as a plain ES module by the browser.
- Reminder: this jumped the live install from `master` straight to `beta`, so the user
  is now also seeing everything else already in beta (i18n, hide-extracted-messages,
  chunked consolidation, etc.) — not just our fixes. Don't mistake beta's existing
  features for something we added.

## Key facts / decisions from the audit

- **Issue #21** ("Failed to insert vector items for collection") is **not a CharMemory
  bug** — traced every vector-related code path; this extension only ever calls
  `/api/vector/list` to check status, never `/api/vector/insert`. That error and the
  "Vectorize all" button both belong to SillyTavern core's own Data Bank UI. Nothing to
  fix here on our end.
- **Issue #19** ("try CharMemory online — send them the link") is spam — third-party
  promo for "socialistic.ai" with UTM-tagged links, dressed up as a favor to the
  maintainer. Never followed the links. Not actionable.
- **Issue #16** is mostly resolved — maintainer (bal-spec) shipped 3 of 5 requested
  features directly in 2.2.0/2.3.0. Remaining: draggable memories (deliberately deferred,
  maintainer wants to design the UX himself — don't build this unsolicited) and manual
  vectorize button (maintainer explicitly declined it — don't build this either).
- There's a spam/self-promo comment from a "kinthaiofficial" account on issue #16 pitching
  a third-party product ("KinthAI") with claims about automatic memory-branching-on-fork.
  A later commenter (AceAI008) appears to have misattributed that third-party claim to
  CharMemory itself, which is where the "does forking copy memory state" confusion
  originated. bal-spec never claimed that behavior.
- **Issue #18** (destructive pointer reset in group chats) and its root cause **#17** are
  both fully fixed in beta already (guarded reset, yellow health check, auto-retry) —
  don't re-implement.
- The maintainer's repo has been dormant since 2026-04-28 despite issues still landing
  through July. Every `docs/plans/*.md` file opens with `> **For Claude:** REQUIRED
  SUB-SKILL: Use superpowers:executing-plans...` — bal-spec was running an entirely
  Claude-Code-driven workflow. The abrupt stop right after a 70-commit burst (all same
  day) is consistent with losing Claude access rather than a gradual wind-down. Unconfirmed
  theory, but it's why the user wants a low-friction "here are some bugfix PRs" approach
  rather than expecting fast maintainer engagement.
- SillyTavern does **not** copy `chat_metadata` into a new checkpoint/branch chat —
  confirmed by reading ST core's `bookmarks.js` (`createNewBookmark`): the new chat's
  metadata is built fresh as `{ main_chat: <parent chat id> }` only. This is a platform
  limitation affecting every extension, not a CharMemory-specific bug. See
  `feature/inherit-pointer-on-checkpoint`.

## Full audit findings

**Audit is complete** — every section of `index.js` (~9800 lines) plus `lib.js`,
`editor.js`, `consolidation.js` has been read (directly or via background agent). See
`tools/AUDIT-NOTES.md` for the complete ranked findings list. All 5 originally-found
high/medium severity bugs are fixed (branches above). Remaining open, not yet implemented:

- Abort signal never reaches actual `fetch()` calls in the LLM provider layer — "Stop
  extraction" can't cancel an in-flight single call.
- Some malformed/unexpected-shape provider responses fail silently (`return ''`) instead
  of surfacing an error.
- Chunked consolidation drops failed chunks silently instead of retrying (maintainer-
  acknowledged limitation, documented in a code comment, not hidden).
- `previewConversion`'s destination-merge could silently drop existing content that
  `parseMemories` can't round-trip, before an atomic overwrite — needs live verification
  with a hand-edited file, not clearly a bug without that.
- `getCharacterName()` reads `characters[this_chid]` instead of
  `characters[context.characterId]` after its own undefined-check — possible mismatch
  during rapid character switches, needs live verification.
- Smaller UI-polish-level items: `previewConversion` re-run silently no-ops on 0 blocks
  with no toast, `convertWithLLM` only salvages `"- "` bullets (not `"* "`),
  `getFilteredNanoGptModels` assumes every model has a `capabilities` array (could throw
  on some live API responses).

None of these remaining items are high-severity; they're candidates for a next pass,
not blockers.

## Working conventions for this fork effort

- One fix = one branch off `beta`, focused diff, own commit message explaining why (not
  just what).
- Always `node --check index.js` + `npm test` (178 tests) before committing.
- Don't touch `CHANGELOG.md` or bump `manifest.json`'s version — that's the maintainer's
  call at release time, not a contributor's.
- Don't scope-creep into deferred/declined features from issue threads (draggable
  memories, manual vectorize button) without checking with the user first.
- Nothing gets pushed to `origin` (the fork) or opened as a PR without explicit go-ahead
  — these are public/visible actions.
