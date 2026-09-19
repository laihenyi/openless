# AGENTS.md

This file provides guidance to coding agents (Claude Code, Codex, Grok) when working with code in this repository.

## Project

OpenLess is a menu-bar/tray voice-input layer. Hold or toggle a global hotkey, speak, and the dictated text is polished and inserted at the current cursor in any app. A second global combo (`Cmd+Shift+;` / `Ctrl+Shift+;`) drives the **划词 QA** flow — capture the foreground selection, dictate a question, get an LLM answer in an overlay. Product overview lives in `openless-all/README.md`; QA-specific design notes are in `docs/qa-reasoning-roadmap.md`. Read those before changing product behavior.

The active codebase lives at `openless-all/app/` and is **Tauri 2 + Rust backend + React/TS frontend**, targeting macOS 12+ and Windows. The legacy Swift implementation (Sources/, Tests/, Package.swift, appcast.xml, Sparkle pipeline) was removed in commit `34d2823`; do not resurrect it.

UI must match `openless-all/design_handoff_openless/*.jsx` pixel-for-pixel; the JSX is reference-only, never imported.

## Build, Run, Test

### Tauri (current — start here)

```bash
cd "openless-all/app"
npm ci

# Dev: vite at :1420 + tauri shell
npm run tauri dev

# Build .app (+ DMG) — use this script, not `tauri build` directly,
# because it threads Apple signing env vars and validates Info.plist.
./scripts/build-mac.sh           # build, sign, install to /Applications, reset TCC
INSTALL=0 ./scripts/build-mac.sh # build only

# Frontend-only TS check
npm run build   # = tsc && vite build

# Rust type-check without full compile
cargo check --manifest-path src-tauri/Cargo.toml
```

### Windows (cross-check only — no macOS runner in CI)

```powershell
# Preflight: verify toolchain
.\scripts\windows-preflight.ps1

# Build (requires Windows host or cross-compile target)
.\scripts\windows-build-gnu.ps1
```

Generated artifacts:
- `openless-all/app/src-tauri/target/release/bundle/macos/OpenLess.app`
- `openless-all/app/src-tauri/target/release/bundle/dmg/OpenLess_<version>_aarch64.dmg`

Logs: `~/Library/Logs/OpenLess/openless.log` (macOS) / `%LOCALAPPDATA%\OpenLess\Logs\openless.log` (Windows).

There is no test runner wired in for the frontend. `src/lib/providerSetup.test.ts` and `src/lib/capsuleLayout.test.ts` are hand-rolled assertion scripts — run with `npx tsx <file>` if you need them. Rust side has no `cargo test` targets yet; behavior is verified by running the app.

Smoke / regression scripts (Windows host required, but useful as authoritative checklists when triaging hotkey/insertion/permission issues on either OS):

```powershell
# All under openless-all/app/scripts/
.\scripts\windows-runtime-smoke.ps1                 # boots the built app and checks tray + hotkey registration
.\scripts\windows-real-asr-insertion-smoke.ps1      # end-to-end: hotkey → Volcengine → insertion
.\scripts\windows-microphone-privacy-smoke.ps1      # TCC equivalent: Settings → Microphone privacy
.\scripts\windows-hotkey-os-hook-smoke.ps1          # validates global keyboard hook survives focus changes
.\scripts\windows-smoke-suite.ps1                   # runs the suite above
```

```bash
npm run check:hotkey-injection                       # node smoke for the hotkey injection helper
```

## Architecture

`coordinator::Coordinator` is the **single owner of session state**. Hotkey edges drive a small phase enum (`Idle → Starting → Listening → Processing`); recorder, ASR, polish, insertion, and history are wired here and nowhere else. Library/module code never calls across modules — they each depend only on shared types.

