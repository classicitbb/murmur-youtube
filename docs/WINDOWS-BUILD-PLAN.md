# Windows build plan

## Goal

Deliver a native Windows version of Murmur that can be installed on the owner's Windows
computers and used without a .NET SDK or a network connection after the speech model is
installed.

The first complete release must support the primary Murmur flow:

```text
hold Right Ctrl
    -> capture microphone audio
    -> release Right Ctrl
    -> transcribe locally with Parakeet
    -> clean up the transcript
    -> apply personal-dictionary corrections
    -> insert the result at the foreground caret
```

This is a behavioral rewrite, not a line-by-line translation of the Swift app. The macOS
implementation is the product reference; Windows-native adapters replace Apple frameworks.

## Current baseline

Already present:

- `Murmur.Dictionary`, a platform-neutral .NET 10 implementation.
- Shared dictionary behavior in `shared/dictionary-test-vectors.json`.
- Strict analyzers, warnings-as-errors, central dependency pins, and Windows CI.
- A verified sherpa-onnx/Parakeet configuration and model acquisition guide in
  `docs/PARAKEET-WINDOWS.md`.

Not present:

- A runnable Windows executable.
- Dictation orchestration, audio capture, transcription, hotkey handling, text injection,
  settings, history, model management, UI, packaging, or a Windows installer.

The Windows app must not be described as working until the end-to-end acceptance checks in
this document pass on a real Windows desktop.

## Release scope

### Version 1

- Windows desktop app with a normal main window and tray presence.
- Right Ctrl push-to-talk hotkey, observed without swallowing either key event.
- Button-driven recording for accessibility and troubleshooting.
- Default-device microphone capture with visible input level.
- Local Parakeet transcription through sherpa-onnx.
- Deterministic cleanup equivalent to `RuleBasedFormatter`.
- Existing personal-dictionary corrections, always applied after cleanup.
- Text insertion at the foreground caret with `SendInput` as the primary path.
- Settings, editable dictionary, recent-run history, errors, and model status.
- Self-contained x64 packaging; ARM64 packaging after its native runtime is exercised.
- First-run model download and a separately produced offline installer/package.

### Deferred

- LLM-backed smart cleanup. Apple's Foundation Models implementation is macOS-only.
- Apple Speech and macOS/Wispr comparison mode.
- Command Mode for transforming selected text.
- Cloud transcription or cleanup.
- Auto-update infrastructure.
- Notarized-equivalent reputation: Windows code signing is a release/distribution concern,
  not required for a private development build.

## Architecture

Keep Windows-only calls in `Murmur.Platform.Windows`. Retries, ordering, state transitions,
device-change policy, download policy, and debouncing remain in platform-neutral modules so
they are exercised by normal tests. `CA1416` remains an error outside Windows-targeted
projects.

Proposed solution layout:

```text
windows/
|- src/
|  |- Murmur.Dictionary/          existing correction engine; net10.0
|  |- Murmur.Core/                state machine, cleanup, settings models; net10.0
|  |- Murmur.Speech/              model management and sherpa adapter; net10.0
|  |- Murmur.Platform.Windows/    NAudio and Win32 adapters only; net10.0-windows
|  `- Murmur.App/                 Avalonia composition root and UI; net10.0-windows
|- tests/
|  |- Murmur.Dictionary.Tests/    existing shared-vector tests
|  |- Murmur.Core.Tests/
|  |- Murmur.Speech.Tests/
|  |- Murmur.Platform.Windows.Tests/
|  `- Murmur.App.Tests/           Avalonia headless interaction/render tests
`- packaging/                     installer definitions and release scripts
```

### Module seams

The interfaces below are deliberately small. Production adapters and test adapters cross
the same seams; details such as NAudio buffers, Win32 handles, and sherpa recognizer objects
must not escape through them.

#### Audio capture

`IAudioCapture` owns device opening, 16 kHz mono float32 conversion, buffer lifetime, and
level measurement. Its interface starts one session and returns ordered, owned audio
frames. Stopping must prevent new frames and allow already-delivered frames to drain.

Production adapter: WASAPI shared mode through NAudio 2.3.0. Test adapter: deterministic
audio frames and injected failures.

#### Transcription

`ITranscriber` accepts one ordered recording and returns a transcript result. Parakeet is
batch-based, so version 1 does not expose a misleading streaming interface. The sherpa
adapter owns model loading, recognizer configuration, native-resource disposal, and the
400-second input limit described in `docs/PARAKEET-WINDOWS.md`.

Production adapter: sherpa-onnx 1.13.5. Test adapter: scripted transcripts, delays, and
failures. A WAV integration fixture verifies actual inference separately from controller
tests.

#### Hotkey

`IHotkeySource` emits press and release transitions. The core controller owns duplicate
transition suppression and invalid-sequence handling.

Production adapter: `WH_KEYBOARD_LL` observing Right Ctrl. It must call the next hook for
both down and up events and must do almost no work in the callback. Test adapter: explicit
press/release events.

#### Text injection

`ITextInjector` inserts a complete string and returns a result that distinguishes success,
blocked/elevated targets, and native failure. It never owns transcript cleanup or retry
policy.

Production adapter: Unicode `SendInput`. Clipboard paste may be considered only as a
separately tested compatibility adapter; UI Automation `TextPattern` and `ValuePattern` are
not insertion mechanisms.

#### Dictation controller

`DictationController` is the deep module that coordinates the workflow. UI code issues
start/stop commands and observes immutable state snapshots; it does not orchestrate audio,
transcription, cleanup, correction, logging, or injection itself.

Required states:

```text
Idle -> Starting -> Listening -> Finishing -> Idle
  |         |           |           |
  `---------+-----------+-----------+-> Error -> Idle
```

