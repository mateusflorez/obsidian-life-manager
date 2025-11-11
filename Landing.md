
# Landing Hub

## 👤 Overview

```dataviewjs
const run = async () => {
  const today = window.moment();
  const monthLabel = today.format("MMM YYYY");
  const currentYear = today.format("YYYY");
  const currentMonthName = today.format("MMMM");

  const container = dv.container.createEl("div", { cls: "landing-overview" });
  container.style.display = "flex";
  container.style.flexDirection = "column";
  container.style.gap = "1rem";
  container.style.marginBottom = "1.5rem";
  container.style.padding = "1rem";
  container.style.border = "1px solid var(--background-modifier-border)";
  container.style.borderRadius = "12px";

  const profileName = await (async () => {
    try {
      const statsContent = await dv.io.load("profile/stats.md");
      const match = statsContent.match(/^\s*-\s*name:\s*(.+)$/im);
      return match ? match[1].trim() : "Welcome";
    } catch {
      return "Welcome";
    }
  })();

  const avatarSrc = (() => {
    const candidates = [
      "profile/pfp.png",
      "profile/pfp.jpg",
      "profile/pfp.jpeg",
      "profile/pfp.webp",
      "profile/pfp.gif",
    ];
    for (const path of candidates) {
      const file = app.vault.getAbstractFileByPath(path);
      if (file && typeof file === "object" && "extension" in file) {
        return app.vault.getResourcePath(file);
      }
    }
    return null;
  })();

  const formatCurrency = (value) => {
    if (value === null || value === undefined || isNaN(value)) return "—";
    return `R$ ${Number(value).toFixed(2).replace(".", ",")}`;
  };

  const makeId = (text) =>
    text
      .toLowerCase()
      .normalize("NFD")
      .replace(/[\u0300-\u036f]/g, "")
      .replace(/[^a-z0-9]+/g, "-")
      .replace(/-+/g, "-")
      .replace(/^-+|-+$/g, "");

  const getTasksSummary = async () => {
    let rawTasks = "";
    try {
      rawTasks = await dv.io.load("todo/tasks.md");
    } catch {
      return { unfinished: 0, total: 0 };
    }

    const sections = [
      { heading: "todo", type: "boolean", statusField: "todoStatus" },
      { heading: "daily", type: "cycle", statusField: "dailyStatus", cycle: "day" },
      { heading: "weekly", type: "cycle", statusField: "weeklyStatus", cycle: "week" },
      { heading: "monthly", type: "cycle", statusField: "monthlyStatus", cycle: "month" },
    ];

    const sectionLines = {};
    let currentHeading = null;
    rawTasks.split("\n").forEach((rawLine) => {
      const line = rawLine.replace(/\r$/, "");
      const headingMatch = line.match(/^##\s+(.+)/i);
      if (headingMatch) {
        const normalized = headingMatch[1].trim().toLowerCase();
        currentHeading =
          sections.find((s) => normalized === s.heading)?.heading ||
          sections.find((s) => normalized.startsWith(s.heading))?.heading ||
          sections.find((s) => s.heading.startsWith(normalized))?.heading ||
          normalized;
        if (!sectionLines[currentHeading]) sectionLines[currentHeading] = [];
        return;
      }
      if (!currentHeading) return;
      sectionLines[currentHeading].push(line.trim());
    });

    const extractItems = (heading) => {
      const lines = sectionLines[heading] ?? [];
      return lines
        .filter((line) => line.startsWith("-"))
        .map((line, idx) => {
          const text = line.replace(/^-\s*/, "").trim();
          return { text, id: makeId(text) || `${heading}-${idx + 1}` };
        })
        .filter((item) => item.text.length > 0);
    };

    const todoPage = dv.page("Todo") ?? {};

    const parseStoredDate = (value) => {
      if (!value) return null;
      if (window.moment.isMoment(value)) return value;
      if (typeof value === "string") {
        const parsed = window.moment(value, "YYYY-MM-DD", true);
        return parsed.isValid() ? parsed : null;
      }
      if (value?.isLuxonDateTime && typeof value.toISODate === "function") {
        return window.moment(value.toISODate(), "YYYY-MM-DD", true);
      }
      if (value instanceof Date) {
        return window.moment(value);
      }
      return null;
    };

    const isCycleDone = (cycle, storedValue) => {
      const date = parseStoredDate(storedValue);
      if (!date || !date.isValid()) return false;
      if (cycle === "day") return date.isSame(today, "day");
      if (cycle === "week")
        return date.isoWeek() === today.isoWeek() && date.isoWeekYear() === today.isoWeekYear();
      if (cycle === "month") return date.isSame(today, "month");
      return false;
    };

    let total = 0;
    let unfinished = 0;

    sections.forEach((section) => {
      const items = extractItems(section.heading);
      const status = todoPage[section.statusField] ?? {};
      items.forEach((item) => {
        total += 1;
        let done = false;
        if (section.type === "boolean") {
          done = Boolean(status[item.id]);
        } else {
          done = isCycleDone(section.cycle, status[item.id]);
        }
        if (!done) unfinished += 1;
      });
    });

    return { unfinished, total };
  };

  const parseMovements = (content) => {
    const sectionMatch = content.match(/##\s*Movements[^\n]*\n([\s\S]*?)(?=\n##\s+|$)/i);
    const block = sectionMatch ? sectionMatch[1] : content;
    return block
      .split("\n")
      .map((line) => line.trim())
      .filter((line) => line.startsWith("-"))
      .map((line) => {
        const amountMatch = line.replace(/^-\s*/, "").match(/^(-?[\d.,]+)/);
        if (!amountMatch) return null;
        const amount = parseFloat(amountMatch[0].replace(",", "."));
        if (isNaN(amount)) return null;
        const tags = [...line.matchAll(/#([^\s#]+)/g)].map((m) => m[1]);
        const dateTag = tags.find((tag) => /^\d{4}-\d{2}-\d{2}$/.test(tag));
        const date = dateTag ? window.moment(dateTag, "YYYY-MM-DD", true) : null;
        return { amount, date: date && date.isValid() ? date : null };
      })
      .filter(Boolean);
  };

  const getMonthlyInvestments = async () => {
    const pages = dv
      .pages('"investments"')
      .where((p) => p.file.path.startsWith("investments/"));
    if (pages.length === 0) return 0;
    let total = 0;
    for (const investment of pages) {
      const content = await dv.io.load(investment.file.path);
      parseMovements(content).forEach((move) => {
        if (!move.date) return;
        if (
          move.date.isSame(today, "month") &&
          move.date.isSame(today, "year")
        ) {
          total += move.amount;
        }
      });
    }
    return total;
  };

  const getFinanceBalance = async () => {
    const noteName =
      currentMonthName.charAt(0).toUpperCase() + currentMonthName.slice(1);
    const path = `finance/${currentYear}/${noteName}.md`;
    const file = app.vault.getAbstractFileByPath(path);
    if (!file) return null;
    const content = await dv.io.load(path);

    const sumSection = (heading) => {
      const section = content.split(new RegExp(`##\\s*${heading}`, "i"))[1];
      if (!section) return 0;
      const body = section.split(/\n##\s+/)[0];
      return body.split("\n").reduce((sum, rawLine) => {
        const line = rawLine.trim();
        if (!line) return sum;
        const match = line.match(/^-?\s*([^:]+)::\s*([\d.,]+)/);
        if (!match) return sum;
        const value = parseFloat(match[2].replace(",", "."));
        return isNaN(value) ? sum : sum + value;
      }, 0);
    };

    const expenses = sumSection("Expenses");
    const income = sumSection("Income");
    return income - expenses;
  };

  const getXp = async () => {
    try {
      const stats = await dv.io.load("profile/stats.md");
      const match = stats.match(/^\s*-\s*xp:\s*(\d+)/im);
      return match ? Number(match[1]) || 0 : 0;
    } catch {
      return 0;
    }
  };

  const [tasks, investedThisMonth, balance, xp] = await Promise.all([
    getTasksSummary(),
    getMonthlyInvestments(),
    getFinanceBalance(),
    getXp(),
  ]);

  const level = Math.floor(xp / 1000);
  const currentProgress = xp % 1000;
  const progressPercent = Math.min(100, (currentProgress / 1000) * 100);

  const header = container.createEl("div");
  header.style.display = "flex";
  header.style.alignItems = "center";
  header.style.gap = "1rem";

  const avatarWrapper = header.createEl("div");
  avatarWrapper.style.width = "72px";
  avatarWrapper.style.height = "72px";
  avatarWrapper.style.borderRadius = "50%";
  avatarWrapper.style.overflow = "hidden";
  avatarWrapper.style.background = "var(--background-secondary)";
  avatarWrapper.style.display = "flex";
  avatarWrapper.style.alignItems = "center";
  avatarWrapper.style.justifyContent = "center";
  avatarWrapper.style.fontSize = "1.5rem";
  avatarWrapper.style.fontWeight = "600";

  if (avatarSrc) {
    avatarWrapper.createEl("img", {
      attr: { src: avatarSrc, alt: profileName },
    }).style.width = "100%";
  } else {
    avatarWrapper.textContent = profileName.charAt(0) || "?";
  }

  const headerInfo = header.createEl("div");
  headerInfo.createEl("p", { text: "Welcome back," }).style.margin = "0";
  const nameEl = headerInfo.createEl("h2", { text: profileName });
  nameEl.style.margin = "0";
  nameEl.style.fontSize = "1.6rem";

  const levelCard = container.createEl("div");
  levelCard.style.display = "flex";
  levelCard.style.flexDirection = "column";
  levelCard.style.gap = "0.35rem";
  levelCard.style.border = "1px solid var(--background-modifier-border)";
  levelCard.style.borderRadius = "8px";
  levelCard.style.padding = "0.75rem";

  const levelLabel = levelCard.createEl("div");
  levelLabel.style.display = "flex";
  levelLabel.style.justifyContent = "space-between";
  levelLabel.style.fontWeight = "600";
  levelLabel.textContent = `Level ${level}`;
  levelCard.createEl("span", {
    text: `${currentProgress}/1000 XP`,
  }).style.fontSize = "0.85rem";

  const progressTrack = levelCard.createEl("div");
  progressTrack.style.width = "100%";
  progressTrack.style.height = "10px";
  progressTrack.style.borderRadius = "999px";
  progressTrack.style.background = "var(--background-modifier-border)";

  const progressFill = progressTrack.createEl("div");
  progressFill.style.height = "100%";
  progressFill.style.borderRadius = "999px";
  progressFill.style.background = "var(--interactive-accent)";
  progressFill.style.width = `${progressPercent}%`;

  const metricsWrapper = container.createEl("div");
  metricsWrapper.style.display = "grid";
  metricsWrapper.style.gridTemplateColumns = "repeat(auto-fit, minmax(180px, 1fr))";
  metricsWrapper.style.gap = "0.75rem";

  [
    {
      label: "Invested this month",
      value: formatCurrency(investedThisMonth),
      hint: monthLabel,
    },
    {
      label: "Open tasks",
      value: `${tasks.unfinished}`,
      hint: `${tasks.total} total`,
    },
    {
      label: "Financial balance",
      value: formatCurrency(balance),
      hint: `${currentMonthName} ${currentYear}`,
    },
  ].forEach((metric) => {
    const card = metricsWrapper.createEl("div");
    card.style.padding = "0.75rem";
    card.style.borderRadius = "8px";
    card.style.border = "1px solid var(--background-modifier-border)";
    card.style.display = "flex";
    card.style.flexDirection = "column";
    card.style.gap = "0.25rem";

    const label = card.createEl("span", { text: metric.label });
    label.style.fontSize = "0.85rem";
    label.style.color = "var(--text-muted)";

    const value = card.createEl("strong", { text: metric.value });
    value.style.fontSize = "1.2rem";

    const hint = card.createEl("span", { text: metric.hint });
    hint.style.fontSize = "0.85rem";
    hint.style.color = "var(--text-normal)";
  });
};

run();
```

```meta-bind-button
label: Finance
icon: briefcase
style: primary
class: ""
cssStyle: ""
backgroundImage: ""
tooltip: ""
id: ""
hidden: false
actions:
  - type: open
    link: "[[Finance]]"
    newTab: false

```

```meta-bind-button
label: To-do
icon: check
style: primary
class: ""
cssStyle: ""
backgroundImage: ""
tooltip: ""
id: ""
hidden: false
actions:
  - type: open
    link: "[[Todo]]"
    newTab: false

```

```meta-bind-button
label: Investments
icon: chart-line
style: primary
class: ""
cssStyle: ""
backgroundImage: ""
tooltip: ""
id: ""
hidden: false
actions:
  - type: open
    link: "[[Investments]]"
    newTab: false

```
