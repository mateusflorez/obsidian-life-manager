# CLAUDE.md

Guidance for Codex/Claude agents working inside the Life Manager Obsidian vault. Read this first to understand the vault’s moving parts, workflows, and guardrails.

## Repository Overview
- Life Manager is an Obsidian-based personal operations hub that tracks finances, investments, and routines through Dataview, Meta Bind, and Templater automations (`README.md:3-38`).
- Primary artifacts are Markdown notes enriched with DataviewJS code blocks and Meta Bind buttons; no external backend or build tooling exists.
- Key modules: `Landing.md` (overview dashboard), `Finance.md` + `finance/<year>/<Month>.md` (monthly ledgers), `Todo.md` + `todo/tasks.md` (task board with XP), `Investments.md` + `investments/*.md` (per-investment logs), `profile/` (avatar + stats), and `templates/` (Templater seeds) (`README.md:7-39`).
- Personal data in `finance/`, `todo/`, `investments/`, and `profile/` is considered sensitive and is ignored by git via `.gitignore` rules (`.gitignore:1-5`).

### Project statistics
- Primary language: Markdown + DataviewJS/Meta Bind glue.
- Lines of Markdown/JS: ~3k.
- Active branch: `main`; commits follow “Feat: …” summary style (`git log --oneline`).
- Maintainer: single user (Mateus per `profile/stats.md:1-2`).

## Quick Start
### Prerequisites
- Obsidian desktop/mobile with the Dataview, Meta Bind, and Templater community plugins enabled (`README.md:42-44`).
- Moment.js is bundled with Obsidian; Chart.js loads from CDN inside the dashboards (`Investments.md:41-55`).
- Local cloning ability (git) and permission to store personal finance data.

### Initial setup
```bash
# Clone and enter the vault
git clone <repo-url> life-manager
cd life-manager

# Optional: inspect status before editing
git status -sb
```

### First-time vault prep
1. Open the folder as an Obsidian vault and enable Dataview, Meta Bind, and Templater (matching README guidance, `README.md:14-44`).
2. Personalize `profile/stats.md` (`profile/stats.md:1-2`) and add an avatar image named `pfp.(png|jpg|jpeg|webp|gif)` under `profile/`.
3. Use the **New finance month** Meta Bind button in `Finance.md` to scaffold the current month from `templates/new finance month.md` (`Finance.md:39-57`, `templates/new finance month.md:1-9`).
4. Seed `todo/tasks.md` with routines or trigger the `templates/new task manager.md` template; task states persist in the `Todo.md` frontmatter (`todo/tasks.md:1-8`, `templates/new task manager.md:1-11`, `Todo.md:1-8`).
5. Create investment notes via the **New investment** button (template `templates/new investment.md`) before logging movements (`Investments.md:6-24`, `templates/new investment.md:1-4`).

## Essential Commands
### Repo hygiene
```bash
git status -sb                     # check for uncommitted note changes
rg -n "dataviewjs" *.md            # find dashboards using Dataview scripts
rg -n "::" finance/2025            # inspect ledger lines in monthly notes
rg -n "## Movements" investments   # audit movement sections across assets
```

### Content utilities
```bash
rg -n "## " todo/tasks.md          # ensure task headings follow conventions
rg -n "xp" profile/stats.md        # confirm XP counters updated after tasks
rg -n "moment().format" -g "*.md"  # see everywhere date logic relies on locale
```

Use `tree -L 2` (if installed) to visualize the directory layout before large edits.

## Architecture & Key Concepts
### Landing hub
- `Landing.md` renders the overview card: it loads profile data, avatar candidates, todo summaries, finance balances, and monthly investment totals via DataviewJS (`Landing.md:6-210`).
- Task stats come from parsing `todo/tasks.md`, respecting boolean vs. cyclical tasks and persisting state IDs (`Landing.md:63-118`).
- XP progress is derived from `profile/stats.md`, and the metric cards surface invested amount, open tasks, and month balance (`Landing.md:140-210`).

