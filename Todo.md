---
todoStatus:
  wash-towel-and-bedsheets: false
dailyStatus: {}
weeklyStatus: {}
monthlyStatus: {}
---

# To-do

> [!info]
> Keep routines aligned: edit `todo/tasks.md`, toggle progress here, and earn 50 XP every time you complete a task.

```meta-bind-button
label: Open tasks list
icon: list-check
style: primary
class: ""
cssStyle: ""
backgroundImage: ""
tooltip: ""
id: ""
hidden: false
actions:
  - type: open
    link: "[[todo/tasks]]"
    newTab: false

```

## Task board
```dataviewjs
const run = async () => {
  const file = app.workspace.getActiveFile();
  if (!file) {
    dv.paragraph("Open the Todo note to see this module.");
    return;
  }

  const tasksPath = "todo/tasks.md";
  let rawTasks = "";
  try {
    rawTasks = await dv.io.load(tasksPath);
  } catch (error) {
    dv.paragraph(`Could not load ${tasksPath}.`);
    return;
  }

  const today = window.moment();
  const todayStr = today.format("YYYY-MM-DD");
  const statsPath = "profile/stats.md";
  const XP_INCREMENT = 50;

  const sections = [
    { key: "todo", title: "Todo", heading: "todo", type: "boolean" },
    { key: "daily", title: "Daily", heading: "daily", type: "cycle", cycle: "day" },
    { key: "weekly", title: "Weekly", heading: "weekly", type: "cycle", cycle: "week" },
    { key: "monthly", title: "Monthly", heading: "monthly", type: "cycle", cycle: "month" },
  ];

  const statusFields = {
    todo: "todoStatus",
    daily: "dailyStatus",
    weekly: "weeklyStatus",
    monthly: "monthlyStatus",
  };

  const makeId = (text) =>
    text
      .toLowerCase()
      .normalize("NFD")
      .replace(/[\u0300-\u036f]/g, "")
      .replace(/[^a-z0-9]+/g, "-")
      .replace(/^-+|-+$/g, "");

  const canonicalHeadings = sections.map((s) => s.heading);
  const sectionLines = {};
  let currentKey = null;

  rawTasks.split("\n").forEach((rawLine) => {
    const line = rawLine.replace(/\r$/, "");
    const headingMatch = line.match(/^##\s+(.+)/i);
    if (headingMatch) {
      const normalized = headingMatch[1].trim().toLowerCase();
      currentKey =
        canonicalHeadings.find((key) => normalized === key) ||
        canonicalHeadings.find((key) => normalized.startsWith(key)) ||
        canonicalHeadings.find((key) => key.startsWith(normalized)) ||
        normalized;
      if (!sectionLines[currentKey]) sectionLines[currentKey] = [];
      return;
    }

    if (!currentKey) return;
    sectionLines[currentKey].push(line.trim());
  });

  const extractItems = (heading) => {
    const lines = sectionLines[heading.toLowerCase()] ?? [];
    return lines
      .filter((line) => line.startsWith("-"))
      .map((line, idx) => {
        const text = line.replace(/^-\s*/, "").trim();
        return { text, id: makeId(text) || `${heading}-${idx + 1}` };
      })
      .filter((item) => item.text.length > 0);
  };

  const page = dv.page(file.path) ?? {};
  const statusCache = {};
  Object.values(statusFields).forEach((field) => {
    const raw = page[field];
    statusCache[field] = raw && typeof raw === "object" ? { ...raw } : {};
  });

  const persistStatus = async (field, id, value) => {
    await app.fileManager.processFrontMatter(file, (fm) => {
      if (!fm[field] || typeof fm[field] !== "object") fm[field] = {};
      if (value === null || value === undefined) {
        delete fm[field][id];
      } else {
        fm[field][id] = value;
      }
    });

    if (!statusCache[field]) statusCache[field] = {};
    if (value === null || value === undefined) {
      delete statusCache[field][id];
    } else {
      statusCache[field][id] = value;
    }
  };

  const cleanupStatus = async (field, validIds) => {
    const statusMap = statusCache[field] ?? {};
    const toRemove = Object.keys(statusMap).filter((id) => !validIds.has(id));
    if (toRemove.length === 0) return;

    await app.fileManager.processFrontMatter(file, (fm) => {
      if (!fm[field] || typeof fm[field] !== "object") return;
      toRemove.forEach((id) => delete fm[field][id]);
    });

    toRemove.forEach((id) => delete statusMap[id]);
  };

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

  const container = dv.container.createEl("div", { cls: "todo-module" });

  const incrementXp = async () => {
    const statsFile = app.vault.getAbstractFileByPath(statsPath);
    if (!statsFile) return;
    let content = await dv.io.load(statsPath);
    const xpRegex = /^(\s*-\s*xp:\s*)(\d+)/im;
    if (xpRegex.test(content)) {
      content = content.replace(xpRegex, (_, prefix, value) => {
        const current = Number(value) || 0;
        return `${prefix}${current + XP_INCREMENT}`;
      });
    } else {
      const suffix = content.endsWith("\n") || content.length === 0 ? "" : "\n";
      content = `${content}${suffix}- xp: ${XP_INCREMENT}\n`;
    }
    await app.vault.modify(statsFile, content);
  };

  const createRow = (parent, labelText) => {
    const row = parent.createEl("label", { cls: "todo-row" });
    row.style.display = "flex";
    row.style.alignItems = "center";
    row.style.gap = "0.5rem";
    row.style.marginBottom = "0.35rem";

    const checkbox = row.createEl("input", { type: "checkbox" });
    checkbox.style.margin = "0";

    const textEl = row.createEl("span", { text: labelText });
    textEl.style.flex = "1";
    textEl.style.wordBreak = "break-word";

    return checkbox;
  };

  for (const section of sections) {
    const items = extractItems(section.heading);
    const wrapper = container.createEl("div", { cls: "todo-section" });
    wrapper.createEl("h3", { text: section.title });

    const statusField = statusFields[section.key];
    const validIds = new Set(items.map((item) => item.id));
    await cleanupStatus(statusField, validIds);

    if (items.length === 0) {
      wrapper.createEl("p", { text: "No items configured." });
      continue;
    }

    const statusMap = statusCache[statusField] ?? {};

    items.forEach((item) => {
      const checkbox = createRow(wrapper, item.text);

      if (section.type === "boolean") {
        let doneState = Boolean(statusMap[item.id]);
        checkbox.checked = doneState;

        checkbox.onchange = async () => {
          checkbox.disabled = true;
          const checked = checkbox.checked;
          const isCompleting = checked && !doneState;

          await persistStatus(statusField, item.id, checked);

          checkbox.checked = checked;
          checkbox.disabled = false;
          doneState = checked;

          if (isCompleting) {
            await incrementXp();
          }
        };
      } else {
        let doneState = isCycleDone(section.cycle, statusMap[item.id]);
        checkbox.checked = doneState;

        checkbox.onchange = async () => {
          checkbox.disabled = true;
          const checked = checkbox.checked;
          const isCompleting = checked && !doneState;

          await persistStatus(statusField, item.id, checked ? todayStr : null);

          checkbox.checked = checked;
          checkbox.disabled = false;
          doneState = checked;

          if (isCompleting) {
            await incrementXp();
          }
        };
      }
    });
  }
};

run();
```
