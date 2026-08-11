# Reminder Timer

A timer that reminds you to stay on task at set intervals until time's up.

Add a task with a duration and an alert interval, then start it — the app counts down and fires a periodic alert every X minutes (e.g. every 15 minutes) to keep you on track, plus a distinct sound or announcement when time's up. Pause and resume anytime, and run through as many tasks as you like, one at a time.

## Features

- Add tasks with a name, total duration, and custom alert interval (minutes + seconds)
- Three alert modes: same tone throughout, an escalating tone, or a spoken voice announcing the time
- Tone alerts generated in-browser (Web Audio API, no audio files); voice alerts via the browser's built-in text-to-speech (SpeechSynthesis) — pick your preferred installed voice from the accent dropdown
- Pause and resume, with a merged play/pause/stop control per task
- Live countdown, progress bar, and status shown right on the task card
- Optional browser notifications as a backup alert channel
- Keeps the screen awake while a timer is running (Wake Lock API)
- Background-safe countdown via a Web Worker, so it won't drift if the tab is backgrounded
- Tasks and settings are saved automatically and persist between visits

## Usage

Open `reminder-timer.html` in any browser — no build step, no dependencies.

## Deploy

Deployed as a static site on [Vercel](https://vercel.com).

## Built with

Built using [Claude](https://claude.ai).
