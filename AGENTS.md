# CLAUDE.md

This vault uses Claude/Codex-style guidance to operate the Life Manager Obsidian project. Keep this document updated as the automations evolve.

## Repository Overview

Life Manager is an Obsidian vault that consolidates finances, tasks, investments, workouts, and personal profile data. DataviewJS dashboards, Meta Bind buttons, and Templater templates automate note creation, XP tracking, and multilingual UI labels.

### Project Stats
- Primary environment: Obsidian 1.5+ with Dataview, Meta Bind, and Templater
- Languages: Markdown + Dataview JavaScript snippets
- Data domain: personal finance, productivity, training logs
- Storage: plain Markdown under `finance/`, `todo/`, `investments/`, `training/`, `profile/`

## Quick Start

### Prerequisites
- Obsidian desktop/mobile with the vault folder opened directly
- Community plugins enabled: Dataview, Meta Bind, Templater (Buttons is bundled into Meta Bind)
- Optional: ability to open Chart.js CDN (for investment charts)

### Initial Setup
```bash
# Clone or update the vault
git clone <repo-url> life-manager
cd life-manager
```
1. Open the folder as an Obsidian vault.
2. Enable Dataview, Meta Bind, and Templater in Settings → Community Plugins.
3. Copy a profile photo to `profile/pfp.png|jpg|jpeg|webp|gif` and set your stats via `profile/stats.md`.
4. Adjust language/currency in `config/settings.md` by opening `Config.md`.
5. Visit `Landing.md` to ensure the Overview widget renders name, XP, and the metric cards.

### Daily Use
- Use the **New finance month**, **New investment**, and **New training exercise** Meta Bind buttons to scaffold notes.
- Edit `todo/tasks.md` to add routines; toggle completion in `Todo.md`.
- Log exercises inside the per-exercise notes under `training/exercises/`.

## Essential Commands

### Git & Vault Maintenance
```bash
git status -sb                    # Inspect uncommitted vault changes
git pull --rebase origin main     # Sync before editing dashboards
git add -A && git commit -m "Feat: describe change"  # Save work using the existing "Feat:" style
git push origin main              # Publish updates
```

### Obsidian Tasks
- `Ctrl/Cmd+P → Reload app without saving` after Templater changes to refresh scripts.
- `Dataview: Reload Dataview` command if dashboards stop updating.

## Architecture & Key Concepts

### Landing Hub (`Landing.md`)
- Central dashboard that composes name/avatar from `profile/stats.md` and `profile/pfp.*`, XP from the same stats note, tasks from `Todo.md`, finance balance from the current month note under `finance/<year>/<Month>.md`, investments tagged with the current date, and current-month training sessions.
- Implements bilingual UI strings (EN/PT) and currency prefix formatting (BRL/USD) driven by `config/settings.md`.
- XP drives leveling (`level = floor(xp / 1000)`), with progress displayed via a custom progress bar.

### Finance System (`Finance.md`, `finance/<year>/<Month>.md`, `config/consts.md`)
- `Finance.md` renders a yearly overview, per-month cards, and inline forms to append `- category:: amount #tag` entries under `## Income` and `## Expenses`.
- Categories for the select dropdowns are pulled from `config/consts.md` (`expenseCategories` & `incomeCategories` lists).
- Monthly notes must live in `finance/<year>/<English Month>.md` even if the title uses another language. The current month is resolved with `moment().format("MMMM")`.

### Task Manager (`Todo.md`, `todo/tasks.md`)
- `todo/tasks.md` holds raw bullet lists inside `## todo`, `## Daily`, `## Weekly`, and `## Monthly`.
- `Todo.md` persists checkbox state inside frontmatter maps (`todoStatus`, `dailyStatus`, etc.) and awards 50 XP per completion by rewriting `profile/stats.md`.
- Cycle-based sections (daily/weekly/monthly) store the last-complete date to auto-reset checkboxes when the cycle changes; clearing a line in `todo/tasks.md` removes its saved state.

