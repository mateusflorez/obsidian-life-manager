# Training

> [!info]
> Register exercises, log daily workouts, and visualize activity over the last 60 days. Each exercise note stores its own `## Sessions` list.

## Training hub
```dataviewjs
const run = async () => {
  const exercisesFolder = "training/exercises";
  const templatePath = "templates/new training exercise.md";
  const sessionsHeading = "Sessions";
  const statsPath = "profile/stats.md";
  const TRAINING_XP = 10;
  const today = window.moment();
  const todayStr = today.format("YYYY-MM-DD");

  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();

  const translations = {
    en: {
      exercisesTitle: "Exercises",
      noExercises: "No exercises yet.",
      newExercisePlaceholder: "New exercise name",
      createExercise: "Create exercise",
      typeNameFirst: "Type a name first.",
      saving: "Saving...",
      exerciseCreated: "Exercise created.",
      couldNotCreateExercise: "Could not create exercise.",
      exerciseExists: "Exercise already exists.",
      logSession: "Log a session",
      loadPlaceholder: "Load",
      repsPlaceholder: "Reps",
      notesPlaceholder: "Notes (optional)",
      addSession: "Add session",
      createExerciseFirst: "Create an exercise first.",
      chooseExercise: "Choose an exercise.",
      invalidLoad: "Provide a valid load.",
      invalidReps: "Provide valid reps.",
      sessionLogged: "Session logged.",
      couldNotSaveSession: "Could not save the session.",
      sessionsCard: "Sessions",
      totalVolumeCard: "Total volume",
      noSessionsLogged: "No sessions logged yet.",
      tableDate: "Date",
      tableExercise: "Exercise",
      tableLoad: "Load",
      tableReps: "Reps",
      tableVolume: "Volume",
      last60Days: "Last 60 days",
      noTraining: "No training",
      sessionCount: ({ count }) => `${count} session(s)`,
    },
    pt: {
      exercisesTitle: "Exercícios",
      noExercises: "Nenhum exercício ainda.",
      newExercisePlaceholder: "Nome do novo exercício",
      createExercise: "Criar exercício",
      typeNameFirst: "Digite um nome primeiro.",
      saving: "Salvando...",
      exerciseCreated: "Exercício criado.",
      couldNotCreateExercise: "Não foi possível criar o exercício.",
      exerciseExists: "O exercício já existe.",
      logSession: "Registrar treino",
      loadPlaceholder: "Carga",
      repsPlaceholder: "Repetições",
      notesPlaceholder: "Notas (opcional)",
      addSession: "Adicionar treino",
      createExerciseFirst: "Crie um exercício primeiro.",
      chooseExercise: "Escolha um exercício.",
      invalidLoad: "Informe uma carga válida.",
      invalidReps: "Informe repetições válidas.",
      sessionLogged: "Treino registrado.",
      couldNotSaveSession: "Não foi possível salvar o treino.",
      sessionsCard: "Treinos",
      totalVolumeCard: "Volume total",
      noSessionsLogged: "Nenhum treino registrado ainda.",
      tableDate: "Data",
      tableExercise: "Exercício",
      tableLoad: "Carga",
      tableReps: "Repetições",
      tableVolume: "Volume",
      last60Days: "Últimos 60 dias",
      noTraining: "Sem treino",
      sessionCount: ({ count }) => `${count} treino(s)`,
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  const module = dv.container.createEl("div", { cls: "training-module" });

  const sanitizeTitle = (name) =>
    name
      .replace(/[\\/:]/g, "-")
      .replace(/\s+/g, " ")
      .trim();

  const loadExercises = () => {
    const pages = dv
      .pages(`"${exercisesFolder}"`)
      .where((p) => p.file?.path?.startsWith(`${exercisesFolder}/`));

    return pages
      .map((p) => ({ name: p.file.name, path: p.file.path }))
      .sort((a, b) => a.name.localeCompare(b.name));
  };

  const extractFields = (text) => {
    const fields = {};
    const regex = /([a-z]+)::\s*([^#\n]+?)(?=\s+[a-z]+::|$)/gi;
    let match;
    while ((match = regex.exec(text)) !== null) {
      fields[match[1].toLowerCase()] = match[2].trim();
    }
    return fields;
  };

  const parseLine = (line) => {
    const trimmed = line.trim();
    if (!trimmed.startsWith("-")) return null;
    const fields = extractFields(trimmed);
    const loadValue = fields.load ?? null;
    const repsValue = fields.reps ?? null;
    const notes = fields.notes ?? null;
    const dateValue = fields.date ?? null;

    const load = loadValue !== null ? Number(loadValue.replace(/,/g, ".")) : null;
    const reps = repsValue !== null ? Number(repsValue.replace(/,/g, ".")) : null;

    const tags = [...trimmed.matchAll(/#([^\s#]+)/g)].map((m) => m[1]);
    const dateTag = tags.find((tag) => /^\d{4}-\d{2}-\d{2}$/.test(tag));
    const dateSource = dateValue || dateTag;
    const date = dateSource ? window.moment(dateSource, "YYYY-MM-DD", true) : null;

    return {
      load: isFinite(load) ? load : null,
      reps: isFinite(reps) ? reps : null,
      notes,
      date,
      dateStr: date && date.isValid() ? date.format("YYYY-MM-DD") : null,
      volume:
        isFinite(load) && isFinite(reps) ? Number((load * reps).toFixed(2)) : null,
    };
  };

  const parseSessionsFromContent = (content) => {
    const sectionRegex = new RegExp(`##\\s*${sessionsHeading}[^\\n]*\\n([\\s\\S]*?)(?=\\n##\\s+|$)`, "i");
    const match = sectionRegex.exec(content);
    const body = match ? match[1] : "";
    return body
      .split("\n")
      .map(parseLine)
      .filter(Boolean);
  };

  let exercises = loadExercises();
  const exerciseCache = new Map();

  const loadExerciseData = async () => {
    exercises = loadExercises();
    exerciseCache.clear();
    for (const exercise of exercises) {
      try {
        const content = await dv.io.load(exercise.path);
        const sessions = parseSessionsFromContent(content).map((session) => ({
          ...session,
          exercise: exercise.name,
          path: exercise.path,
        }));
        exerciseCache.set(exercise.path, { content, sessions });
      } catch (error) {
        console.warn("Could not load exercise note:", exercise.path, error);
        exerciseCache.set(exercise.path, { content: "", sessions: [] });
      }
    }
  };

  await loadExerciseData();

  let sessions = Array.from(exerciseCache.values()).flatMap((entry) => entry.sessions);

  const exerciseList = module.createEl("div");
  exerciseList.style.marginBottom = "1rem";

  const renderExerciseList = () => {
    exerciseList.innerHTML = "";
    if (exercises.length === 0) {
      exerciseList.createEl("p", { text: t("noExercises") });
      return;
    }
    const ul = exerciseList.createEl("ul");
    exercises.forEach((exercise) => {
      const li = ul.createEl("li");
      const link = li.createEl("a", {
        text: exercise.name,
        cls: "internal-link",
        attr: { href: exercise.path },
      });
      link.style.textDecoration = "none";
    });
  };

  renderExerciseList();

  const formatNumber = (value) => {
    if (!isFinite(value)) return "0";
    return Number(value).toLocaleString(undefined, { maximumFractionDigits: 2 });
  };

  const summaryBox = module.createEl("div");
  summaryBox.style.display = "flex";
  summaryBox.style.gap = "1rem";
  summaryBox.style.flexWrap = "wrap";
  summaryBox.style.margin = "1rem 0";

  const renderSummary = () => {
    summaryBox.innerHTML = "";
    const totalSessions = sessions.length;
    const totalVolume = sessions.reduce((sum, session) => sum + (session.volume || 0), 0);

    const card = (label, value) => {
      const el = summaryBox.createEl("div");
      el.style.border = "1px solid var(--background-modifier-border)";
      el.style.borderRadius = "8px";
      el.style.padding = "0.75rem 1rem";
      el.style.minWidth = "140px";
      el.createEl("div", { text: label, attr: { role: "label" } });
      const strong = el.createEl("strong", { text: value });
      strong.style.fontSize = "1.3em";
    };

    card(t("sessionsCard"), totalSessions.toString());
    card(t("totalVolumeCard"), formatNumber(totalVolume));
  };

  renderSummary();

  const recentWrapper = module.createEl("div");
  recentWrapper.style.marginTop = "1rem";

  const renderRecent = () => {
    recentWrapper.innerHTML = "";
    if (sessions.length === 0) {
      recentWrapper.createEl("p", { text: t("noSessionsLogged") });
      return;
    }
    const table = recentWrapper.createEl("table");
    table.style.width = "100%";
    table.style.borderCollapse = "collapse";
    const headerRow = table.createEl("tr");
    [t("tableDate"), t("tableExercise"), t("tableLoad"), t("tableReps"), t("tableVolume")].forEach((label) => {
      const th = headerRow.createEl("th", { text: label });
      th.style.textAlign = "left";
      th.style.borderBottom = "1px solid var(--background-modifier-border)";
      th.style.padding = "0.35rem";
    });

    const sorted = sessions
      .filter((session) => session.date)
      .sort((a, b) => b.date.valueOf() - a.date.valueOf())
      .slice(0, 8);

    sorted.forEach((session) => {
      const tr = table.createEl("tr");
      const cells = [
        session.dateStr ?? "—",
        session.exercise,
        session.load ?? "—",
        session.reps ?? "—",
        session.volume ?? "—",
      ];
      cells.forEach((value) => {
        const td = tr.createEl("td", { text: value.toString() });
        td.style.padding = "0.35rem";
        td.style.borderBottom = "1px solid var(--background-modifier-border)";
      });
    });
  };

  renderRecent();

  const heatmapWrapper = module.createEl("div");
  heatmapWrapper.style.marginTop = "1.5rem";

  const renderHeatmap = () => {
    heatmapWrapper.innerHTML = "";
    const title = heatmapWrapper.createEl("h4", { text: t("last60Days") });
    title.style.marginBottom = "0.5rem";

    const grid = heatmapWrapper.createEl("div");
    grid.style.display = "grid";
    grid.style.gridTemplateColumns = "repeat(15, 1fr)";
    grid.style.gap = "4px";

    const counts = sessions.reduce((acc, session) => {
      if (!session.dateStr) return acc;
      acc[session.dateStr] = (acc[session.dateStr] || 0) + 1;
      return acc;
    }, {});

    const start = today.clone().subtract(59, "days");
    const maxCount = Object.values(counts).reduce((max, value) => Math.max(max, value), 0);

    for (let i = 0; i < 60; i++) {
      const day = start.clone().add(i, "days");
      const key = day.format("YYYY-MM-DD");
      const value = counts[key] || 0;
      const intensity = maxCount === 0 ? 0 : value / maxCount;
      const cell = grid.createEl("div");
      cell.style.width = "16px";
      cell.style.height = "16px";
      cell.style.borderRadius = "4px";
      cell.style.backgroundColor = intensity
        ? `rgba(76, 175, 80, ${0.2 + intensity * 0.8})`
        : "var(--background-modifier-border)";
      cell.title = `${key}: ${
        value ? t("sessionCount", { count: value }) : t("noTraining")
      }`;
    }
  };

  renderHeatmap();

  const incrementTrainingXp = async () => {
    const statsFile = app.vault.getAbstractFileByPath(statsPath);
    if (!statsFile) return;
    let content = await dv.io.load(statsPath);
    const xpRegex = /^(\s*-\s*xp:\s*)(\d+)/im;
    if (xpRegex.test(content)) {
      content = content.replace(xpRegex, (_, prefix, value) => {
        const current = Number(value) || 0;
        return `${prefix}${current + TRAINING_XP}`;
      });
    } else {
      const suffix = content.endsWith("\n") || content.length === 0 ? "" : "\n";
      content = `${content}${suffix}- xp: ${TRAINING_XP}\n`;
    }
    await app.vault.modify(statsFile, content);
  };

  const refreshAggregations = async () => {
    await loadExerciseData();
    sessions = Array.from(exerciseCache.values()).flatMap((entry) => entry.sessions);
    renderExerciseList();
    renderSummary();
    renderRecent();
    renderHeatmap();
  };

  const header = module.createEl("h3", { text: t("exercisesTitle") });
  header.style.marginTop = "1.5rem";

  const getTemplateContent = (() => {
    let cache = null;
    let attempted = false;
    return async () => {
      if (attempted) return cache;
      attempted = true;
      try {
        cache = await dv.io.load(templatePath);
      } catch (error) {
        console.warn("Could not load exercise template.", error);
        cache = null;
      }
      return cache;
    };
  })();

  const createExercise = async (rawName) => {
    const normalized = sanitizeTitle(rawName || "");
    if (!normalized) throw new Error("Provide a valid name.");
    if (exercises.some((ex) => ex.name.toLowerCase() === normalized.toLowerCase())) {
      throw new Error(t("exerciseExists"));
    }

    const targetPath = `${exercisesFolder}/${normalized}.md`;
    if (app.vault.getAbstractFileByPath(targetPath)) {
      throw new Error("File already exists.");
    }

    const template = await getTemplateContent();
    const content = template
      ? template.replace(/{{EXERCISE_NAME}}/g, normalized)
      : `# ${normalized}\n\n> [!info]\n> Track ${normalized} volume (load x reps) over time.\n\n## Sessions\n`;

    await app.vault.create(targetPath, content);
    return { name: normalized, path: targetPath };
  };

  const exerciseForm = module.createEl("form");
  exerciseForm.style.display = "flex";
  exerciseForm.style.gap = "0.5rem";
  exerciseForm.style.flexWrap = "wrap";
  exerciseForm.style.marginBottom = "1rem";

  const exerciseInput = exerciseForm.createEl("input", {
    attr: { type: "text", placeholder: t("newExercisePlaceholder") },
  });
  exerciseInput.style.flex = "1";

  const exerciseButton = exerciseForm.createEl("button", { text: t("createExercise") });
  exerciseButton.type = "submit";

  exerciseForm.onsubmit = async (event) => {
    event.preventDefault();
    const name = exerciseInput.value.trim();
    if (!name) {
      new Notice(t("typeNameFirst"));
      return;
    }
    exerciseButton.disabled = true;
    exerciseButton.textContent = t("saving");
    try {
      const created = await createExercise(name);
      exercises.push(created);
      await refreshAggregations();
      updateExerciseSelect();
      exerciseInput.value = "";
      new Notice(t("exerciseCreated"));
    } catch (error) {
      console.error(error);
      new Notice(error?.message ?? t("couldNotCreateExercise"));
    } finally {
      exerciseButton.disabled = false;
      exerciseButton.textContent = t("createExercise");
    }
  };

  const sessionHeader = module.createEl("h3", { text: t("logSession") });
  sessionHeader.style.marginTop = "1rem";

  const sessionForm = module.createEl("form");
  sessionForm.style.display = "flex";
  sessionForm.style.flexWrap = "wrap";
  sessionForm.style.gap = "0.5rem";
  sessionForm.style.marginBottom = "1rem";

  let exerciseSelect = null;
  const selectEl = sessionForm.createEl("select");
  selectEl.style.flex = "1";
  exerciseSelect = selectEl;
  const loadInput = sessionForm.createEl("input", {
    attr: { type: "number", placeholder: t("loadPlaceholder"), step: "0.5", min: "0" },
  });
  loadInput.style.flex = "0 0 110px";

  const repsInput = sessionForm.createEl("input", {
    attr: { type: "number", placeholder: t("repsPlaceholder"), min: "1", step: "1" },
  });
  repsInput.style.flex = "0 0 90px";

  const dateInput = sessionForm.createEl("input", {
    attr: { type: "date", value: todayStr },
  });
  dateInput.style.flex = "0 0 150px";

  const notesInput = sessionForm.createEl("input", {
    attr: { type: "text", placeholder: t("notesPlaceholder") },
  });
  notesInput.style.flex = "1";

  const sessionButton = sessionForm.createEl("button", { text: t("addSession") });
  sessionButton.type = "submit";

  let updateExerciseSelect = () => {};
  updateExerciseSelect = () => {
    if (!exerciseSelect) return;
    exerciseSelect.innerHTML = "";
    exercises.forEach((exercise) => {
      exerciseSelect.createEl("option", { value: exercise.name, text: exercise.name });
    });
  };

  updateExerciseSelect();

  const insertSession = async (exerciseEntry, line) => {
    const file = app.vault.getAbstractFileByPath(exerciseEntry.path);
    if (!file) throw new Error("Exercise note not found.");
    let content = exerciseCache.get(exerciseEntry.path)?.content;
    if (content === undefined) {
      content = await app.vault.read(file);
    }

    const sectionRegex = new RegExp(`(##\\s*${sessionsHeading}[^\\n]*\\n)([\\s\\S]*?)(?=\\n##\\s+|$)`, "i");
    const match = sectionRegex.exec(content);
    const lineWithBreak = line.endsWith("\n") ? line : `${line}\n`;

    if (match) {
      const before = content.slice(0, match.index);
      const after = content.slice(match.index + match[0].length);
      const body = match[2].trimEnd();
      const newBody = `${body ? body + "\n" : ""}${lineWithBreak}`;
      content = before + match[1] + newBody + after;
    } else {
      const trimmed = content.trimEnd();
      const prefix = trimmed.length ? `${trimmed}\n\n` : "";
      content = `${prefix}## ${sessionsHeading}\n${lineWithBreak}`;
    }

    await app.vault.modify(file, content);
    const sessionsList = parseSessionsFromContent(content).map((session) => ({
      ...session,
      exercise: exerciseEntry.name,
      path: exerciseEntry.path,
    }));
    exerciseCache.set(exerciseEntry.path, { content, sessions: sessionsList });
  };

  sessionForm.onsubmit = async (event) => {
    event.preventDefault();
    if (exercises.length === 0) {
      new Notice(t("createExerciseFirst"));
      return;
    }

    const selectedExercise = exerciseSelect.value;
    if (!selectedExercise) {
      new Notice(t("chooseExercise"));
      return;
    }

    const loadValue = Number(loadInput.value);
    const repsValue = Number(repsInput.value);
    const dateValue = dateInput.value || todayStr;
    const notesValue = notesInput.value.trim();

    if (!isFinite(loadValue) || loadValue <= 0) {
      new Notice(t("invalidLoad"));
      return;
    }
    if (!Number.isFinite(repsValue) || repsValue <= 0) {
      new Notice(t("invalidReps"));
      return;
    }

    sessionButton.disabled = true;
    sessionButton.textContent = t("saving");
    try {
      const exerciseEntry = exercises.find(
        (exercise) => exercise.name.toLowerCase() === selectedExercise.toLowerCase()
      );
      if (!exerciseEntry) throw new Error("Exercise not found.");

      const parts = [`- date:: ${dateValue}`, `load:: ${loadValue}`, `reps:: ${repsValue}`];
      if (notesValue) parts.push(`notes:: ${notesValue}`);
      const line = parts.join(" ");
      await insertSession(exerciseEntry, line);
      await incrementTrainingXp();
      await refreshAggregations();
      updateExerciseSelect();
      loadInput.value = "";
      repsInput.value = "";
      notesInput.value = "";
      new Notice(t("sessionLogged"));
    } catch (error) {
      console.error(error);
      new Notice(t("couldNotSaveSession"));
    } finally {
      sessionButton.disabled = false;
      sessionButton.textContent = t("addSession");
    }
  };

  module.append(
    summaryBox,
    heatmapWrapper,
    header,
    exerciseForm,
    exerciseList,
    sessionHeader,
    sessionForm,
    recentWrapper
  );
};

run();
```
