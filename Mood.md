# Mood

```dataviewjs
const run = async () => {
  const LOG_PATH = "mood/log.md";
  const ENTRIES_HEADING = "Entries";
  const DAYS = 60;
  const RECENT_LIMIT = 6;
  const statsPath = "profile/stats.md";
  const MOOD_XP = 10;
  const faces = {
    1: "😞",
    2: "😕",
    3: "😐",
    4: "🙂",
    5: "😄",
  };

  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();

  const translations = {
    en: {
      calloutTitle: "Mood tracker",
      calloutBody: "Follow the last 60 days and jot optional notes to remember how each day felt.",
      dateLabel: "Entry date",
      dateInvalid: "Pick a valid date.",
      recentEntriesTitle: "Recent entries",
      noRecentEntries: "No recent notes yet.",
      noteLabel: "Note",
      chartTitle: "Mood trend (last 60 days)",
      noEntries: "No mood entries yet.",
      chartFallback: "Chart unavailable (offline?). Entries are still saved.",
      logTitle: "Log today's mood",
      logDescription: "Choose how the day went (1 = sad, 5 = happy) and leave an optional note.",
      sliderLabel: "Mood score",
      sliderValueLabel: ({ value, face }) => `${face} Mood: ${value}/5`,
      notePlaceholder: "Optional note about today",
      saveButton: "Save mood",
      saving: "Saving...",
      saved: "Mood saved!",
      saveError: "Could not save mood. Check the console for details.",
    },
    pt: {
      calloutTitle: "Rastreador de humor",
      calloutBody: "Acompanhe os últimos 60 dias e escreva notas opcionais para lembrar como cada dia foi.",
      dateLabel: "Data do registro",
      dateInvalid: "Escolha uma data válida.",
      recentEntriesTitle: "Últimos registros",
      noRecentEntries: "Nenhuma nota recente.",
      noteLabel: "Nota",
      chartTitle: "Humor (últimos 60 dias)",
      noEntries: "Nenhum registro de humor ainda.",
      chartFallback: "Não foi possível carregar o gráfico (offline?). Os registros foram salvos.",
      logTitle: "Registrar humor de hoje",
      logDescription: "Escolha como o dia foi (1 = triste, 5 = feliz) e escreva uma nota opcional.",
      sliderLabel: "Nota do humor",
      sliderValueLabel: ({ value, face }) => `${face} Humor: ${value}/5`,
      notePlaceholder: "Nota opcional sobre o dia",
      saveButton: "Salvar humor",
      saving: "Salvando...",
      saved: "Humor registrado!",
      saveError: "Não foi possível salvar o humor. Veja o console para detalhes.",
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  const callout = dv.container.createEl("div", {
    cls: "callout",
    attr: { "data-callout": "info" },
  });
  callout.createEl("div", {
    cls: "callout-title",
    text: t("calloutTitle"),
  });
  callout.createEl("div", {
    cls: "callout-content",
    text: t("calloutBody"),
  });

  const module = dv.container.createEl("div", { cls: "mood-hub" });
  module.style.display = "grid";
  module.style.gap = "1rem";
  module.style.gridTemplateColumns = "repeat(auto-fit, minmax(280px, 1fr))";

  const styleCard = (card) => {
    card.classList.add("mood-card");
    card.style.border = "1px solid var(--background-modifier-border)";
    card.style.borderRadius = "12px";
    card.style.padding = "1rem";
    card.style.backgroundColor = "var(--background-primary)";
    card.style.boxShadow = "var(--drop-shadow, none)";
  };

  const chartCard = module.createEl("div");
  styleCard(chartCard);
  const formCard = module.createEl("div");
  styleCard(formCard);
  const recentCard = module.createEl("div");
  recentCard.style.gridColumn = "1 / -1";
  styleCard(recentCard);

  const extractFields = (line) => {
    const fields = {};
    const regex = /([a-z]+)::\s*([^#\n]+?)(?=\s+[a-z]+::|$)/gi;
    let match;
    while ((match = regex.exec(line)) !== null) {
      fields[match[1].toLowerCase()] = match[2].trim();
    }
    return fields;
  };

  const parseEntries = (content) => {
    const sectionRegex = new RegExp(
      `##\\s*${ENTRIES_HEADING}[^\\n]*\\n([\\s\\S]*?)(?=\\n##\\s+|$)`,
      "i"
    );
    const match = sectionRegex.exec(content);
    const body = match ? match[1] : content;
    return body
      .split("\n")
      .map((line) => line.trim())
      .filter((line) => line.startsWith("-"))
      .map((line) => {
        const fields = extractFields(line);
        const moodValue = Number(fields.mood ?? fields.score ?? fields.rating);
        const dateStr =
          fields.date ||
          (line.match(/#(\d{4}-\d{2}-\d{2})/)?.[1] ?? null);
        const note = fields.note ?? fields.notes ?? "";
        const date = dateStr
          ? window.moment(dateStr, "YYYY-MM-DD", true)
          : null;
        if (!date || !date.isValid() || !Number.isFinite(moodValue)) {
          return null;
        }
        return {
          date,
          dateStr: date.format("YYYY-MM-DD"),
          mood: Math.min(5, Math.max(1, Math.round(moodValue))),
          note: note.trim(),
          raw: line,
        };
      })
      .filter(Boolean);
  };

  const ensureFolder = async (folderPath) => {
    if (!folderPath) return;
    const folder = app.vault.getAbstractFileByPath(folderPath);
    if (folder) return;
    const segments = folderPath.split("/");
    for (let i = 1; i <= segments.length; i += 1) {
      const partial = segments.slice(0, i).join("/");
      if (!partial) continue;
      const exists = app.vault.getAbstractFileByPath(partial);
      if (!exists) {
        await app.vault.createFolder(partial);
      }
    }
  };

  const ensureLogFile = async () => {
    const file = app.vault.getAbstractFileByPath(LOG_PATH);
    if (file) return file;
    const folderPath = LOG_PATH.split("/").slice(0, -1).join("/");
    await ensureFolder(folderPath);
    const initialContent = `# Mood log\n\n## ${ENTRIES_HEADING}\n`;
    return app.vault.create(LOG_PATH, initialContent);
  };

  const loadEntries = async () => {
    const file = app.vault.getAbstractFileByPath(LOG_PATH);
    if (!file) return [];
    try {
      const content = await dv.io.load(LOG_PATH);
      return parseEntries(content);
    } catch (error) {
      console.warn("Could not load mood log:", error);
      return [];
    }
  };

  const insertEntry = async (line) => {
    const file = await ensureLogFile();
    const content = await app.vault.read(file);
    const sectionRegex = new RegExp(
      `(##\\s*${ENTRIES_HEADING}[^\\n]*\\n)([\\s\\S]*?)(?=\\n##\\s+|$)`,
      "i"
    );
    const match = sectionRegex.exec(content);
    const lineWithBreak = line.endsWith("\n") ? line : `${line}\n`;

    if (match) {
      const before = content.slice(0, match.index);
      const after = content.slice(match.index + match[0].length);
      const heading = match[1];
      const body = match[2].trimEnd();
      const newBody = `${body ? body + "\n" : ""}${lineWithBreak}`;
      const next = `${before}${heading}${newBody}\n${after}`.replace(
        /\n{3,}/g,
        "\n\n"
      );
      await app.vault.modify(file, next.trimEnd() + "\n");
    } else {
      const nextContent = `${content.trim()}\n\n## ${ENTRIES_HEADING}\n${lineWithBreak}`;
      await app.vault.modify(file, nextContent.trimEnd() + "\n");
    }
  };

  const incrementXp = async () => {
    const statsFile = app.vault.getAbstractFileByPath(statsPath);
    if (!statsFile) return;
    let content = "";
    try {
      content = await dv.io.load(statsPath);
    } catch {
      return;
    }
    const xpRegex = /^(\s*-\s*xp:\s*)(\d+)/im;
    if (xpRegex.test(content)) {
      content = content.replace(xpRegex, (_, prefix, value) => {
        const current = Number(value) || 0;
        return `${prefix}${current + MOOD_XP}`;
      });
    } else {
      const suffix = content.endsWith("\n") || content.length === 0 ? "" : "\n";
      content = `${content}${suffix}- xp: ${MOOD_XP}\n`;
    }
    await app.vault.modify(statsFile, content);
  };

  const loadChartJs = (() => {
    let promise;
    return () => {
      if (window.Chart) return Promise.resolve();
      if (!promise) {
        promise = new Promise((resolve, reject) => {
          const script = document.createElement("script");
          script.src = "https://cdn.jsdelivr.net/npm/chart.js";
          script.onload = resolve;
          script.onerror = reject;
          document.head.appendChild(script);
        });
      }
      return promise;
    };
  })();

  let chartInstance = null;
  let entries = await loadEntries();

  const buildChartPoints = () => {
    const today = window.moment().startOf("day");
    const grouped = entries.reduce((acc, entry) => {
      if (!entry.dateStr) return acc;
      if (!acc[entry.dateStr]) acc[entry.dateStr] = [];
      acc[entry.dateStr].push(entry.mood);
      return acc;
    }, {});

    const points = [];
    for (let offset = DAYS - 1; offset >= 0; offset -= 1) {
      const day = today.clone().subtract(offset, "days");
      const key = day.format("YYYY-MM-DD");
      const label =
        language === "pt"
          ? day.format("DD/MM")
          : day.format("MMM D");
      const values = grouped[key] ?? [];
      const average =
        values.length > 0
          ? Number(
              (
                values.reduce((sum, value) => sum + value, 0) /
                values.length
              ).toFixed(2)
            )
          : null;
      points.push({ label, value: average });
    }
    return points;
  };

  const renderChart = async () => {
    chartCard.innerHTML = "";
    chartCard.createEl("h3", { text: t("chartTitle") });

    if (entries.length === 0) {
      chartCard.createEl("p", { text: t("noEntries") });
      return;
    }

    const canvasWrapper = chartCard.createEl("div");
    canvasWrapper.style.minHeight = "280px";
    const canvas = canvasWrapper.createEl("canvas");

    const points = buildChartPoints();
    const labels = points.map((point) => point.label);
    const values = points.map((point) => point.value);

    try {
      await loadChartJs();
    } catch (error) {
      console.warn("Chart.js unavailable:", error);
      chartCard.createEl("p", { text: t("chartFallback") });
      return;
    }

    if (chartInstance) {
      chartInstance.destroy();
      chartInstance = null;
    }

    chartInstance = new Chart(canvas, {
      type: "line",
      data: {
        labels,
        datasets: [
          {
            label: t("sliderLabel"),
            data: values,
            borderColor: "var(--color-yellow, #facc15)",
            backgroundColor: "rgba(250, 204, 21, 0.15)",
            borderWidth: 2,
            tension: 0.35,
            pointRadius: 3,
            pointBackgroundColor: "var(--color-yellow, #facc15)",
            spanGaps: true,
          },
        ],
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          legend: { display: false },
        },
        scales: {
          y: {
            min: 1,
            max: 5,
            ticks: {
              stepSize: 1,
            },
          },
        },
      },
    });
  };

  const renderRecentEntries = () => {
    recentCard.innerHTML = "";
    recentCard.createEl("h3", { text: t("recentEntriesTitle") });
    if (entries.length === 0) {
      recentCard.createEl("p", { text: t("noRecentEntries") });
      return;
    }
    const list = recentCard.createEl("div");
    list.style.display = "flex";
    list.style.flexDirection = "column";
    list.style.gap = "0.75rem";

    const sorted = [...entries].sort((a, b) => {
      if (!a.date || !b.date) return 0;
      return b.date.valueOf() - a.date.valueOf();
    });

    sorted.slice(0, RECENT_LIMIT).forEach((entry) => {
      const item = list.createEl("div");
      item.style.display = "flex";
      item.style.flexDirection = "column";
      item.style.border = "1px solid var(--background-modifier-border)";
      item.style.borderRadius = "8px";
      item.style.padding = "0.65rem";
      item.style.gap = "0.35rem";

      const header = item.createEl("div");
      header.style.display = "flex";
      header.style.justifyContent = "space-between";
      header.style.alignItems = "center";
      header.style.fontWeight = "600";

      const dateLabel = language === "pt"
        ? entry.date.format("DD/MM/YYYY")
        : entry.date.format("MMM D, YYYY");

      header.createEl("span", { text: dateLabel });
      header.createEl("span", {
        text: `${faces[entry.mood] ?? "🙂"} ${entry.mood}/5`,
      });

      if (entry.note) {
        const noteLabel = item.createEl("span");
        noteLabel.style.fontSize = "0.85rem";
        noteLabel.style.color = "var(--text-muted)";
        noteLabel.textContent = `${t("noteLabel")}: ${entry.note}`;
      }
    });
  };

  const buildForm = () => {
    formCard.innerHTML = "";
    formCard.createEl("h3", { text: t("logTitle") });
    formCard.createEl("p", { text: t("logDescription") });

    const dateLabel = formCard.createEl("label", {
      text: t("dateLabel"),
    });
    dateLabel.style.display = "block";
    dateLabel.style.fontWeight = "600";
    dateLabel.style.marginBottom = "0.25rem";

    const dateField = formCard.createEl("input", {
      type: "date",
    });
    dateField.value = window.moment().format("YYYY-MM-DD");
    dateField.style.width = "100%";
    dateField.style.marginBottom = "0.75rem";

    const sliderLabel = formCard.createEl("label", {
      text: t("sliderLabel"),
    });
    sliderLabel.style.display = "block";
    sliderLabel.style.fontWeight = "600";
    sliderLabel.style.marginBottom = "0.25rem";

    const sliderRow = formCard.createEl("div");
    sliderRow.style.display = "grid";
    sliderRow.style.gridTemplateColumns = "auto 1fr auto";
    sliderRow.style.gap = "0.5rem";
    sliderRow.style.alignItems = "center";

    const minFace = sliderRow.createEl("span", { text: `${faces[1]} 1` });
    minFace.style.fontSize = "1.2rem";

    const slider = sliderRow.createEl("input");
    slider.type = "range";
    slider.min = "1";
    slider.max = "5";
    slider.step = "1";
    slider.value = "3";
    slider.style.width = "100%";

    const maxFace = sliderRow.createEl("span", { text: `5 ${faces[5]}` });
    maxFace.style.fontSize = "1.2rem";
    maxFace.style.textAlign = "right";

    const sliderValue = formCard.createEl("div");
    sliderValue.style.marginTop = "0.25rem";
    sliderValue.style.fontWeight = "500";

    const updateSliderValue = () => {
      const value = String(slider.value);
      sliderValue.textContent = t("sliderValueLabel", {
        value,
        face: faces[value] ?? faces[3],
      });
    };
    slider.addEventListener("input", updateSliderValue);
    updateSliderValue();

    const noteField = formCard.createEl("textarea", {
      placeholder: t("notePlaceholder"),
    });
    noteField.rows = 4;
    noteField.style.marginTop = "0.75rem";
    noteField.style.width = "100%";
    noteField.style.resize = "vertical";

    const saveButton = formCard.createEl("button", {
      text: t("saveButton"),
    });
    saveButton.type = "button";
    saveButton.style.marginTop = "0.75rem";
    saveButton.style.width = "100%";
    saveButton.style.padding = "0.75rem";
    saveButton.style.fontWeight = "600";
    saveButton.style.borderRadius = "8px";
    saveButton.style.border = "none";
    saveButton.style.backgroundColor = "var(--interactive-accent)";
    saveButton.style.color = "var(--text-on-accent)";
    saveButton.style.cursor = "pointer";

    const sanitizeNote = (text) =>
      text
        .replace(/\r?\n/g, " ")
        .replace(/\s+/g, " ")
        .trim();

    const handleSave = async () => {
      if (saveButton.disabled) return;
      const rating = Number(slider.value);
      if (!Number.isFinite(rating)) return;
      const note = sanitizeNote(noteField.value);
      const chosenDate = dateField.value?.trim() || window.moment().format("YYYY-MM-DD");
      const parsed = window.moment(chosenDate, "YYYY-MM-DD", true);
      if (!parsed.isValid()) {
        if (typeof Notice === "function") new Notice(t("dateInvalid"));
        return;
      }
      const dateStr = parsed.format("YYYY-MM-DD");
      const line =
        `- date:: ${dateStr} mood:: ${rating}` +
        (note ? ` note:: ${note}` : "");

      saveButton.disabled = true;
      saveButton.textContent = t("saving");

      try {
        await insertEntry(line);
        await incrementXp();
        noteField.value = "";
        updateSliderValue();
        if (typeof Notice === "function") {
          new Notice(t("saved"));
        }
        entries = await loadEntries();
        await renderChart();
        renderRecentEntries();
      } catch (error) {
        console.error("Could not save mood entry:", error);
        if (typeof Notice === "function") {
          new Notice(t("saveError"));
        }
      } finally {
        saveButton.disabled = false;
        saveButton.textContent = t("saveButton");
      }
    };

    saveButton.addEventListener("click", handleSave);
  };

  await renderChart();
  renderRecentEntries();
  buildForm();
};

run();
```