### Investments (`Investments.md`, `investments/*.md`)
- Each investment note stores `## Movements` lines (`- amount #YYYY-MM-DD #tag`). Only deltas are appended when you input a new total in `Investments.md`.
- The dashboard loads Chart.js from `https://cdn.jsdelivr.net/npm/chart.js` to plot the last-12-month contribution streak. If offline, charts simply fail silently; note this under troubleshooting.

### Training Hub (`Training.md`, `training/exercises/*.md`)
- Exercises live as individual notes created from `templates/new training exercise.md`. Their `## Sessions` list uses `- date:: YYYY-MM-DD load:: N reps:: N notes:: optional`.
- `Training.md` can create new exercises, log sessions, award 10 XP per session, show a 60-day heatmap, and tabulate recent sessions across all exercises.

### Config & Profile (`Config.md`, `config/settings.md`, `profile/stats.md`)
- `Config.md` exposes language and currency dropdowns that update `config/settings.md` frontmatter via the Obsidian API.
- `profile/stats.md` is a YAML-ish bullet list (`- name:`, `- xp:`). Every automation expects both keys; xp mutations append/update the `- xp:` line.

### Templates (`templates/*.md`)
- Provide consistent scaffolding for finance months, investment notes, todo sections, stats seed data, and training exercises with pre-wired Dataview charts.
- Meta Bind buttons reference these templates; renaming or moving them requires updating the button definitions embedded in the respective dashboard files.

## Project Structure
```
.
├── Landing.md                # Overview dashboard
├── Finance.md                # Finance hub + entry forms
├── Todo.md                   # Task board with persisted state
├── Investments.md            # Investment tracker and chart
├── Training.md               # Training dashboard + exercise creator
├── Config.md                 # Language & currency controls
├── README.md                 # High-level usage primer
├── config/
│   ├── settings.md           # Frontmatter storing language/currency
│   └── consts.md             # Income/expense category lists
├── finance/<year>/<Month>.md # Monthly ledgers (Expenses/Income)
├── investments/*.md          # Per-investment movement logs
├── todo/tasks.md             # Source lists consumed by Todo.md
├── profile/
│   ├── stats.md              # Name + XP
│   └── pfp.jpg               # Avatar used on the landing page
├── training/exercises/*.md   # Per-exercise notes with sessions
└── templates/*.md            # Templater sources for buttons
```

## Important Patterns

### Data Entry Formats
- Finance lines: `- category:: 123.45 #optional-tag` under the correct heading. Amounts stored as decimal strings; tags are slugified.
- Task lines: plain bullets, optional `#YYYY-MM-DD` or `#HH:MM` tags for metadata.
- Investment movements: `- 100 #YYYY-MM-DD #tag`; the dashboard calculates totals from the raw list.
- Training sessions: `- date:: 2025-11-10 load:: 100 reps:: 5 notes:: optional`.

### Localization & Currency
- `config/settings.md` frontmatter controls UI strings (en/pt) and currency formatting (BRL/USD). Always mutate via `Config.md` to keep observers in sync.
- Month detection relies on English month folder names regardless of localized note titles.

### State Persistence
- Task completion state lives in `Todo.md` frontmatter maps keyed by generated slugs.
- XP totals reside in `profile/stats.md`; both Todo and Training modules rewrite the `- xp:` line, so keep it unique and near the top of the file.

## Development Workflows

1. **Create a Finance Month**
   - Click **New finance month** in `Finance.md`.
   - Rename the generated file from `finance/<year>/month.md` to `finance/<year>/<English Month>.md`.
   - Log `Expenses`/`Income` lines directly or via the inline form; keep categories in sync with `config/consts.md`.
2. **Add Investments**
   - Use **New investment** on `Investments.md` to scaffold a note from `templates/new investment.md`.
   - When the asset total changes, enter the new total in the dashboard form; the script appends the delta tagged with today’s date.