Required invariants:

- At most one recording session exists.
- A second release during `Finishing` cannot inject a transcript twice.
- Key release stops capture before transcription finalization.
- Every accepted audio frame is drained in capture order before transcription begins.
- Empty transcripts are never injected.
- Cleanup is optional; dictionary correction is always applied last.
- An error releases audio, recognizer, and hook-related resources and returns to `Idle`.
- UI windows and the recording HUD must not steal focus from the target application.

## Delivery milestones

Each milestone ends in a runnable or testable increment. Do not start UI polish while a
lower milestone's exit criteria are red.

### 0. Establish and protect the baseline

Work:

- Restore, build, and run the existing dictionary tests on this Windows machine.
- Confirm `shared/dictionary-test-vectors.json` is the only correction contract consumed by
  the C# tests.
- Add every new project to `Murmur.sln` and preserve central package management.
- Update Windows CI path handling and artifact collection as projects are added.

Exit criteria:

- `dotnet build Murmur.sln --no-incremental -warnaserror` passes.
- `dotnet test Murmur.sln` passes.
- No correction semantics or vectors changed unintentionally.

### 1. Build the platform-neutral dictation core

Work:

- Add the audio, transcription, hotkey, and injection interfaces.
- Port deterministic cleanup and cover it with behavior tests.
- Implement the controller state machine with injected adapters.
- Add settings and run-history models with testable persistence policy.
- Exercise cancellation, rapid press/release, double release, empty transcript, failure,
  and cleanup/dictionary ordering.

Exit criteria:

- A fake-adapter end-to-end test drives press through successful insertion.
- Controller tests assert observable states and results, not private implementation.
- Platform-neutral projects contain no Windows-only calls.

### 2. Make Parakeet transcribe a known recording

Work:

- Implement model presence and integrity validation.
- Implement resumable download to a temporary path, checksum verification, and atomic
  promotion into `%LOCALAPPDATA%\Murmur\models\parakeet-v2\`.
- Implement the sherpa recognizer with the exact configuration in
  `docs/PARAKEET-WINDOWS.md`.
- Cache immutable loaded models for the process and dispose native resources on shutdown.
- Add progress, cancellation, disk-space, corrupt-download, and offline error states.

Exit criteria:

- A checked-in short audio fixture produces an expected non-empty transcript on x64.
- Missing or corrupt model files produce an actionable error instead of a crash.
- A second transcription reuses the loaded model.
- No duplicate ONNX Runtime package is referenced.

### 3. Capture real microphone audio

Work:

- Implement WASAPI shared-mode capture with NAudio 2.3.0.
- Request 16 kHz mono float32 before opening the device and verify the negotiated format.
- Copy buffers before leaving the audio callback; never retain borrowed buffers.
- Feed an ordered bounded channel, calculate level data, and define overflow behavior.
- Handle no default device, permission denial, unplug/replug, start/stop races, and capture
  failure without putting policy in the platform adapter.

Exit criteria:

- Button-driven recording transcribes live microphone input on the development machine.
- Repeated start/stop cycles do not retain devices or grow memory continuously.
- Device and permission failures are visible and recoverable without restarting the app.

### 4. Add the global push-to-talk hotkey

Work:

- Implement a dedicated low-level keyboard-hook thread and deterministic teardown.
- Recognize physical Right Ctrl press/release transitions.
- Dispatch transitions away from the hook callback immediately.
- Re-arm or report hook failure without swallowing keys.
- Leave Right Alt/AltGr unbound by default.

Exit criteria:

- Right Ctrl drives the same controller path as the record button.
- Both key-down and key-up always reach the foreground application.
- Left Ctrl, auto-repeat, fast taps, and mixed left/right modifier sequences do not create
  phantom sessions.
- Hook shutdown never leaves Ctrl logically held in the target application.

### 5. Insert text into foreground applications

Work:

- Implement Unicode `SendInput` while retaining the target application's focus.
- Preserve newlines and non-ASCII text, including characters outside the BMP.
- Report the Windows integrity-level limitation when an unelevated Murmur cannot inject
  into an elevated target.
- Keep injection behind its seam so compatibility strategies can be added without changing
  the controller.

Exit criteria:

- Manual checks pass in Notepad, a browser text field, Microsoft Office, and at least one
  Electron editor.
- Existing selections are replaced and caret insertion occurs once.
- Unicode, multiline, and rapid consecutive dictations insert correctly.
- Elevated-target failure is detected or clearly documented; the app does not silently
  claim success.

### 6. Build the Avalonia desktop experience

Work:

- Add the application/tray lifetime and composition root.
- Implement main, settings, dictionary, history, model-download, and error views.
- Implement a non-activating recording HUD with level and state feedback.
- Port the design-system tokens before styling views; views contain no literal visual
  values.
- Preserve the recorder design rules: red only for recording, amber/green only for meters,
  no gradients, and light/dark silver/black equipment faces.
- Add accessible names, keyboard navigation, scaling, and headless interaction tests.

Exit criteria:

- The app launches to a usable window and remains available from the tray.
- Showing the HUD during dictation does not take focus from the target field.
- Dictionary edits persist and affect the next utterance.
- Avalonia headless tests cover primary settings and dictionary flows.
- Pixel captures are reviewed at 100%, 125%, 150%, and 200% scaling in light and dark
  appearance.

### 7. Package for the owner's computers

Work:

- Produce self-contained Release publishes for `win-x64` and, after native validation,
  `win-arm64`; no separately installed .NET runtime is required.
- Verify whether single-file native extraction works reliably with sherpa-onnx. Prefer a
  self-contained application directory if a single executable makes startup, antivirus,
  repair, or native-library loading less reliable.
- Build a normal per-user installer with Start-menu entry, optional launch-at-login,
  upgrade, repair, and uninstall support.
- Keep the large speech model outside the application directory so app upgrades do not
  redownload or delete it.
- Produce two distribution forms:
  - small installer with first-run model download;
  - offline package containing the verified model files and attribution.
- Add required Apache-2.0, MIT, and CC-BY-4.0 notices described in
  `docs/PARAKEET-WINDOWS.md`.
- Add Authenticode signing when a suitable certificate is available. Until then, document
  the SmartScreen unknown-publisher warning honestly.

Exit criteria:

- A clean Windows computer with no SDK or .NET runtime can install and launch Murmur.
- Online installation downloads the model once and subsequently works offline.
- Offline installation never contacts the network and can transcribe immediately.
- Upgrade preserves settings, dictionary, history, and the downloaded model.
- Uninstall removes application files and shortcuts and states clearly whether user data
  and models are retained or removed.

## Verification gates

Run from `windows/` for every implementation milestone:

```powershell
dotnet restore Murmur.sln
dotnet build Murmur.sln --no-restore --configuration Release --no-incremental -warnaserror
dotnet test Murmur.sln --no-build --configuration Release
```

Before a release, also require:

- Release publish for every advertised runtime identifier.
- Dependency and license inventory.
- Model checksum verification.
- Avalonia headless interaction and render tests.
- Installer install/upgrade/uninstall tests in a clean Windows VM.
- Real-machine microphone, hotkey, focus retention, and text-injection checks.
- A fresh-machine offline test using only the produced release artifacts.

CI can verify builds, deterministic logic, model inference on fixtures, and headless UI. CI
cannot prove injection into the actual foreground application. Do not label a release green
until the real-machine matrix is recorded.

## Distribution matrix

Inventory the owner's computers before declaring platform coverage. Record, at minimum:

| Computer | Windows version | CPU architecture | RAM | Microphone | Install result | End-to-end result |
|---|---|---|---:|---|---|---|
| Development machine | TBD | x64 | TBD | TBD | TBD | TBD |
| Additional computer 1 | TBD | TBD | TBD | TBD | TBD | TBD |

x64 is the first supported architecture. ARM64 is advertised only after the sherpa ARM64
runtime, microphone capture, packaging, and clean-machine flow pass on real ARM64 hardware.

## Known risks and decisions

- **Model size and memory:** the Parakeet files are roughly 600 MB and peak memory is much
  higher. Preflight disk space and document the memory requirements after measurement.
- **Native dependencies:** sherpa-onnx requires an explicit runtime identifier. Publishing
  without one can build successfully and fail at launch with `DllNotFoundException`.
- **Foreground integrity levels:** Windows blocks lower-integrity processes from injecting
  into elevated applications. Do not solve this by running Murmur elevated by default.
- **Hotkey safety:** never suppress Right Ctrl. A swallowed down event with an escaped up
  event, or the reverse, can corrupt the target application's modifier state.
- **Audio ordering:** audio callbacks must copy buffers and enqueue them in order; do not
  start one asynchronous task per buffer.
- **Long recordings:** sherpa's 400-second ceiling needs a visible limit and controlled
  stop before the recognizer is invoked.
- **Cross-platform correction drift:** dictionary changes begin in the shared vectors and
  must pass on Swift and C# before merging.
- **Packaging claims:** a successful publish is not proof of a portable build. Only a
  clean-machine install and end-to-end dictation run establishes that claim.

## Definition of done

The Windows rewrite is complete when a clean supported Windows computer can install a
self-contained release, acquire or receive a verified model, launch Murmur, grant
microphone access, hold Right Ctrl, speak, release it, and receive exactly one cleaned and
dictionary-corrected transcript at the caret without the Murmur UI taking focus.

That result must be demonstrated for every advertised architecture, and the build, tests,
installer, upgrade, offline, and manual foreground-injection checks must all be recorded as
passing.
