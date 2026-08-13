# MyCue

A timer that reminds you to stay on task at set intervals until time's up — laid out as a daily schedule.

Click a time slot on the day timeline (or use the "+ Add new task" button) to create a task with a duration and an alert interval, then start it — the app counts down and fires a periodic alert every X minutes (e.g. every 15 minutes) to keep you on track, plus a distinct sound or announcement when time's up. Pause, resume, or extend the countdown anytime, and run through as many tasks as you like, one at a time.

## Features

- **Day timeline schedule** — a scrollable 24-hour view where each task appears as a block sized by its duration; overlapping tasks lay out side by side like a calendar
- **Click-to-create** — click any hour on the timeline (or the "+ Add new task" button under Settings) to open a popup pre-filled with that start time
- Add tasks with a name, start time, total duration (hours + minutes), and custom alert interval
- Three alert modes: same tone throughout, an escalating tone, or a spoken voice announcing the time
- Tone alerts generated in-browser (Web Audio API, no audio files); voice alerts via the browser's built-in text-to-speech (SpeechSynthesis) — pick your preferred installed voice from the accent dropdown
- Pause and resume, with a merged play/pause/stop control per task
- **Extend a running/paused task** by +1, +5, +10, or +15 minutes without stopping it — the added time is saved permanently on the task
- Live countdown, progress bar, and status shown right on the task's own card and reflected on its timeline block
- Completed tasks show an on-time vs. extra-time indicator bar on the timeline
- Optional browser notifications as a backup alert channel
- Keeps the screen awake while a timer is running (Wake Lock API)
- Background-safe countdown via a Web Worker, so it won't drift if the tab is backgrounded
- Tasks and settings are saved automatically and persist between visits

## Usage

Open `index.html` in any browser — no build step, no dependencies.

## Deploy

Deployed as a static site on [Vercel](https://vercel.com).

## Built with

Built using [Claude](https://claude.ai).