3. **Manage Tasks**
   - Maintain raw lists in `todo/tasks.md`.
   - Toggle completion in `Todo.md`; XP updates automatically. Removing a task bullet also clears stored status.
4. **Track Training**
   - Create exercises via **New training exercise** inside `Training.md`.
   - Log sets via the dashboard form to maintain consistent formatting and XP.
5. **Adjust UI Preferences**
   - Open `Config.md` and change language/currency; the script writes to `config/settings.md` and prompts you to reload notes.

## Dependencies & External Services
- **Obsidian API** (`app.vault`, `app.fileManager`, `Notice`) is invoked inside DataviewJS blocks.
- **Dataview** supplies `dv.pages`, `dv.io.load`, and `dv.container`.
- **Meta Bind** powers the declarative buttons that call Templater or open notes.
- **Templater** populates new note skeletons.
- **Chart.js** is fetched from the CDN for investment and exercise charts; without connectivity the metrics render but charts do not.
- **Moment.js** (bundled with Obsidian) handles all date math and formatting.

## Testing & Quality
- No automated tests exist; validation is experiential.
- Recommended manual checks per change:
  - Reload Dataview (`Cmd/Ctrl+R` or the Dataview reload command) to ensure scripts execute without console errors.
  - Create a sandbox finance note using the button to verify templater placeholders still resolve.
  - Toggle a representative task/exercise entry to confirm XP mutations and state persistence.
  - Inspect the developer console for Chart.js/network errors when modifying investment visuals.
- Quality gates rely on linting-by-convention: keep Markdown headings stable, avoid renaming folders referenced by scripts, and keep `config/settings.md` frontmatter valid YAML.

## Git History & Evolution
- `a22b0bf`: repository bootstrapped with finance button scaffolding.
- `50b842a`, `edd5acd`, `5233b2b`: incremental addition of investment, todo, and training modules.
- `f2a7e9c`: XP handling introduced and later expanded (`d6eb5d7` ties exercises to XP).
- `898fe15`: landing dashboard revamp; `5917725` adds localization/config module.
- Commit messages follow a `Feat: <description>` convention; keep future commits aligned for consistency and searchability.

## Hidden Context & Gotchas
- Finance month detection expects English folder names even if the note title is localized; mismatched names break the Landing balance card.
- Investment charts fail silently without CDN access; note this when working offline.
- Task IDs derive from slugified text. Editing the text of an existing task produces a new ID, effectively resetting its stored completion history.
- XP writes append the `- xp:` line if it disappears; do not convert `profile/stats.md` to frontmatter.
- Buttons rely on Meta Bind’s `templaterCreateNote` action; disablement or renaming the template paths breaks note creation.

## Troubleshooting
1. **Landing metrics show `—`**
   - Confirm `finance/<year>/<Month>.md` exists using the current English month name.
   - Check `investments/*` for at least one `#YYYY-MM-DD` tag in the current month if “Invested this month” stays at zero.
2. **XP stops updating**
   - Ensure `profile/stats.md` still contains a single `- xp:` line.
   - Verify Obsidian has permission to modify files (sync conflicts can lock the vault).
3. **Charts missing**
   - Reconnect to the internet or bundle Chart.js locally; otherwise accept text-only summaries.
4. **Finance form shows no categories**
   - Validate `config/consts.md` still lists bullet items beneath `## expenseCategories` and `## incomeCategories`.

## Maintenance Tasks
- Monthly: create the new finance note, archive/trim old tasks, and review investment deltas for accuracy.
- Quarterly: refresh category lists in `config/consts.md`, ensure templates match real usage, and prune stale exercises.
- As needed: replace `profile/pfp.*`, reset XP (`profile/stats.md`) if you want to rebalance leveling, and review `.gitignore` to keep private data excluded.

## Validation Questions
1. Are there any critical workflows or patterns I missed?
2. Any project-specific conventions that should be highlighted?
3. Are the commands accurate for your development setup?

