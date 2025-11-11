# Achievements

> [!info]
> Track milestone progress for books, investments, tasks, and training sessions. Progress updates automatically from the source modules.

```dataviewjs
const run = async () => {
  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();
  const currency = (settingsPage.currency || "BRL").toUpperCase();

  const translations = {
    en: {
      title: "Achievements",
      chaptersTitle: "Chapters read",
      chaptersTargetLabel: ({ count }) => `${count} chapter(s) read`,
      investedTitle: ({ currency }) => `${currency} invested`,
      investedTargetLabel: ({ amount }) => `${amount} invested`,
      tasksTitle: "Tasks completed",
      tasksTargetLabel: ({ count }) => `${count} task(s) completed this cycle`,
      trainingTitle: "Training sessions",
      trainingTargetLabel: ({ count }) => `${count} session(s) logged`,
      levelTitle: "Level milestones",
      levelTargetLabel: ({ level }) => `Reach level ${level}`,
      overallTitle: "Achievement completion",
      overallLabel: ({ completed, total }) => `${completed}/${total} milestones complete`,
      completed: "Completed",
      progress: ({ current, target }) => `${current} / ${target}`,
      noTasksHint: "Keep completing tasks in Todo.md to reach these goals.",
    },
    pt: {
      title: "Conquistas",
      chaptersTitle: "Capítulos lidos",
      chaptersTargetLabel: ({ count }) => `${count} capítulo(s) lido(s)`,
      investedTitle: ({ currency }) => `${currency} investidos`,
      investedTargetLabel: ({ amount }) => `${amount} investidos`,
      tasksTitle: "Tarefas finalizadas",
      tasksTargetLabel: ({ count }) => `${count} tarefa(s) finalizada(s) neste ciclo`,
      trainingTitle: "Sessões de treino",
      trainingTargetLabel: ({ count }) => `${count} treino(s) registrado(s)`,
      levelTitle: "Marcos de nível",
      levelTargetLabel: ({ level }) => `Chegar ao nível ${level}`,
      overallTitle: "Conclusão das conquistas",
      overallLabel: ({ completed, total }) => `${completed}/${total} metas concluídas`,
      completed: "Concluído",
      progress: ({ current, target }) => `${current} / ${target}`,
      noTasksHint: "Finalize tarefas no Todo.md para avançar nessas metas.",
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  const palette = ["#4b5563", "#22c55e", "#3b82f6", "#a855f7", "#fb923c"];

  const extractFields = (line = "") => {
    const fields = {};
    const regex = /([a-zA-Z]+)::\s*([^#\n]+?)(?=\s+[a-zA-Z]+::|$)/g;
    let match;
    while ((match = regex.exec(line)) !== null) {
      fields[match[1].toLowerCase()] = match[2].trim();
    }
    return fields;
  };

  const parseChaptersFromContent = (content = "") => {
    const sectionRegex = /##\s*Chapters[^\n]*\n([\s\S]*?)(?=\n##\s+|$)/i;
    const match = sectionRegex.exec(content);
    const body = match ? match[1] : "";
    return body
      .split("\n")
      .map((line) => line.trim())
      .filter((line) => line.startsWith("-"))
      .map((line) => extractFields(line))
      .filter((fields) => Object.keys(fields).length > 0);
  };

  const loadTotalChaptersRead = async () => {
    const pages = dv
      .pages('"books"')
      .where((p) => p.file?.path?.startsWith("books/"))
      .array();
    if (pages.length === 0) return 0;

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
        total += 1;
        continue;
      }

      if (type === "book") {
        try {
          const content = await dv.io.load(page.file.path);
          total += parseChaptersFromContent(content).length;
        } catch (error) {
          console.warn("Could not read book note:", page.file.path, error);
        }
      }
    }
    return total;
  };

  const parseMovements = (content = "") => {
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
        return isNaN(amount) ? null : amount;
      })
      .filter((amount) => amount !== null);
  };

  const loadTotalInvested = async () => {
    const pages = dv
      .pages('"investments"')
      .where((p) => p.file?.path?.startsWith("investments/"))
      .array();
    if (pages.length === 0) return 0;

    let total = 0;
    for (const page of pages) {
      try {
        const content = await dv.io.load(page.file.path);
        parseMovements(content).forEach((amount) => {
          if (amount > 0) total += amount;
        });
      } catch (error) {
        console.warn("Could not read investment note:", page.file.path, error);
      }
    }
    return total;
  };

  const loadTrainingSessions = async () => {
    const folder = "training/exercises";
    const pages = dv
      .pages(`"${folder}"`)
      .where((p) => p.file?.path?.startsWith(`${folder}/`))
      .array();
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
        total += body
          .split("\n")
          .map((line) => line.trim())
          .filter((line) => line.startsWith("-")).length;
      } catch (error) {
        console.warn("Could not read exercise note:", page.file.path, error);
      }
    }
    return total;
  };

  const loadTasksCompleted = async () => {
    let rawTasks = "";
    try {
      rawTasks = await dv.io.load("todo/tasks.md");
    } catch {
      return 0;
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
          const id = text
            .toLowerCase()
            .normalize("NFD")
            .replace(/[\u0300-\u036f]/g, "")
            .replace(/[^a-z0-9]+/g, "-")
            .replace(/-+/g, "-")
            .replace(/^-+|-+$/g, "");
          return { text, id: id || `${heading}-${idx + 1}` };
        })
        .filter((item) => item.text.length > 0);
    };

    const todoPage = dv.page("Todo") ?? {};
    const today = window.moment();

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
      if (value instanceof Date) return window.moment(value);
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

    let completed = 0;
    sections.forEach((section) => {
      const items = extractItems(section.heading);
      const status = todoPage[section.statusField] ?? {};
      items.forEach((item) => {
        let done = false;
        if (section.type === "boolean") {
          done = Boolean(status[item.id]);
        } else {
          done = isCycleDone(section.cycle, status[item.id]);
        }
        if (done) completed += 1;
      });
    });

    return completed;
  };

  const formatCurrency = (value) => {
    const symbol = currency === "USD" ? "$" : "R$";
    const formatter = new Intl.NumberFormat(language === "pt" ? "pt-BR" : "en-US", {
      minimumFractionDigits: 0,
      maximumFractionDigits: 0,
    });
    return `${symbol} ${formatter.format(value)}`;
  };

  const loadXpLevel = async () => {
    try {
      const stats = await dv.io.load("profile/stats.md");
      const match = stats.match(/^\s*-\s*xp:\s*(\d+)/im);
      const xp = match ? Number(match[1]) || 0 : 0;
      return Math.floor(xp / 1000);
    } catch {
      return 0;
    }
  };

  const [
    chaptersRead,
    totalInvested,
    tasksCompleted,
    trainingSessions,
    currentLevel,
  ] = await Promise.all([
    loadTotalChaptersRead(),
    loadTotalInvested(),
    loadTasksCompleted(),
    loadTrainingSessions(),
    loadXpLevel(),
  ]);

  const achievements = [
    {
      key: "levels",
      title: t("levelTitle"),
      value: currentLevel,
      tiers: [10, 25, 50, 100, 500],
      label: (target) => t("levelTargetLabel", { level: target }),
    },
    {
      key: "chapters",
      title: t("chaptersTitle"),
      value: chaptersRead,
      tiers: [50, 100, 500, 1000, 5000],
      label: (target) => t("chaptersTargetLabel", { count: target }),
    },
    {
      key: "investments",
      title: t("investedTitle", { currency }),
      value: totalInvested,
      tiers: [1000, 5000, 10000, 50000, 100000],
      label: (target) => t("investedTargetLabel", { amount: formatCurrency(target) }),
    },
    {
      key: "tasks",
      title: t("tasksTitle"),
      value: tasksCompleted,
      tiers: [50, 100, 500, 1000, 5000],
      label: (target) => t("tasksTargetLabel", { count: target }),
    },
    {
      key: "training",
      title: t("trainingTitle"),
      value: trainingSessions,
      tiers: [50, 100, 500, 1000, 5000],
      label: (target) => t("trainingTargetLabel", { count: target }),
    },
  ];

  const module = dv.container.createEl("div", { cls: "achievements-module" });
  module.style.display = "flex";
  module.style.flexDirection = "column";
  module.style.gap = "1.5rem";

  module.createEl("h2", { text: t("title") });

  const totalTiers = achievements.reduce((sum, achievement) => sum + achievement.tiers.length, 0);
  const completedTiers = achievements.reduce(
    (sum, achievement) =>
      sum + achievement.tiers.filter((target) => achievement.value >= target).length,
    0
  );
  const completionPercent =
    totalTiers === 0 ? 0 : Math.min(100, Math.round((completedTiers / totalTiers) * 100));

  const summary = module.createEl("div");
  summary.style.display = "flex";
  summary.style.flexDirection = "column";
  summary.style.gap = "0.45rem";
  summary.style.padding = "1rem";
  summary.style.border = "1px solid var(--background-modifier-border)";
  summary.style.borderRadius = "10px";
  summary.style.background = "var(--background-secondary)";

  summary.createEl("strong", { text: t("overallTitle") });
  summary.createEl("span", {
    text: t("overallLabel", { completed: completedTiers, total: totalTiers }),
  }).style.fontSize = "0.9rem";

  const summaryTrack = summary.createEl("div");
  summaryTrack.style.width = "100%";
  summaryTrack.style.height = "8px";
  summaryTrack.style.borderRadius = "999px";
  summaryTrack.style.background = "var(--background-modifier-border)";

  const summaryFill = summaryTrack.createEl("div");
  summaryFill.style.height = "100%";
  summaryFill.style.borderRadius = "999px";
  summaryFill.style.background = "var(--interactive-accent)";
  summaryFill.style.width = `${completionPercent}%`;

  achievements.forEach((achievement) => {
    const section = module.createEl("section");
    section.style.display = "flex";
    section.style.flexDirection = "column";
    section.style.gap = "0.75rem";

    section.createEl("h3", { text: achievement.title }).style.marginBottom = "0";

    const cards = section.createEl("div");
    cards.style.display = "grid";
    cards.style.gridTemplateColumns = "repeat(auto-fit, minmax(220px, 1fr))";
    cards.style.gap = "0.75rem";

    achievement.tiers.forEach((target, index) => {
      const card = cards.createEl("div");
      card.style.display = "flex";
      card.style.flexDirection = "column";
      card.style.gap = "0.35rem";
      card.style.padding = "0.9rem";
      card.style.borderRadius = "10px";
      card.style.color = "#fff";
      card.style.background = palette[index] ?? palette[palette.length - 1];
      card.style.boxShadow = "0 2px 6px rgba(0,0,0,0.25)";

      const header = card.createEl("div");
      header.style.display = "flex";
      header.style.alignItems = "center";
      header.style.gap = "0.5rem";

      const title = header.createEl("strong", { text: achievement.label(target) });
      title.style.flex = "1";

      const isCompleted = achievement.value >= target;

      const trophy = header.createEl("span", { text: "🏆" });
      trophy.style.fontSize = "1.3rem";
      trophy.style.lineHeight = "1";
      trophy.style.color = isCompleted ? "#fff" : "#d1d5db";

      if (isCompleted) {
        const done = card.createEl("span", { text: t("completed") });
        done.style.fontWeight = "600";
      } else {
        const progressValue =
          achievement.key === "investments"
            ? `${formatCurrency(achievement.value)} / ${formatCurrency(target)}`
            : t("progress", { current: achievement.value, target });
        const progressLabel = card.createEl("span", { text: progressValue });
        progressLabel.style.fontSize = "0.85rem";

        const track = card.createEl("div");
        track.style.width = "100%";
        track.style.height = "6px";
        track.style.borderRadius = "999px";
        track.style.background = "rgba(255,255,255,0.35)";

        const fill = track.createEl("div");
        fill.style.height = "100%";
        fill.style.borderRadius = "999px";
        fill.style.background = "#fff";
        fill.style.width = `${Math.min(100, Math.round((achievement.value / target) * 100))}%`;
      }
    });

    if (achievement.key === "tasks" && tasksCompleted === 0) {
      const hint = section.createEl("p", { text: t("noTasksHint") });
      hint.style.margin = "0";
      hint.style.fontSize = "0.85rem";
      hint.style.color = "var(--text-muted)";
    }
  });
};

run();
```