### Finance workspace
- `Finance.md` defines Dataview properties, a Meta Bind “New finance month” button, and a DataviewJS dashboard that aggregates yearly/monthly totals, category breakdowns, and Chart.js visualizations (`Finance.md:1-300`).
- Expense/income categories are sourced dynamically from `config/consts.md`, so extending categories only requires editing that file (`Finance.md:81-101`, `config/consts.md:1-15`).
- Monthly notes follow the format `finance/<year>/<Month>.md` with sections `## Expenses` and `## Income`, each containing `category:: value #tag` lines (`finance/2025/November.md:3-20`).

### Task workflow & XP loop
- `Todo.md` stores completion state in frontmatter maps (`todoStatus`, `dailyStatus`, etc.) and exposes a DataviewJS board that mirrors `todo/tasks.md` sections (`Todo.md:1-118`).
- Completing a task toggles state persistence via `app.fileManager.processFrontMatter` and awards 50 XP by editing `profile/stats.md` (`Todo.md:116-190`, `Todo.md:52-55`).
- Source tasks live in `todo/tasks.md` and must stay under the canonical headings for parsing to work (`todo/tasks.md:1-8`).

### Investments tracker
- `Investments.md` lists investment notes, injects Chart.js lazily from a CDN, parses `## Movements` blocks, and back-fills the last 12 months of growth per asset (`Investments.md:1-128`).
- Movements are bullet lines with an amount and tags; the `#YYYY-MM-DD` tag is how the dashboard anchors values in time (`Investments.md:73-105`, `investments/Piggy Bank.md:1-4`).
- The module can append new movement lines to a target note via Meta Bind actions, storing only the delta typed into the dashboard form (`Investments.md:107-188`).

### Profile & persona data
- `profile/stats.md` holds simple YAML-like bullet entries (`- name:`, `- xp:`) consumed by landing and XP incrementers (`profile/stats.md:1-2`, `Landing.md:22-31`, `Todo.md:176-190`).
- Avatar lookup follows a fixed filename priority order inside `profile/` (`Landing.md:32-47`).

### Templates & config glue
- Finance, investment, and task scaffolding lives under `templates/` and is referenced by Meta Bind buttons (`templates/new finance month.md:1-9`, `templates/new investment.md:1-4`, `templates/new task manager.md:1-11`).
- The stats template (`templates/stats template.md:1`) seeds the profile note.
- Global vocab such as category lists are centralized in `config/consts.md:1-15`.

## Project Structure
```
.
├── README.md                 # High-level usage guide and plugin requirements
├── Landing.md                # Overview dashboard (DataviewJS + Meta Bind buttons)
├── Finance.md                # Finance table, creation button, and analytics
├── finance/
│   └── 2025/
│       └── November.md       # Sample month note with Expenses/Income sections
├── Todo.md                   # Task dashboard with XP automation
├── todo/
│   └── tasks.md              # Source task definitions grouped by cadence
├── Investments.md            # Investments dashboard and Chart.js metrics
├── investments/
│   ├── Automatic Reserve.md  # Individual investment log with movements
│   ├── Home Purchase.md
│   └── Piggy Bank.md
├── profile/
│   ├── stats.md              # Name + XP counters
│   └── pfp.jpg               # Avatar source (any supported extension works)
├── templates/                # Templater snippets for finance/investments/tasks
└── config/
    └── consts.md             # Shared category lists
```

## Important Patterns & Workflows
- **Month naming:** finance notes must use English month names (`README.md:8`, `README.md:42-44`) so `moment().format("MMMM")` finds them.
- **Ledger syntax:** every ledger line follows `category:: value #optional-tag`; anything else is ignored by the parsers (`finance/2025/November.md:3-20`, `Finance.md:104-125`).
- **Task slugs:** IDs derive from lowercased text; editing a task’s wording creates a new slug and loses historical state, so edit carefully (`Todo.md:68-128`).
- **XP workflow:** XP increments by 50 when toggling from incomplete to done; rapid toggling can double awards, so avoid spam-clicking (`Todo.md:52-55`, `Todo.md:233-266`).
- **Data entry vs. automation:** `Finance.md` and `Investments.md` can both mutate target notes; keep Obsidian foregrounded until operations complete to avoid partial writes.
- **Git hygiene:** personal data dirs are ignored, so copying new months/investments from elsewhere may require `git add -f` if you intentionally want them tracked (`.gitignore:1-5`).