```
Rust (openless-all/app/src-tauri/src)        Purpose
──────────────────────────────────────        ────────────────────────────────
types.rs                                      Pure value types: DictationSession, PolishMode, HotkeyBinding, QaHotkeyBinding, errors
hotkey.rs                                     Dictation hotkey monitor (modifier-only edges; macOS CGEventTap, Win/Linux rdev)
qa_hotkey.rs                                  QA combo-key monitor (Cmd/Ctrl+Shift+;) via `global-hotkey` crate
selection.rs                                  Foreground-selection capture: macOS AX → Cmd/Ctrl+C clipboard snapshot → none (Linux)
recorder.rs                                   Mic → 16 kHz mono Int16 PCM, RMS callback
asr/{mod,frame,volcengine,whisper}.rs         ASR providers: Volcengine streaming WebSocket + Whisper HTTP
polish.rs                                     OpenAI-compatible chat completions (Ark / DeepSeek / etc.) + QA reasoning prompts
insertion.rs                                  AX focused-element write → clipboard + Cmd+V → copy-only fallback
persistence.rs                                History/preferences/vocab JSON + Keychain credentials
coordinator.rs + commands.rs + lib.rs         State machine, IPC surface, tray icon, window plumbing
permissions.rs                                TCC checks (Accessibility / Microphone)

Frontend (openless-all/app/src)
src/components/Capsule.tsx                    Capsule view + state enum
src/components/FloatingShell.tsx              Always-on-top dictation/QA shell window
src/pages/{Overview,History,Vocab,Style,Settings,QaPanel,SelectionAsk,Translation}.tsx  Main-window pages
src/pages/_atoms.tsx                          UI primitives (PageHeader/Card/Pill/Btn), ported 1:1 from design_handoff_openless/pages.jsx — *not* Recoil; repo has no global state library
src/state/{useAppState.ts,HotkeySettingsContext.tsx}  React hook + context: app phase, hotkey capability/binding from backend
src/i18n/                                     react-i18next init + zh-CN / en resources
```

### Dictation pipeline

```
hotkey edge (1st)  →  beginSession:  Recorder.start → ASR.openSession → BufferingAudioConsumer.attach
hotkey edge (2nd)  →  endSession:    Recorder.stop → ASR.sendLastFrame → awaitFinal → Polish → Insert → History.save
.cancelled         →  ASR.cancel, Recorder.stop, capsule .cancelled
```

### QA pipeline (划词问答)

```
qa_hotkey edge (1st)  →  selection.capture (AX or Cmd/Ctrl+C snapshot) → Recorder.start → ASR.openSession
qa_hotkey edge (2nd)  →  Recorder.stop → ASR.awaitFinal → Polish (reasoning prompt with selection as context)
                          → render answer in QaPanel overlay (no insertion)
```

Invariants:
- **Polish/ASR fallbacks are silent.** Missing Ark creds → insert raw transcript. Missing Volcengine creds → mock pipeline copies a placeholder. The contract is *"the user's words don't get lost"* — don't add hard errors here.
- **`BufferingAudioConsumer`** queues PCM until the WebSocket is ready, then drains. Recorder always pushes to it; ASR is attached after `openSession` resolves.
- **Hotkey is toggle-only**, not press-and-hold. Both `hotkey.rs` and `qa_hotkey.rs` yield one edge per keydown; the coordinator interprets odd/even.
- **QA selection is truncated to 4000 chars** (head 2000 + tail 2000 + `[…truncated…]`) before being passed to the LLM — keep that policy in `selection.rs`, don't shift it into prompt assembly.
- **Dictation and QA share the recorder/ASR singletons** but are mutually exclusive sessions; coordinator rejects starting one while the other is active.

### Permissions, credentials, on-disk state

