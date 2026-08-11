# CLAUDE.md

This file gives Claude (and Claude Code) the context needed to work on this project without re-discovering it from scratch each session.

## Project overview

**Reminder Timer** is a single-file, client-side web app for running a countdown timer against a personal task list. A user adds tasks (name, duration, alert interval), starts one from its own card, and the app counts down, firing a periodic alert at the chosen interval (tone, escalating tone, or spoken voice) and a distinct completion sound/announcement when time runs out.

There is no backend. Everything — UI, logic, and persistence — lives in one HTML file.

## Tech stack

- Plain HTML + CSS + vanilla JavaScript. No frameworks, no build step, no package.json.
- No external libraries or CDN imports.
- Persistence via the `window.storage` browser-storage API (get/set/delete/list), **not** `localStorage`/`sessionStorage` — those are unavailable in the Claude Artifacts sandbox this app is developed in and must never be reintroduced. **Note:** `window.storage` is also not available in a normal deployed context (e.g. the Vercel deployment) — this needs a real persistence layer (e.g. `localStorage`) swapped in before/after it leaves Claude's environment, or storage calls will fail.
- Audio via the Web Audio API (`AudioContext`), generated in-code (oscillators) — no audio files.
- Spoken alerts via the `SpeechSynthesis` API — no audio files or TTS backend calls; uses whatever voices the browser/OS exposes.
- Background-safe ticking via a dedicated `Worker` (created from an inline `Blob`), because `setInterval` on the main thread gets throttled when the tab is backgrounded.
- Best-effort `navigator.wakeLock` to keep the screen awake while a timer runs.
- Browser `Notification` API as a backup alert channel alongside sound.

## File structure

```
reminder-timer.html   # the entire app: markup, styles, and script in one file
CLAUDE.md              # this file
```

There is nothing to install and nothing to build. To preview changes, just open `reminder-timer.html` directly in a browser (or reload it in whatever environment is serving it).

## How to run / test

- No dev server or build tooling — open the file in a browser.
- Manual test checklist after changes:
  1. Add a task (name + minutes/seconds + alert interval) via "+ Add task".
  2. Click the ▶ icon on a task card — it should start, and the pause/stop icon should immediately show as enabled (❙❙ Pause), not stay disabled.
  3. Let an interval alert fire — confirm the correct alert plays for the selected Settings → Alert sound mode (same tone / escalating tone / human voice).
  4. If "Human voice" is selected, confirm it speaks only the time (e.g. "15 minutes.") — never the task name.
  5. Try the voice accent dropdown (only visible when "Human voice" is selected) — confirm it lists real installed voices and previews on change.
  6. Let the timer reach zero — confirm the completion chime (or "Time is up." if in voice mode) plays and the card shows "Time is up".
  7. Pause, resume (via ▶), and stop (via the same icon showing ■ once paused) from the task's own icon row.
  8. Delete (×) a task while it's active — confirm the timer actually stops (worker stopped, wake lock released), not just removed from the list.
  9. Reload the page — tasks, sound-mode, and voice-accent preferences should still be there (loaded from `window.storage`).
  10. Check the "Enable notifications" button — it should only be visible when permission is still `"default"`; once granted/denied for that origin it correctly disappears.
- There is no automated test suite. If one is ever added, document how to run it here.

## Architecture notes

Everything runs in one `<script>` block at the bottom of the file. Key pieces:

- **Timer state** is a small set of module-level `let` variables (`totalSeconds`, `remaining`, `intervalSec`, `timerId`, `isPaused`, `endTime`, `activeTaskId`, `soundMode`, `selectedVoiceKey`, etc.). Only **one task can be active at a time** — `activeTaskId` tracks which.
- **`tick()`** recomputes `remaining` from `endTime - Date.now()` every second (driven by the worker's `postMessage`) rather than decrementing a counter, so background-tab throttling can't cause drift.
- **`startTimer(name, durationSeconds, intervalMinutes, taskId)`** is the single entry point for starting a countdown, called from a task's own ▶ icon. It sets `timerId = true` **before** calling `renderTasks()` — this ordering matters: rendering before setting `timerId` was a bug that left the pause/stop icon looking disabled on the very first click.
- **`togglePause()`** and **`stopTimer()`** are shared functions used by the per-task icon buttons.
- **The running clock/status/progress bar live inside the active task's own card.** `renderTasks()` conditionally injects a `.task-clock-box` (with `#statusMsg`, `#clock`, `#progressBar` — reused IDs, safe because only one task is ever active) below the task's meta line, sized to the **full width of the card** — not indented under a narrower column. Because of this, `startTimer()` calls `renderTasks()` before touching those elements, and `tick()`/`updateDisplay()` null-check them defensively in case a re-render hasn't happened yet.
- **On completion**, `activeTaskId` is deliberately *not* cleared — the task stays "active" showing "Time is up" until the user explicitly stops (■) or restarts (▶) it from its own card.
- **Sound**: `playAlertTone(progress, message)` routes to one of three places based on `soundMode`:
  - `"same"` → `beep()` — a calm, fixed-pitch "tick tick".
  - `"escalating"` → `beepEscalating(progress)` — pitch/waveform intensifies with `progress` (0→1), always exactly 2 ticks.
  - `"voice"` → `speak(message)` — spoken via `SpeechSynthesis`. Voice-mode messages are deliberately minimal: **just the time** (e.g. `"15 minutes."`, `"Time is up."`) — never the task name.
  - `beepDone()` is a separate, distinct 3-note descending chime reserved for task completion in tone modes; in voice mode, completion instead speaks `"Time is up."` (see `tick()`).
- **Voice selection** (`pickVoice()` / `populateVoiceSelect()`): the `#voiceSelect` dropdown (only shown when `soundMode === "voice"`) is populated dynamically from every **English** `SpeechSynthesis` voice actually installed on the device (`v.lang` starting with `"en"`), not a hardcoded list — quality/availability varies a lot by browser/OS (e.g. Chrome/Edge expose "Google …" voices requiring network access; Safari/Firefox only expose OS-native voices). `"Default (Auto)"` (empty `selectedVoiceKey`) falls back to preferring `en-US`, then any English voice, then whatever's first. Voice list loads asynchronously (`speechSynthesis.onvoiceschanged`), so the dropdown repopulates when it becomes available.
- **Persistence**: three `window.storage` keys, all non-shared (`shared: false`, i.e. private to the user):
  - `reminder-timer-tasks` — JSON array of task objects `{ id, name, durationMin, durationSec, interval }`.
  - `reminder-timer-sound-mode` — `"same"` | `"escalating"` | `"voice"`.
  - `reminder-timer-voice-name` — `""` (auto) or a `"name__lang"` composite key identifying a specific installed voice.
  - `loadTasks()` migrates any task saved under the old pre-split `duration` (minutes-only) field into `durationMin`/`durationSec` on load.
- **No drag-and-drop, no separate timer panel.** These existed in an earlier version and were intentionally removed — starting a task is only ever done via its own ▶ icon. Don't reintroduce a `#timerCard` drop target unless explicitly asked.

## Current UI layout

Single card ("Today's tasks"), top to bottom:

1. **Settings card** (nested, boxed section labeled "Settings"):
   - Alert sound radios: Same tone / Escalating tone / Human voice.
   - Accent `<select>` (`#voiceSelect`), only visible when "Human voice" is selected, listing installed English voices.
   - Notifications: "Enable notifications" button + status text — button is conditionally hidden once permission is already granted or denied for the origin.
2. **"+ Add task" toggle button** — sits directly above its form, doubling as "Cancel" (text/style swap) when the form is open, so it's always right next to the fields it controls.
3. **Add-task form** (`#addTaskForm`) — name, minutes, seconds, alert-interval, "Add task" submit.
4. **Task list** (`#taskList`) — each task card:
   - Row 1 (single line): task name, then 3 icon buttons — ▶ Play/Resume, a **merged pause/stop icon** (❙❙ while running → ■ once paused), and × Close/Delete.
   - Row 2: meta line — `"X min · alert every Y min"`, with `" · running"` appended while active.
   - Row 3 (only when active): the full-width `.task-clock-box` (status text, big clock, progress bar).

There is no separate right-hand "Reminder timer" panel — everything lives in this one card, max-width ~560px.

## Conventions to follow when editing

- Keep everything in the single `reminder-timer.html` file unless explicitly asked to split it up.
- Never introduce `localStorage`/`sessionStorage` — use the `window.storage` API and wrap calls in try/catch.
- Keep tone-based alerts implemented as Web Audio oscillator code, not audio file assets. Keep voice alerts as `SpeechSynthesis`, not TTS audio files or network calls.
- Voice-mode spoken text stays minimal (time only) — don't add the task name or other detail back into what gets spoken unless asked.
- Don't build accent/voice filtering logic that singles out or excludes any specific accent/nationality (e.g. excluding Indian English) — list what's available, or curate positively (a preferred/known-good list), never exclude by nationality.
- When adding new per-task UI, remember only one task is ever "active" — reused fixed IDs (`#clock`, `#statusMsg`, `#progressBar`) inside the active card are intentional, not a bug.
- Match the existing visual style: soft neutral palette (`#f5f5f2` background, white cards, `#222` as the primary dark accent), rounded corners, no external fonts/icons (icons are inline Unicode/HTML entities, e.g. `&#9654;` for ▶).

## Open items / known gotchas

- **`window.storage` won't work outside Claude's Artifacts sandbox.** The user has deployed this to Vercel; on that deployment, saved tasks/settings likely aren't actually persisting since `window.storage` doesn't exist there. This needs a real persistence layer (e.g. `localStorage`, or a small backend) before it'll work correctly in production.
- Voice quality/accent availability is entirely dependent on the browser and OS the person is using — there's no way to guarantee a specific voice (e.g. "Google US English") is available on every device; the UI is built to degrade gracefully (dynamic list + "Default (Auto)" fallback) rather than assume anything is present.
