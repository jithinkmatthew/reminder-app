# CLAUDE.md

This file gives Claude (and Claude Code) the context needed to work on this project without re-discovering it from scratch each session.

## Project overview

**MyCue** (formerly "Reminder Timer", internal keys/ids still use `reminder-timer-*`) is a single-file, client-side web app for running a countdown timer against a daily task schedule. A user creates tasks (name, start time, duration, alert interval) by clicking a slot on a day timeline (or an "Add new task" button), starts one from its own card, and the app counts down, firing a periodic alert at the chosen interval (tone, escalating tone, or spoken voice) and a distinct completion sound/announcement when time runs out.

There is no backend. Everything — UI, logic, and persistence — lives in one HTML file.

## Tech stack

- Plain HTML + CSS + vanilla JavaScript. No frameworks, no build step, no package.json.
- No external libraries or CDN imports.
- Persistence via plain `localStorage` (see below — this replaced the earlier `window.storage`-based approach, which only worked inside the Claude Artifacts sandbox and didn't persist in the real Vercel deployment).
- Audio via the Web Audio API (`AudioContext`), generated in-code (oscillators) — no audio files.
- Spoken alerts via the `SpeechSynthesis` API — no audio files or TTS backend calls; uses whatever voices the browser/OS exposes.
- Background-safe ticking via a dedicated `Worker` (created from an inline `Blob`), because `setInterval` on the main thread gets throttled when the tab is backgrounded.
- Best-effort `navigator.wakeLock` to keep the screen awake while a timer runs.
- Browser `Notification` API as a backup alert channel alongside sound.

## File structure

```
index.html   # the entire app: markup, styles, and script in one file
CLAUDE.md    # this file
README.md    # user-facing feature summary
```

There is nothing to install and nothing to build. To preview changes, just open `index.html` directly in a browser (or reload it in whatever environment is serving it).

## How to run / test

- No dev server or build tooling — open the file in a browser.
- Manual test checklist after changes:
  1. Click an hour label or empty spot on the day timeline (or "+ Add new task" under Settings) — the modal should open, pre-filled with that start time (blank if opened via the button).
  2. Add a task (name + start time + hours/minutes + alert interval) — it should appear both in the task list and as a sized block on the day timeline at the right position.
  3. Click the ▶ icon on a task card — it should start, and the pause/stop icon should immediately show as enabled (❙❙ Pause), not stay disabled. The timeline block should switch to its "running" style.
  4. Let an interval alert fire — confirm the correct alert plays for the selected Settings → Alert sound mode (same tone / escalating tone / human voice).
  5. If "Human voice" is selected, confirm it speaks only the time (e.g. "15 minutes.") — never the task name.
  6. Try the voice accent dropdown (only visible when "Human voice" is selected) — confirm it lists real installed voices and previews on change.
  7. While a task is running or paused, click one of the +1/+5/+10/+15 min extend buttons — confirm the countdown, progress bar, and timeline block height all grow immediately, and that the extra minutes are still reflected after stopping/restarting or reloading the page.
  8. Let the timer reach zero — confirm the completion chime (or "Time is up." if in voice mode) plays, the card shows "Time is up", and the timeline block shows the on-time/extra-time status bar.
  9. Pause, resume (via ▶), and stop (via the same icon showing ■ once paused) from the task's own icon row.
  10. Delete (×) a task while it's active — confirm the timer actually stops (worker stopped, wake lock released), not just removed from the list and timeline.
  11. Reload the page — tasks, sound-mode, and voice-accent preferences should still be there (loaded from `localStorage`).
  12. Check the "Enable notifications" button — it should only be visible when permission is still `"default"`; once granted/denied for that origin it correctly disappears.
- There is no automated test suite. If one is ever added, document how to run it here.

## Architecture notes

Everything runs in one `<script>` block at the bottom of the file. Key pieces:

- **Timer state** is a small set of module-level `let` variables (`totalSeconds`, `remaining`, `intervalSec`, `timerId`, `isPaused`, `endTime`, `activeTaskId`, `soundMode`, `selectedVoiceKey`, etc.). Only **one task can be active at a time** — `activeTaskId` tracks which.
- **`tick()`** recomputes `remaining` from `endTime - Date.now()` every second (driven by the worker's `postMessage`) rather than decrementing a counter, so background-tab throttling can't cause drift.
- **`startTimer(name, durationSeconds, intervalMinutes, taskId)`** is the single entry point for starting a countdown, called from a task's own ▶ icon. It sets `timerId = true` **before** calling `renderTasks()` — this ordering matters: rendering before setting `timerId` was a bug that left the pause/stop icon looking disabled on the very first click.
- **`togglePause()`** and **`stopTimer()`** are shared functions used by the per-task icon buttons.
- **`extendTimer(minutes)`** adds minutes to the *currently active* task's countdown mid-run (called from the +1/+5/+10/+15 buttons in its clock box). It adjusts `totalSeconds`/`remaining`/`endTime` depending on whether the timer is paused or running, re-anchors the alert-interval bucket so the extension doesn't itself trigger an alert, and — importantly — persists the added minutes onto `task.extendedMin` (not just transient runtime state), so the extra time and the "+X min" shown for it survive stop/restart/reload. Both `renderTasks()` (task card duration math) and `renderDayTimeline()` (block height) read `task.extendedMin`.
- **The running clock/status/progress bar live inside the active task's own card.** `renderTasks()` conditionally injects a `.task-clock-box` (with `#statusMsg`, `#clock`, `#progressBar` — reused IDs, safe because only one task is ever active) below the task's meta line, sized to the **full width of the card** — not indented under a narrower column. Because of this, `startTimer()` calls `renderTasks()` before touching those elements, and `tick()`/`updateDisplay()` null-check them defensively in case a re-render hasn't happened yet.
- **On completion**, `activeTaskId` is deliberately *not* cleared — the task stays "active" showing "Time is up" until the user explicitly stops (■) or restarts (▶) it from its own card.
- **Sound**: `playAlertTone(progress, message)` routes to one of three places based on `soundMode`:
  - `"same"` → `beep()` — a calm, fixed-pitch "tick tick".
  - `"escalating"` → `beepEscalating(progress)` — pitch/waveform intensifies with `progress` (0→1), always exactly 2 ticks.
  - `"voice"` → `speak(message)` — spoken via `SpeechSynthesis`. Voice-mode messages are deliberately minimal: **just the time** (e.g. `"15 minutes."`, `"Time is up."`) — never the task name.
  - `beepDone()` is a separate, distinct 3-note descending chime reserved for task completion in tone modes; in voice mode, completion instead speaks `"Time is up."` (see `tick()`).
- **Voice selection** (`pickVoice()` / `populateVoiceSelect()`): the `#voiceSelect` dropdown (only shown when `soundMode === "voice"`) is populated dynamically from every **English** `SpeechSynthesis` voice actually installed on the device (`v.lang` starting with `"en"`), not a hardcoded list — quality/availability varies a lot by browser/OS (e.g. Chrome/Edge expose "Google …" voices requiring network access; Safari/Firefox only expose OS-native voices). `"Default (Auto)"` (empty `selectedVoiceKey`) falls back to preferring `en-US`, then any English voice, then whatever's first. Voice list loads asynchronously (`speechSynthesis.onvoiceschanged`), so the dropdown repopulates when it becomes available.
- **Day timeline** (`renderDayTimeline()`): renders a 24-hour vertical schedule — an hour-label sidebar plus a track where each task with a `scheduledTime` is drawn as an absolutely-positioned block, `top`/`height` computed from its start time and total duration (`durationMin + durationSec/60 + extendedMin`, `TIMELINE_HOUR_HEIGHT` px per hour). Overlapping tasks are assigned side-by-side columns via a greedy interval-scheduling pass (`columnEnds`). Blocks get `.active`/`.running`/`.done` styling mirroring the task card's state. A completed task's block additionally renders a two-segment status bar (green = on-time portion, orange = extra-time portion, sized by `extendedMin`'s share of total time taken).
- **Click-to-create modal** (`openTaskModal()` / `closeTaskModal()` / `#taskModalOverlay`): the only way to add a task. Opened three ways — clicking an hour label in the timeline sidebar, clicking a spot on the timeline track (`Math.floor` of the click's Y-position into an hour), or the "+ Add new task" button under Settings (opens with a blank start time). Fields: task name, start time, duration as **Hours + Minutes** (no seconds in the UI — `durationSec` is always saved as `0` for tasks created through the modal), and alert interval. `#modalAddBtn`'s handler validates a non-empty name and non-zero total duration before calling `addTask(...)`.
- **`addTask(name, scheduledTime, durMin, durSec, interval)`** pushes a new task object (with a generated id and `extendedMin: 0`) and re-renders both the list and the timeline.
- **Persistence**: three `localStorage` keys (kept as the original `reminder-timer-*` names even after the MyCue rebrand, since renaming them would strand existing users' saved data):
  - `reminder-timer-tasks` — JSON array of task objects `{ id, name, scheduledTime, durationMin, durationSec, interval, extendedMin }`.
  - `reminder-timer-sound-mode` — `"same"` | `"escalating"` | `"voice"`.
  - `reminder-timer-voice-name` — `""` (auto) or a `"name__lang"` composite key identifying a specific installed voice.
  - `loadTasks()` runs three migrations on load for tasks saved by older versions of the app: pre-split `duration` (minutes-only) → `durationMin`/`durationSec`; missing `scheduledTime` → `""`; missing `extendedMin` → `0`.
- **No drag-and-drop, no separate timer panel, no inline add-task form.** These existed in earlier versions and were intentionally removed — starting a task is only ever done via its own ▶ icon, and creating a task is only ever done via the click-to-create modal (`openTaskModal`). Don't reintroduce a `#timerCard` drop target or a `#addTaskForm` inline form unless explicitly asked.

## Current UI layout

A `.site-header` (inline SVG logo + "MyCue" wordmark) sits above the main content, followed by the two-column layout (`.app`): a "Today's tasks" card on the left, a "Today's schedule" (day timeline) card on the right. Below 720px wide, the two cards stack vertically.

**"Today's tasks" card**, top to bottom:
1. **Settings card** (nested, boxed, collapsible accordion — click the "Settings" header to expand/collapse):
   - Alert sound radios: Same tone / Escalating tone / Human voice.
   - Accent `<select>` (`#voiceSelect`), only visible when "Human voice" is selected, listing installed English voices.
   - Notifications: "Enable notifications" button + status text — button is conditionally hidden once permission is already granted or denied for the origin.
2. **"+ Add new task" button** (`#addTaskBtn`) — sits directly under the Settings card; opens the same click-to-create modal used by the timeline, with a blank start time.
3. **Task list** (`#taskList`) — each task card:
   - Row 1 (single line): task name, then 3 icon buttons — ▶ Play/Resume, a **merged pause/stop icon** (❙❙ while running → ■ once paused), and × Close/Delete.
   - Row 2: meta line — `"X min [+Y min] · alert every Z min"`, with `" · running"` appended while active.
   - Row 3 (only when active): the full-width `.task-clock-box` (status text, big clock, progress bar, and — while running/paused — the +1/+5/+10/+15 min extend button row).

**"Today's schedule" card**: a scrollable 24-hour vertical timeline (`#dayTimeline`) — an hour-label sidebar (click a label to open the create-modal at that hour) plus a track where task blocks are positioned/sized by start time and duration, side-by-side when overlapping.

**Click-to-create modal** (`#taskModalOverlay`, hidden until opened): name, start time, duration (Hours + Minutes fields), alert interval, Cancel/Add buttons.

## Conventions to follow when editing

- Keep everything in the single `index.html` file unless explicitly asked to split it up.
- Persistence uses `localStorage` directly — wrap calls in try/catch (private browsing / storage-disabled contexts can throw), but there's no longer a sandbox-storage abstraction to route through.
- The task-list card heading stays "Today's tasks" — it was tried as "MyCue" and reverted; don't change it without being asked.
- Keep tone-based alerts implemented as Web Audio oscillator code, not audio file assets. Keep voice alerts as `SpeechSynthesis`, not TTS audio files or network calls.
- Voice-mode spoken text stays minimal (time only) — don't add the task name or other detail back into what gets spoken unless asked.
- Don't build accent/voice filtering logic that singles out or excludes any specific accent/nationality (e.g. excluding Indian English) — list what's available, or curate positively (a preferred/known-good list), never exclude by nationality.
- When adding new per-task UI, remember only one task is ever "active" — reused fixed IDs (`#clock`, `#statusMsg`, `#progressBar`) inside the active card are intentional, not a bug.
- Task duration is entered as **Hours + Minutes** in the create modal (`#modalDurationHour`, `#modalDurationMin`) — there is no seconds field in the UI. Internally tasks still store `durationMin`/`durationSec` (seconds always `0` for newly created tasks) so existing duration math (`formatDuration`, timeline sizing, countdown) doesn't need to change.
- Match the existing visual style: soft neutral palette (`#f5f5f2` background, white cards, `#222` as the primary dark accent), rounded corners, no external fonts/icons (icons are inline Unicode/HTML entities, e.g. `&#9654;` for ▶).

## Open items / known gotchas

- Voice quality/accent availability is entirely dependent on the browser and OS the person is using — there's no way to guarantee a specific voice (e.g. "Google US English") is available on every device; the UI is built to degrade gracefully (dynamic list + "Default (Auto)" fallback) rather than assume anything is present.
- `localStorage` is per-origin and per-browser: switching devices/browsers, or clearing site data, loses tasks and settings. There's no sync or backend.
