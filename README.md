# Life Manager Vault

Welcome to your personal Obsidian vault. It centralizes finances, tasks, investments, and profile data in one place powered by Dataview, Meta Bind, and Templater.

## 📁 Structure

- `Landing.md`: the main hub with module buttons and the “Overview” panel (avatar, name, metrics for tasks/finance/investments).
- `Finance.md` + `finance/<year>/<Month>.md`: dashboards and monthly notes (use English month names, e.g., `finance/2025/November.md`).
- `Todo.md` + `todo/tasks.md`: task manager where states persist in the frontmatter.
- `Investments.md` + `investments/*.md`: per-investment notes with movement logs, growth chart, and form to update totals.
- `Training.md` hub + `training/exercises/`: workout dashboard with exercise creator, per-exercise session lists, calendar heatmap, and inline charts.
- `Books.md` + `books/`: reading tracker with per-book progress bars and standalone chapter logs.
- `Config.md` + `config/settings.md`: UI preferences (currently language selection) that cascade through the dashboards.
- `profile/`: stores `stats.md` (user data) and `pfp.*` (avatar rendered on the Landing page).
- `templates/`: Templater files used by the Meta Bind buttons (new finance month, new investment, etc.).

## 🚀 How to use

1. **Landing / Overview**
   - Update `profile/stats.md` with `- name: Your Name` and add a picture at `profile/pfp.png|jpg|jpeg|webp|gif`.
   - The panel shows:
     - Amount invested in the current month (sums contributions tagged with the current date in `investments/`).
     - Open tasks (todo/daily/weekly/monthly) using the persisted state in `Todo.md`.
     - Current month financial balance (Income – Expenses of the active note in `finance/`).

2. **Finances**
   - Click “New finance month” in `Finance.md` to create a note from `templates/new finance month.md`.
   - Inside each month, log lines as `category:: value #tag`. The dashboards read every category to build tables and charts.

3. **Tasks**
   - Add tasks to `todo/tasks.md` under `## todo`, `## Daily`, `## Weekly`, and `## Monthly`.
   - `Todo.md` renders the lists and stores their state in `todoStatus`, `dailyStatus`, etc. Removing a line also clears its saved state.

4. **Investments**
   - Every note under `investments/` contains `## Movements` with lines like `- value #YYYY-MM-DD (#initial optional)`.
   - In `Investments.md`, type the new total amount; the script saves only the delta with today’s date and (optionally) your custom tag.
   - A Chart.js line chart tracks the last 12 months of growth per investment.

5. **Training**
   - Open `Training.md` to create exercises (notes live under `training/exercises/` and ship with a volume chart).
   - Each exercise note keeps its own `## Sessions` list via lines like `- date:: 2025-11-10 load:: 100 reps:: 5`.
   - The Training hub aggregates all exercises to show last sessions, total volume, and a calendar heatmap, while each note renders its personal chart + table.

6. **Books**
   - Open `Books.md` to register a book by name + total chapters; notes live under `books/`.
   - Use **Read chapter/Ler capítulo** on a card to append the next `- chapter:: N finished:: ISO` line, or the standalone form for quick “finished in” logs.
   - Every logged chapter (book or standalone) gives you +20 XP automatically.

7. **Config**
   - Open `Config.md` to choose the interface language (English or Portuguese) and preferred currency (Real or Dollar). These options only change UI labels/symbols—logic and filenames stay the same.
   - Preferences are stored in `config/settings.md`, so they sync across devices with the vault.

8. **Templates**
   - `templates/new finance month.md`: default structure for Expenses/Income.
   - `templates/new training exercise.md`: skeleton for every exercise note (volume chart included).
   - Additional templates can be triggered via Meta Bind for new investments or other workflows.

## ✅ Requirements

- Obsidian with **Dataview**, **Meta Bind**, and **Templater** enabled.
- Monthly notes must use English month names so the Landing dashboard can find them via `moment().format("MMMM")`.

## 👋 Welcome

Open `Landing.md`, set your name/avatar, and start with the main buttons:

1. Create the current month via **Finance** (“New finance month” button).
2. Register your tasks in `todo/tasks.md` and follow progress in **To-do**.
3. Log investments in `investments/` and monitor them inside **Investments**.

That’s it! Your financial and productivity routine stays in sync every time you open Obsidian. Happy tracking! 🧠📈
