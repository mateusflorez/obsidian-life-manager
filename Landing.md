
# Landing Hub

## 👤 Overview

```dataviewjs
const run = async () => {
  const today = window.moment();
  const monthLabel = today.format("MMM YYYY");
  const currentYear = today.format("YYYY");
  const currentMonthName = today.format("MMMM");

  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();
  const currency = (settingsPage.currency || "BRL").toUpperCase();

  const monthTranslations = {
    January: { short: "jan", full: "janeiro" },
    February: { short: "fev", full: "fevereiro" },
    March: { short: "mar", full: "março" },
    April: { short: "abr", full: "abril" },
    May: { short: "mai", full: "maio" },
    June: { short: "jun", full: "junho" },
    July: { short: "jul", full: "julho" },
    August: { short: "ago", full: "agosto" },
    September: { short: "set", full: "setembro" },
    October: { short: "out", full: "outubro" },
    November: { short: "nov", full: "novembro" },
    December: { short: "dez", full: "dezembro" },
  };

  const translations = {
    en: {
      welcomeBack: "Welcome back,",
      level: "Level",
      xpSuffix: "XP",
      investedLabel: "Invested this month",
      openTasksLabel: "Open tasks",
      tasksHint: ({ total }) => `${total} total`,
      trainingLabel: "Training sessions",
      chaptersLabel: "Chapters read",
      balanceLabel: "Financial balance",
      monthHint: ({ shortLabel }) => shortLabel,
      monthFullHint: ({ fullLabel }) => fullLabel,
    },
    pt: {
      welcomeBack: "Bem-vindo de volta,",
      level: "Nível",
      xpSuffix: "XP",
      investedLabel: "Investido no mês",
      openTasksLabel: "Tarefas em aberto",
      tasksHint: ({ total }) => `${total} no total`,
      trainingLabel: "Treinos no mês",
      chaptersLabel: "Capítulos lidos",
      balanceLabel: "Balanço financeiro",
      monthHint: ({ shortLabel }) => shortLabel,
      monthFullHint: ({ fullLabel }) => fullLabel,
    },
  };

  const translate = (key, data = {}) => {
    const entry =
      translations[language]?.[key] ??
      translations.en[key] ??
      key;
    return typeof entry === "function" ? entry(data) : entry;
  };

  const localizedMonthShort =
    language === "pt"
      ? `${monthTranslations[currentMonthName]?.short ?? today.format("MMM").toLowerCase()} ${currentYear}`
      : monthLabel;

  const localizedMonthFull =
    language === "pt"
      ? `${monthTranslations[currentMonthName]?.full ?? currentMonthName.toLowerCase()} ${currentYear}`
      : `${currentMonthName} ${currentYear}`;

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

  const currencySymbols = {
    BRL: { prefix: "R$", format: (value) => Number(value).toFixed(2).replace(".", ",") },
    USD: { prefix: "$", format: (value) => Number(value).toFixed(2) },
  };
  const currencyConfig = currencySymbols[currency] || currencySymbols.BRL;
  const formatCurrency = (value) => {
    if (value === null || value === undefined || isNaN(value)) return "—";
    return `${currencyConfig.prefix} ${currencyConfig.format(value)}`;
  };

  const extractFields = (line) => {
    const fields = {};
    const regex = /([a-z]+)::\s*([^#\n]+?)(?=\s+[a-z]+::|$)/gi;
    let match;
    while ((match = regex.exec(line)) !== null) {
      fields[match[1].toLowerCase()] = match[2].trim();
    }
    return fields;
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

  const getMonthlyTrainingSessions = async () => {
    const folder = "training/exercises";
    const pages = dv
      .pages(`"${folder}"`)
      .where((p) => p.file.path.startsWith(`${folder}/`));
    if (pages.length === 0) return 0;

    const sessionsHeading = "Sessions";
    const sectionRegex = new RegExp(
      `##\\s*${sessionsHeading}[^\\n]*\\n([\\s\\S]*?)(?=\\n##\\s+|$)`,
      "i"
    );

    let total = 0;

    for (const page of pages) {
      try {
        const content = await dv.io.load(page.file.path);
        const match = sectionRegex.exec(content);
        const body = match ? match[1] : "";
        body
          .split("\n")
          .map((line) => line.trim())
          .filter((line) => line.startsWith("-"))
          .forEach((line) => {
            const fields = extractFields(line);
            const dateValue = fields.date ?? null;
            if (!dateValue) return;
            const date = window.moment(dateValue, "YYYY-MM-DD", true);
            if (!date.isValid()) return;
            if (date.isSame(today, "month") && date.isSame(today, "year")) {
              total += 1;
            }
          });
      } catch (error) {
        console.warn("Could not read exercise note:", page.file.path, error);
      }
    }

    return total;
  };

  const getMonthlyChaptersRead = async () => {
    const pages = dv
      .pages('"books"')
      .where((p) => p.file?.path?.startsWith("books/"));
    if (pages.length === 0) return 0;

    const chaptersRegex = /##\s*Chapters[^\n]*\n([\s\S]*?)(?=\n##\s+|$)/i;
    let total = 0;

    for (const page of pages) {
      const type = (
        page.booktype ??
        page.bookType ??
        page.file?.frontmatter?.bookType ??
        ""
      )
        .toString()
        .toLowerCase();

      if (type === "entry") {
        const finishedValue =
          page.finishedat ??
          page.finishedAt ??
          page.file?.frontmatter?.finishedAt ??
          null;
        if (!finishedValue) continue;
        const finished = window.moment(finishedValue, window.moment.ISO_8601, true);
        if (finished.isValid() && finished.isSame(today, "month") && finished.isSame(today, "year")) {
          total += 1;
        }
        continue;
      }

      if (type === "book") {
        try {
          const content = await dv.io.load(page.file.path);
          const match = chaptersRegex.exec(content);
          const body = match ? match[1] : "";
          body
            .split("\n")
            .map((line) => line.trim())
            .filter((line) => line.startsWith("-"))
            .forEach((line) => {
              const fields = extractFields(line);
              const finished = fields.finished ?? fields.date ?? null;
              if (!finished) return;
              const finishedDate = window.moment(finished, window.moment.ISO_8601, true);
              if (
                finishedDate.isValid() &&
                finishedDate.isSame(today, "month") &&
                finishedDate.isSame(today, "year")
              ) {
                total += 1;
              }
            });
        } catch (error) {
          console.warn("Could not read book note:", page.file.path, error);
        }
      }
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

  const [
    tasks,
    investedThisMonth,
    balance,
    xp,
    trainingThisMonth,
    chaptersThisMonth,
  ] = await Promise.all([
    getTasksSummary(),
    getMonthlyInvestments(),
    getFinanceBalance(),
    getXp(),
    getMonthlyTrainingSessions(),
    getMonthlyChaptersRead(),
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
  headerInfo.createEl("p", { text: translate("welcomeBack") }).style.margin = "0";
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
  levelLabel.textContent = `${translate("level")} ${level}`;
  levelCard.createEl("span", {
    text: `${currentProgress}/1000 ${translate("xpSuffix")}`,
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
      label: translate("investedLabel"),
      value: formatCurrency(investedThisMonth),
      hint: translate("monthHint", { shortLabel: localizedMonthShort }),
    },
    {
      label: translate("openTasksLabel"),
      value: `${tasks.unfinished}`,
      hint: translate("tasksHint", { total: tasks.total }),
    },
    {
      label: translate("trainingLabel"),
      value: `${trainingThisMonth}`,
      hint: translate("monthFullHint", { fullLabel: localizedMonthFull }),
    },
    {
      label: translate("chaptersLabel"),
      value: `${chaptersThisMonth}`,
      hint: translate("monthFullHint", { fullLabel: localizedMonthFull }),
    },
    {
      label: translate("balanceLabel"),
      value: formatCurrency(balance),
      hint: translate("monthFullHint", { fullLabel: localizedMonthFull }),
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

```dataviewjs
const run = async () => {
  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();

  const translations = {
    en: {
      finance: "Finance",
      todo: "To-do",
      investments: "Investments",
      training: "Training",
      books: "Books",
      config: "Config",
    },
    pt: {
      finance: "Finanças",
      todo: "Tarefas",
      investments: "Investimentos",
      training: "Treinos",
      books: "Livros",
      config: "Configurações",
    },
  };

  const t = (key) =>
    translations[language]?.[key] ??
    translations.en[key] ??
    key;

  const container = dv.container.createEl("div", { cls: "landing-buttons" });
  container.style.display = "flex";
  container.style.flexWrap = "wrap";
  container.style.gap = "0.75rem";

  const buttons = [
    { key: "finance", icon: "💼", link: "Finance" },
    { key: "todo", icon: "✅", link: "Todo" },
    { key: "investments", icon: "📈", link: "Investments" },
    { key: "training", icon: "🏋️", link: "Training" },
    { key: "books", icon: "📚", link: "Books" },
    { key: "config", icon: "⚙️", link: "Config" },
  ];

  buttons.forEach((btn) => {
    const button = container.createEl("button");
    button.style.flex = "1 1 160px";
    button.style.display = "flex";
    button.style.alignItems = "center";
    button.style.justifyContent = "center";
    button.style.gap = "0.35rem";
    button.style.padding = "0.65rem";
    button.style.border = "1px solid var(--background-modifier-border)";
    button.style.borderRadius = "8px";
    button.style.background = "var(--interactive-accent)";
    button.style.color = "var(--text-on-accent)";
    button.style.cursor = "pointer";
    button.style.fontWeight = "600";
    button.style.fontSize = "1rem";

    const icon = button.createEl("span", { text: btn.icon });
    icon.ariaHidden = "true";

    button.createEl("span", { text: t(btn.key) });

    button.onclick = () => {
      app.workspace.openLinkText(btn.link, "", false);
    };
  });
};

run();
```