## Code Style & Conventions
- Markdown headings organize modules; DataviewJS blocks stay fenced with ```dataviewjs to enable execution (`Landing.md:6`, `Todo.md:31`, `Investments.md:26`).
- JavaScript follows ES modules style (arrow functions, `const`) and relies on Obsidian globals (`app`, `dv`, `window.moment`)—avoid importing external libraries.
- Frontmatter is YAML but stored as raw JSON-like maps (see `Todo.md:1-8`); always mutate via `app.fileManager.processFrontMatter` inside Obsidian scripts.
- Templates prefer lowercase filenames with spaces (Obsidian-friendly) while notes in data folders use Title Case for readability.
- Tags in movement/expense lines are single words (no spaces) to keep slug logic stable.

## Dependencies & External Services
- **Obsidian + community plugins:** Dataview, Meta Bind, Templater (core requirement) (`README.md:42-44`).
- **Moment.js:** provided by Obsidian; used for all date math (`Landing.md:8-11`, `Finance.md:198-216`, `Investments.md:38-40`).
- **Chart.js:** lazily loaded via CDN for finance and investment visualizations; requires network access unless cached (`Finance.md:205-273`, `Investments.md:41-55`).
- **Local filesystem:** Meta Bind actions create/modify Markdown files; ensure vault permissions allow writes.

## Testing & Quality
- No automated tests or CI scripts exist; verification is entirely manual.
- Recommended checks before committing:
  1. Open `Finance.md` and ensure the dashboard renders without console errors; the yearly chart should reflect the latest month.
  2. Toggle a task in `Todo.md` to confirm state persistence and XP increment (`Todo.md:233-266`).
  3. Add a dummy movement via `Investments.md` and ensure the line appears in the target note with the correct date tag.
  4. Refresh `Landing.md` to verify all three metrics resolve (investments, tasks, balance) without `—` placeholders.
- Because data folders are git-ignored, snapshot sensitive notes elsewhere before major experiments.

## Git History & Evolution
- Early commits established the repo scaffold and finance creation flow (“Feat: create repo”, “Feat: add new finance button”).
- Subsequent milestones introduced gitignore rules, CSS/visual polish, separate todo and investment modules, XP handling, and the current landing experience (`git log --oneline`).
- Commit messages consistently start with `Feat:` and are pushed directly to `main`, indicating a trunk-based, single-author workflow.
- There is no evidence of branching strategies or release tags; assume rolling deployment by syncing the vault.

## Hidden Context & Troubleshooting
- **Locale sensitivity:** All date parsing depends on `window.moment` using English month names—stick to English note names even if content is localized (`README.md:42-44`, `Landing.md:170-189`).
- **Chart.js CDN:** Offline mode prevents charts from rendering; consider packaging a local copy if long-term offline work is needed (`Investments.md:41-55`).
- **Task cleanup:** Removing an item from `todo/tasks.md` leaves stale IDs in the frontmatter until the dashboard’s cleanup runs—open `Todo.md` after editing the task list to trigger cleanup (`Todo.md:210-218`).
- **Finance button folder:** The “New finance month” action currently targets `finance/2025`; adjust `folderPath` when a new year starts or it will keep creating notes in the wrong year (`Finance.md:50-55`).
- **Security/privacy:** Sensitive financial data lives in plain text and is git-ignored; double-check remote backups and sharing permissions.
- **Avatar fallback:** If no `profile/pfp.*` exists, the landing card shows the name initial; ensure image formats match the candidate list before troubleshooting (`Landing.md:32-47`).

## Maintenance Checklist
- Update `folderPath` and template titles annually to point at the current year (`Finance.md:50-55`, `templates/new finance month.md:1`).
- Extend category lists in `config/consts.md` whenever new budget categories are introduced; keep names lowercase and hyphen-free for slug stability (`config/consts.md:1-15`).
- Refresh `templates/` when workflows evolve so Meta Bind buttons stay in sync.
- Periodically export or archive ignored data folders for offsite backups since git will not store them (`.gitignore:1-5`).
- Review `profile/stats.md` XP totals for realism if bulk task edits occurred (`profile/stats.md:1-2`).

---
