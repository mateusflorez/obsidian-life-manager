---
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
  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();

  const translations = {
    en: {
      openModule: "Open the Todo note to see this module.",
      loadError: ({ path }) => `Could not load ${path}.`,
      sectionTodo: "To-do",
      sectionDaily: "Daily",
      sectionWeekly: "Weekly",
      sectionMonthly: "Monthly",
      inputPlaceholder: ({ title }) => `Add to ${title}`,
      typeDescription: "Type a task description first.",
      saving: "Saving...",
      taskAdded: "Task added! Reload the note to see it here.",
      taskAddError: "Could not add the task.",
      noItems: "No items configured.",
      taskRemoved: "Task completed and removed.",
      taskRemoveError: "Task completed but could not be removed.",
      addButton: "Add",
    },
    pt: {
      openModule: "Abra a nota Todo para ver este módulo.",
      loadError: ({ path }) => `Não foi possível carregar ${path}.`,
      sectionTodo: "Tarefas",
      sectionDaily: "Diárias",
      sectionWeekly: "Semanais",
      sectionMonthly: "Mensais",
      inputPlaceholder: ({ title }) => `Adicionar em ${title}`,
      typeDescription: "Escreva a descrição da tarefa primeiro.",
      saving: "Salvando...",
      taskAdded: "Tarefa adicionada! Recarregue a nota para vê-la aqui.",
      taskAddError: "Não foi possível adicionar a tarefa.",
      noItems: "Nenhum item configurado.",
      taskRemoved: "Tarefa concluída e removida.",
      taskRemoveError: "Tarefa concluída, mas não pôde ser removida.",
      addButton: "Adicionar",
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  const file = app.workspace.getActiveFile();
  if (!file) {
    dv.paragraph(t("openModule"));
    return;
  }

  const tasksPath = "todo/tasks.md";
  let rawTasks = "";
  try {
    rawTasks = await dv.io.load(tasksPath);
  } catch (error) {
    dv.paragraph(t("loadError", { path: tasksPath }));
    return;
  }

  const today = window.moment();
  const todayStr = today.format("YYYY-MM-DD");
  const statsPath = "profile/stats.md";
  const XP_INCREMENT = 50;

  const sections = [
    { key: "todo", title: t("sectionTodo"), heading: "todo", type: "boolean" },
    { key: "daily", title: t("sectionDaily"), heading: "daily", type: "cycle", cycle: "day" },
    { key: "weekly", title: t("sectionWeekly"), heading: "weekly", type: "cycle", cycle: "week" },
    { key: "monthly", title: t("sectionMonthly"), heading: "monthly", type: "cycle", cycle: "month" },
  ];

  const statusFields = {
    todo: "todoStatus",
    daily: "dailyStatus",
    weekly: "weeklyStatus",
    monthly: "monthlyStatus",
  };

  const STATE_PATH = "todo/state.json";

  const createEmptyState = () => ({
    todoStatus: {},
    dailyStatus: {},
    weeklyStatus: {},
    monthlyStatus: {},
  });

  let stateFile = app.vault.getAbstractFileByPath(STATE_PATH) ?? null;

  const ensureStateFolder = async () => {
    const parts = STATE_PATH.split("/");
    parts.pop();
    if (!parts.length) return;
    const folderPath = parts.join("/");
    if (app.vault.getAbstractFileByPath(folderPath)) return;
    await app.vault.createFolder(folderPath);
  };

  const getStateFile = async () => {
    if (stateFile) {
      return stateFile;
    }
    const existing = app.vault.getAbstractFileByPath(STATE_PATH);
    if (existing) {
      stateFile = existing;
      return stateFile;
    }
    await ensureStateFolder();
    stateFile = await app.vault.create(STATE_PATH, JSON.stringify(createEmptyState(), null, 2));
    return stateFile;
  };

  const normalizeState = (value) => {
    const empty = createEmptyState();
    if (!value || typeof value !== "object") return empty;
    Object.keys(empty).forEach((key) => {
      const bucket = value[key];
      empty[key] = bucket && typeof bucket === "object" ? { ...bucket } : {};
    });
    return empty;
  };

  const loadState = async () => {
    const existing = app.vault.getAbstractFileByPath(STATE_PATH);
    if (existing) {
      stateFile = existing;
      try {
        const raw = await app.vault.read(existing);
        const parsed = JSON.parse(raw);
        return normalizeState(parsed);
      } catch (error) {
        console.warn("Todo: could not parse state file.", error);
      }
    }
    return createEmptyState();
  };

  let state = await loadState();

  const saveState = async () => {
    const fileRef = await getStateFile();
    await app.vault.modify(fileRef, JSON.stringify(state, null, 2));
  };

  const migrateFrontmatterState = async () => {
    const page = dv.page(file.path) ?? {};
    let mutated = false;
    Object.values(statusFields).forEach((field) => {
      const data = page[field];
      if (data && typeof data === "object" && Object.keys(data).length) {
        state[field] = { ...(state[field] ?? {}), ...data };
        mutated = true;
      }
    });
    if (!mutated) return;
    await saveState();
    await app.fileManager.processFrontMatter(file, (fm) => {
      Object.values(statusFields).forEach((field) => delete fm[field]);
    });
  };

  await migrateFrontmatterState();

  const makeId = (text) =>
    text
      .toLowerCase()
      .normalize("NFD")
      .replace(/[\u0300-\u036f]/g, "")
      .replace(/[^a-z0-9]+/g, "-")
      .replace(/^-+|-+$/g, "");

  const parseTaskContent = (rawText) => {
    let dateTag = null;
    let timeTag = null;
    let cleaned = rawText;

    cleaned = cleaned.replace(/#(\d{4}-\d{2}-\d{2})\b/g, (_, value) => {
      if (!dateTag) dateTag = value;
      return "";
    });

    cleaned = cleaned.replace(/#((?:[01]?\d|2[0-3]):[0-5]\d)\b/g, (_, value) => {
      if (!timeTag) timeTag = value;
      return "";
    });

    cleaned = cleaned.replace(/\s{2,}/g, " ").trim();

    return { text: cleaned, dateTag, timeTag };
  };

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
        const parsed = parseTaskContent(line.replace(/^-\s*/, "").trim());
        const baseText = parsed.text || "";
        return {
          text: baseText,
          id: makeId(baseText) || `${heading}-${idx + 1}`,
          dateTag: parsed.dateTag,
          timeTag: parsed.timeTag,
        };
      })
      .filter((item) => item.text.length > 0);
  };

  const statusCache = {
    todoStatus: { ...(state.todoStatus ?? {}) },
    dailyStatus: { ...(state.dailyStatus ?? {}) },
    weeklyStatus: { ...(state.weeklyStatus ?? {}) },
    monthlyStatus: { ...(state.monthlyStatus ?? {}) },
  };

  const persistStatus = async (field, id, value) => {
    if (!state[field] || typeof state[field] !== "object") state[field] = {};
    if (!statusCache[field]) statusCache[field] = {};

    if (value === null || value === undefined) {
      delete state[field][id];
      delete statusCache[field][id];
    } else {
      state[field][id] = value;
      statusCache[field][id] = value;
    }

    await saveState();
  };

  const cleanupStatus = async (field, validIds) => {
    if (!state[field] || typeof state[field] !== "object") state[field] = {};
    const statusMap = state[field];
    const toRemove = Object.keys(statusMap).filter((id) => !validIds.has(id));
    if (!toRemove.length) return;

    toRemove.forEach((id) => {
      delete statusMap[id];
      if (statusCache[field]) delete statusCache[field][id];
    });

    await saveState();
  };

  const insertTask = async (heading, text) => {
    const file = app.vault.getAbstractFileByPath(tasksPath);
    if (!file) throw new Error("tasks.md not found");
    const content = await app.vault.read(file);
    const sectionRegex = new RegExp(`(##\\s*${heading}[^\\n]*\\n)([\\s\\S]*?)(?=\\n##\\s+|$)`, "i");
    const match = sectionRegex.exec(content);
    const line = `- ${text}\n`;
    if (match) {
      const before = content.slice(0, match.index);
      const after = content.slice(match.index + match[0].length);
      const body = match[2].trimEnd();
      const newBody = `${body ? body + "\n" : ""}${line}`;
      const updated = before + match[1] + newBody + after;
      await app.vault.modify(file, updated);
    } else {
      const updated = `${content.trim()}\n\n## ${heading}\n${line}`;
      await app.vault.modify(file, updated);
    }
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

  const incrementCompletedTasks = async () => {
    const statsFile = app.vault.getAbstractFileByPath(statsPath);
    if (!statsFile) return;
    let content = await dv.io.load(statsPath);
    const tasksRegex = /^(\s*-\s*completed tasks::\s*)(\d+)/im;
    if (tasksRegex.test(content)) {
      content = content.replace(tasksRegex, (_, prefix, value) => {
        const current = Number(value) || 0;
        return `${prefix}${current + 1}`;
      });
    } else {
      const suffix = content.endsWith("\n") || content.length === 0 ? "" : "\n";
      content = `${content}${suffix}- completed tasks:: 1\n`;
    }
    await app.vault.modify(statsFile, content);
  };

  const createRow = (parent, labelText, metaParts = []) => {
    const row = parent.createEl("label", { cls: "todo-row" });
    row.style.display = "flex";
    row.style.alignItems = "center";
    row.style.gap = "0.5rem";
    row.style.marginBottom = "0.35rem";

    const checkbox = row.createEl("input", { type: "checkbox" });
    checkbox.style.margin = "0";

    const textWrapper = row.createEl("div");
    textWrapper.style.flex = "1";
    textWrapper.style.wordBreak = "break-word";
    textWrapper.style.display = "flex";
    textWrapper.style.flexDirection = "column";
    textWrapper.style.gap = "0.1rem";

    textWrapper.createEl("span", { text: labelText });

    if (metaParts.length) {
      const meta = textWrapper.createEl("span", { text: metaParts.join(" • ") });
      meta.style.fontSize = "0.8em";
      meta.style.color = "var(--text-muted)";
    }

    return { checkbox, row };
  };

  const removeTask = async (heading, id) => {
    const file = app.vault.getAbstractFileByPath(tasksPath);
    if (!file) return false;

    const content = await app.vault.read(file);
    const sectionRegex = new RegExp(`(##\\s*${heading}[^\\n]*\\n)([\\s\\S]*?)(?=\\n##\\s+|$)`, "i");
    const match = sectionRegex.exec(content);
    if (!match) return false;

    const hadTrailingNewline = match[2].endsWith("\n");
    const lines = match[2].split("\n");
    let removed = false;
    const newLines = [];

    for (const line of lines) {
      if (!removed) {
        const trimmed = line.trim();
        if (trimmed.startsWith("-")) {
          const parsed = parseTaskContent(trimmed.replace(/^-\s*/, "").trim());
          const candidateId = makeId(parsed.text || "") || "";
          if (candidateId === id) {
            removed = true;
            continue;
          }
        }
      }
      newLines.push(line);
    }

    if (!removed) return false;

    let newBody = newLines.join("\n");
    if (newBody && hadTrailingNewline && !newBody.endsWith("\n")) {
      newBody += "\n";
    }

    const updatedSection = match[1] + newBody;
    const updatedContent =
      content.slice(0, match.index) + updatedSection + content.slice(match.index + match[0].length);

    await app.vault.modify(file, updatedContent);
    return true;
  };

  for (const section of sections) {
    const items = extractItems(section.heading);
    const wrapper = container.createEl("div", { cls: "todo-section" });
    wrapper.createEl("h3", { text: section.title });

    const statusField = statusFields[section.key];
    const validIds = new Set(items.map((item) => item.id));
    await cleanupStatus(statusField, validIds);

    if (items.length === 0) {
      wrapper.createEl("p", { text: t("noItems") });
    }

    const form = wrapper.createEl("form");
    form.style.display = "flex";
    form.style.flexWrap = "wrap";
    form.style.gap = "0.35rem";
    form.style.marginBottom = "0.5rem";
    const input = form.createEl("input", {
      attr: { type: "text", placeholder: t("inputPlaceholder", { title: section.title }) },
    });
    input.style.flex = "1";
    const dateInput = form.createEl("input", {
      attr: { type: "date", "aria-label": "Date (optional)" },
    });
    dateInput.style.flex = "0 0 auto";
    dateInput.style.minWidth = "11rem";
    const timeInput = form.createEl("input", {
      attr: { type: "time", "aria-label": "Time (optional)" },
    });
    timeInput.style.flex = "0 0 auto";
    timeInput.style.minWidth = "7rem";
    const button = form.createEl("button", { text: t("addButton") });
    button.type = "submit";
    button.style.cursor = "pointer";

    form.onsubmit = async (event) => {
      event.preventDefault();
      const text = input.value.trim();
      if (!text) {
        new Notice(t("typeDescription"));
        return;
      }
      const dateValue = dateInput.value.trim();
      const timeValue = timeInput.value.trim();

      const parts = [text];
      if (dateValue) parts.push(`#${dateValue}`);
      if (timeValue) parts.push(`#${timeValue}`);
      const composedText = parts.join(" ");

      button.disabled = true;
      button.textContent = t("saving");
      try {
        await insertTask(section.heading, composedText);
        new Notice(t("taskAdded"));
        input.value = "";
        dateInput.value = "";
        timeInput.value = "";
      } catch (error) {
        console.error(error);
        new Notice(t("taskAddError"));
      } finally {
        button.disabled = false;
        button.textContent = t("addButton");
      }
    };

    if (items.length === 0) {
      continue;
    }

    const statusMap = statusCache[statusField] ?? {};

    items.forEach((item) => {
      const metaParts = [];
      if (item.dateTag) metaParts.push(`#${item.dateTag}`);
      if (item.timeTag) metaParts.push(`#${item.timeTag}`);
      const { checkbox, row } = createRow(wrapper, item.text, metaParts);

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
            await incrementCompletedTasks();
            if (section.key === "todo") {
              try {
                const removed = await removeTask(section.heading, item.id);
                if (removed) {
                  await persistStatus(statusField, item.id, null);
                  row.remove();
                  new Notice(t("taskRemoved"));
                }
              } catch (error) {
                console.error(error);
                new Notice(t("taskRemoveError"));
              }
            }
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
            await incrementCompletedTasks();
          }
        };
      }
    });
  }
};

run();
```