- **Bundle ID `com.openless.app`** is hard-coded in `openless-all/app/src-tauri/tauri.conf.json` and `CredentialsVault.serviceName`. Changing it breaks Keychain lookups *and* every existing TCC grant.
- **TCC**: Microphone + Accessibility + AppleEvents. `NSMicrophoneUsageDescription` / `NSAccessibilityUsageDescription` / `NSAppleEventsUsageDescription` live in `openless-all/app/src-tauri/Info.plist`. After a fresh build that resets TCC, the app must be **fully quit and relaunched** after granting Accessibility before the global hotkey tap installs.
- **Credentials** live in Keychain under accounts in `CredentialAccount` (`volcengine.app_key`, `volcengine.access_key`, `volcengine.resource_id`, `ark.api_key`, `ark.model_id`, `ark.endpoint`). The plaintext fallback at `~/.openless/credentials.json` is read on first launch so legacy users keep their creds without re-entering. Never hard-code keys.
- **Per-user data**:
  - macOS: `~/Library/Application Support/OpenLess/{history.json, preferences.json, dictionary.json}` — capped at 200 history entries. **Do not rename `dictionary.json` to `vocab.json`** (drops user data).
  - Windows: `%APPDATA%\OpenLess\`
  - Linux: `$XDG_DATA_HOME/OpenLess`

### Release pipeline

Push a `v*-tauri` tag → `.github/workflows/release-tauri.yml` builds macOS arm64 `.dmg` and Windows x64 `.msi`. macOS Developer ID signing + notarization runs only when `APPLE_CERTIFICATE` / `APPLE_CERTIFICATE_PASSWORD` / `APPLE_ID` / `APPLE_PASSWORD` / `APPLE_TEAM_ID` secrets are set; otherwise it falls back to ad-hoc signing with a CI warning.

When bumping versions, update **both** `version` fields: `openless-all/app/package.json` and `openless-all/app/src-tauri/tauri.conf.json` (and `Cargo.toml`).

## Repo conventions

- **Comments, log messages, user-facing strings, and most docs are in Simplified Chinese.** UI strings additionally route through `react-i18next` (`src/i18n/{zh-CN,zh-TW,en}.ts`) so we ship English and Traditional Chinese (Taiwan) alongside; `zh-CN.ts` is source of truth. `zh-TW.ts` is auto-generated from `zh-CN.ts` via `npm run gen:zh-tw` (using OpenCC s2twp dicts from `src-tauri/dicts/` + manual overrides from `zh-TW.overrides.json`). Commit both files after changing `zh-CN.ts`.
- **Language / locale** — three-layer conversion with double insurance:
  1. **i18n build-time**: `scripts/gen-zh-tw.mjs` reads the same OpenCC dicts as Rust (`STCharacters.txt` → `STPhrases.txt` → `TWPhrases.txt` → `TWVariants.txt`), builds a longest-match trie, converts all strings in `zh-CN.ts` to `zh-TW.ts`, then applies manual overrides from `zh-TW.overrides.json`.
  2. **ASR runtime**: `coordinator.rs` applies `text_locale::s2twp()` to ASR partial/final results when `prefs.language` resolves to `TraditionalTW`. This ensures the capsule preview and history are in traditional Chinese from the start.
  3. **Polish runtime**: `polish.rs::compose_system_prompt()` appends a Taiwan customary phrase instruction to the LLM system prompt when `output_locale == TraditionalTW`. After polish, `coordinator.rs` runs `text_locale::s2twp()` again on the LLM output — the LLM occasionally mixes simplified characters (especially near the end of long outputs), and this second pass catches them. This double insurance guarantees pure traditional output in history and insertion.
  - `preferences.language` field: `"auto"` (default, resolves via OS locale), `"zh-CN"`, `"zh-TW"`, `"en"`. The `resolve_language` Tauri command resolves `"auto"` by calling `text_locale::system_locale()` (macOS AppleLocale → LC_ALL/LANG env vars). Migration: existing users without the field get `"auto"` (not silently switched to traditional).
  - QA flow also uses this locale: `repolish` reads the resolved language and passes it through the same pipeline.
- **macOS hotkey monitor must use native `CGEventTap`, never `rdev`.** `rdev` synchronously calls `TSMGetInputSourceProperty` from non-main threads, which macOS 14+ aborts via `dispatch_assert_queue_fail` → SIGTRAP. macOS uses CGEventTap; `rdev` is only used on Linux/Windows.
- **Don't `NSApp.activate` on the dictation path** — it steals focus and breaks insertion. Only call `set_activation_policy(Regular)` + `activateIgnoringOtherApps` from `show_main_window` / mic-permission prompts, never from `start_dictation`.
- Rust modules wrap shared mutable state with `Arc<Mutex<...>>` (parking_lot). Keep that locking discipline when adding fields.
- Rust modules depend only on `types.rs`. New cross-module wiring goes in `coordinator.rs`, not in the leaf modules.

### Adding a new module

1. Add a `<name>.rs` (or directory) under `openless-all/app/src-tauri/src/`, importing only from `types`.
2. Register it in `lib.rs` (`mod <name>;`).
3. Wire it into `coordinator.rs` and expose any frontend-callable surface via `commands.rs` + `invoke_handler!`.
4. Add the matching TS wrapper in `openless-all/app/src/lib/ipc.ts` (with a mock branch for browser dev).
