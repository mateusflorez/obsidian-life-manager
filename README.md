# Life Manager Vault

## 1. Welcome

Life Manager is an Obsidian-first dashboard that centralizes finances, tasks, investments, workouts, books, mood tracking, pomodoro focus timers, and achievements. DataviewJS, Meta Bind, and Templater power the automations that create notes, persist state, and keep XP totals in sync. Open `Landing.md`, set your name/avatar in `profile/stats.md`, and use the quick buttons to jump into each module.

## 2. How to Update the Vault

1. Use git source control of git plugin to pull the latest changes
2. Or manually download the latest release from the GitHub repository

## 3. How to Use Each Module

### Landing
- Edit `profile/stats.md` (`- name:` and `- xp:`) and place an avatar at `profile/pfp.png|jpg|jpeg|webp|gif`.
- `Landing.md` shows your XP/level, avatar, and cards for monthly investments, open tasks, training sessions, chapters read, financial balance, and a quick-launch strip to every module.

### Finance & Credit Cards
- Click **New finance month** to scaffold `finance/<account>/<year>/<English Month>.md` from the template.
- Log income/expenses as `- category:: value #optional-tag`.
- The yearly dashboard renders a summary table, a trend chart, and per-month cards with inline “Add income/expense” forms.
- **Credit cards** (`finance/cards/`) store limit, close/due dates, and charges. Logged charges are mirrored into the monthly note using tagged expenses.
- **Recurrences** (`finance/recurrences.md`) define `- id:: ... title:: ... value:: N` entries that auto-populate each month. You can pause/resume them from the dashboard.
- A pie chart inside the current month card visualizes category splits for the ongoing month.

### Tasks
- `todo/tasks.md` contains the headings `## todo`, `## Daily`, `## Weekly`, `## Monthly`. The dashboard auto-creates this note with empty headings if it is missing.
- Manage text-only bullets in that file; `Todo.md` renders them with checkboxes and awards 50 XP per completion, saving state in `todo/state.json`.
- Removing a bullet also clears its stored completion history.

### Investments
- Each note under `investments/` stores `## Movements` lines like `- 100 #2025-03-10 #bonus`.
- Inside `Investments.md`, enter the new total to append only the delta tagged with today’s date; the dashboard charts the last 12 months using Chart.js.

### Training
- Use **New training exercise** to create notes in `training/exercises/`.
- Log sessions with `- date:: YYYY-MM-DD load:: N reps:: N notes:: optional`.
- The hub lists exercises, aggregates sessions, awards 10 XP per log, and renders a 60-day heatmap.

### Books
- Register a book in `Books.md` (name + total chapters). Notes live under `books/` with `## Chapters` lines `- chapter:: N finished:: ISO`.
- The “Read chapter” button appends the next entry; the standalone form logs one-off chapters.
- Each logged chapter grants 20 XP.

### Mood
- `Mood.md` offers a slider (1–5) plus an optional note field. Saving writes `- date:: YYYY-MM-DD mood:: N note:: text` to `mood/log.md`, displays a 60-day trend chart, shows recent logs, and adds 10 XP automatically.

### Achievements
- `Achievements.md` pulls live totals for chapters, investments, tasks, training sessions, mood logs, and XP levels.
- Cards progress through color tiers (gray → orange) and display trophies, progress bars, and completion badges.

### Pomodoro
- `Pomodoro.md` lets you choose the focus and break lengths (for example, 15 minutes on / 5 minutes off) plus how many cycles you want to run.
- Every finished focus block logs a line inside `pomodoro/log.md`, so the dashboard can total lifetime minutes, highlight today’s focused time, and list the latest sessions.
- The timer plays a brief tone and triggers a desktop notification at each focus/break transition, so you still get nudged even if Obsidian is unfocused.

### Config & Profile
- `Config.md` writes language/currency preferences to `config/settings.md`; the rest of the vault reads those values for UI strings and currency symbols.
- `profile/stats.md` must keep a single `- name:`.
- avatar image files go in `profile/pfp.*`.

### Templates
- `templates/` houses the Templater sources used by Meta Bind buttons:
  - `new finance month.md`
  - `new training exercise.md`
  - `new investment.md`
  - `new task manager.md` (reference for the section headings)

## 4. Directory Reference

- `Landing.md` — overview dashboard.
- `Finance.md` / `finance/<account>/<year>/<Month>.md` — finance data + credit cards + recurrences.
- `Todo.md` / `todo/tasks.md` / `todo/state.json` — task lists and persisted states.
- `Investments.md` / `investments/*.md` — movement logs and charts.
- `Training.md` / `training/exercises/*.md` — workouts and per-exercise notes.
- `Books.md` / `books/` — book notes and chapter entries.
- `Mood.md` / `mood/log.md` — mood tracker + log.
- `Pomodoro.md` / `pomodoro/log.md` — focus timer with notifications + history.
- `Achievements.md` — milestone dashboard.
- `Config.md` / `config/settings.md` — localization + currency.
- `profile/` — stats and avatar.
- `templates/` — Templater sources for the Meta Bind buttons.

## 5. Requirements & Tips

- Enable **Dataview**, **Meta Bind**, and **Templater** in Obsidian.
- Monthly finance folders must use English month names so scripts that rely on `moment().format("MMMM")` can resolve the files.
- Chart.js is loaded from the CDN; when offline, charts silently fall back to text summaries.
- Reload Dataview (Cmd/Ctrl+R) whenever you edit DataviewJS blocks to avoid stale data.

Happy tracking! 🎯
