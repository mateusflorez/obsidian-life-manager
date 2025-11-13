# Pomodoro

> [!info]
> Configure focus/break intervals, receive notifications at the end of each cycle, and keep a running journal of how many minutes you stayed in deep work.

```dataviewjs
const run = async () => {
  const LOG_PATH = "pomodoro/log.md";
  const ENTRIES_HEADING = "Entries";
  const RECENT_LIMIT = 8;

  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();

  const translations = {
    en: {
      calloutTitle: "Pomodoro timer",
      calloutBody: "Start guided focus blocks, get notified at every phase change, and keep a history of your concentrated minutes.",
      statsTitle: "Focus totals",
      totalLabel: ({ minutes }) => `${minutes} min total`,
      todayLabel: ({ minutes }) => `${minutes} min today`,
      lastSessionLabel: ({ minutes, date, time }) =>
        minutes === 0 ? "—" : `${minutes} min · ${date}${time ? ` · ${time}` : ""}`,
      noSessions: "No sessions logged yet.",
      timerTitle: "Run a session",
      timerDescription: "Pick the lengths you prefer (e.g., 15 min focus / 5 min break) and press start.",
      focusLabel: "Focus (min)",
      breakLabel: "Break (min)",
      cyclesLabel: "Cycles",
      startButton: "Start",
      stopButton: "Stop",
      idleLabel: "Idle",
      focusPhase: "Focus",
      breakPhase: "Break",
      historyTitle: "Recent focus blocks",
      noHistory: "Nothing logged yet.",
      invalidFocus: "Enter a focus interval of at least 1 minute.",
      invalidBreak: "Enter a break interval of at least 1 minute.",
      invalidCycles: "Enter at least 1 cycle.",
      focusDone: "Focus finished! Time for a break.",
      breakDone: "Break finished! Back to focus.",
      fullSessionDone: "All cycles completed. Nice work!",
      notificationTitle: "Pomodoro",
      notificationPermission: "Enable notifications to see Pomodoro alerts.",
      saveError: "Could not store the session. Check the console for details.",
    },
    pt: {
      calloutTitle: "Timer Pomodoro",
      calloutBody: "Inicie blocos guiados de foco, receba alertas em cada fase e acompanhe quantos minutos de concentração você acumulou.",
      statsTitle: "Totais de foco",
      totalLabel: ({ minutes }) => `${minutes} min no total`,
      todayLabel: ({ minutes }) => `${minutes} min hoje`,
      lastSessionLabel: ({ minutes, date, time }) =>
        minutes === 0 ? "—" : `${minutes} min · ${date}${time ? ` · ${time}` : ""}`,
      noSessions: "Nenhuma sessão registrada ainda.",
      timerTitle: "Rodar uma sessão",
      timerDescription: "Escolha os tempos preferidos (ex.: 15 min foco / 5 min descanso) e pressione iniciar.",
      focusLabel: "Foco (min)",
      breakLabel: "Descanso (min)",
      cyclesLabel: "Ciclos",
      startButton: "Iniciar",
      stopButton: "Parar",
      idleLabel: "Parado",
      focusPhase: "Foco",
      breakPhase: "Descanso",
      historyTitle: "Focos recentes",
      noHistory: "Nada registrado ainda.",
      invalidFocus: "Informe um tempo de foco de pelo menos 1 minuto.",
      invalidBreak: "Informe um tempo de descanso de pelo menos 1 minuto.",
      invalidCycles: "Informe pelo menos 1 ciclo.",
      focusDone: "Foco concluído! Hora da pausa.",
      breakDone: "Descanso concluído! De volta ao foco.",
      fullSessionDone: "Todos os ciclos finalizados. Boa!",
      notificationTitle: "Pomodoro",
      notificationPermission: "Ative as notificações para ver os alertas do Pomodoro.",
      saveError: "Não foi possível salvar a sessão. Veja o console para detalhes.",
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  const GLOBAL_KEY = "__lifeManagerPomodoroState__";
  const defaultState = () => ({
    running: false,
    focusMinutes: 25,
    breakMinutes: 5,
    cyclesTarget: 4,
    cyclesCompleted: 0,
    phase: "idle",
    endsAt: null,
    currentFocusStartISO: null,
    audioContext: null,
  });

  const globalStore = window[GLOBAL_KEY] ?? { state: defaultState(), intervalId: null };
  window[GLOBAL_KEY] = globalStore;
  const state = globalStore.state;
  const renderToken = Symbol("pomodoro-render");
  globalStore.renderToken = renderToken;

  const ensureDefaults = () => {
    const defaults = defaultState();
    Object.keys(defaults).forEach((key) => {
      if (state[key] === undefined || state[key] === null) {
        state[key] = defaults[key];
      }
    });
  };
  ensureDefaults();
  if (globalStore.intervalId) {
    window.clearInterval(globalStore.intervalId);
    globalStore.intervalId = null;
  }

  const module = dv.container.createEl("div", { cls: "pomodoro-module" });
  module.style.display = "grid";
  module.style.gap = "1rem";
  module.style.gridTemplateColumns = "repeat(auto-fit, minmax(280px, 1fr))";

  const styleCard = (card) => {
    card.style.border = "1px solid var(--background-modifier-border)";
    card.style.borderRadius = "12px";
    card.style.padding = "1rem";
    card.style.backgroundColor = "var(--background-primary)";
    card.style.boxShadow = "var(--drop-shadow, none)";
  };

  const statsCard = module.createEl("div");
  styleCard(statsCard);
  const timerCard = module.createEl("div");
  styleCard(timerCard);
  const historyCard = module.createEl("div");
  historyCard.style.gridColumn = "1 / -1";
  styleCard(historyCard);

  const extractFields = (line) => {
    const fields = {};
    const regex = /([a-z]+)::\s*([^#\n]+?)(?=\s+[a-z]+::|$)/gi;
    let match;
    while ((match = regex.exec(line)) !== null) {
      fields[match[1].toLowerCase()] = match[2].trim();
    }
    return fields;
  };

  const ensureFolder = async (folderPath) => {
    if (!folderPath) return;
    const folder = app.vault.getAbstractFileByPath(folderPath);
    if (folder) return;
    const parts = folderPath.split("/");
    for (let i = 1; i <= parts.length; i += 1) {
      const partial = parts.slice(0, i).join("/");
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
    const initial = `# Pomodoro log\n\n## ${ENTRIES_HEADING}\n`;
    return app.vault.create(LOG_PATH, initial);
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
        const focus = Number(fields.focus ?? fields.minutes ?? fields.duration);
        if (!Number.isFinite(focus)) return null;
        const breakMinutes = Number(fields.break ?? fields.rest ?? 0);
        const started = fields.started || fields.start || fields.timestamp || null;
        const date = fields.date || null;
        const startedMoment = started
          ? window.moment(started, window.moment.ISO_8601, true)
          : null;
        const dateStr = date
          ? window.moment(date, "YYYY-MM-DD", true)
          : startedMoment;
        const normalizedDate = dateStr && dateStr.isValid()
          ? dateStr.format("YYYY-MM-DD")
          : startedMoment && startedMoment.isValid()
            ? startedMoment.format("YYYY-MM-DD")
            : null;
        return {
          focusMinutes: Math.max(0, Math.round(focus)),
          breakMinutes: Number.isFinite(breakMinutes)
            ? Math.max(0, Math.round(breakMinutes))
            : 0,
          startedAt: startedMoment && startedMoment.isValid() ? startedMoment : null,
          startedISOString:
            startedMoment && startedMoment.isValid()
              ? startedMoment.toISOString()
              : null,
          dateStr: normalizedDate,
          raw: line,
          sortValue:
            startedMoment && startedMoment.isValid()
              ? startedMoment.valueOf()
              : normalizedDate
                ? window.moment(normalizedDate, "YYYY-MM-DD").valueOf()
                : 0,
        };
      })
      .filter(Boolean)
      .sort((a, b) => (b.sortValue ?? 0) - (a.sortValue ?? 0));
  };

  const loadEntries = async () => {
    const file = app.vault.getAbstractFileByPath(LOG_PATH);
    if (!file) return [];
    try {
      const content = await dv.io.load(LOG_PATH);
      return parseEntries(content);
    } catch (error) {
      console.warn("Pomodoro: could not load log", error);
      return [];
    }
  };

  const appendEntry = async ({ focusMinutes, breakMinutes, startedISOString }) => {
    try {
      const file = await ensureLogFile();
      const content = await app.vault.read(file);
      const headingRegex = new RegExp(
        `(##\\s*${ENTRIES_HEADING}[^\\n]*\\n)`
      );
      const startedMoment = startedISOString
        ? window.moment(startedISOString)
        : window.moment();
      const line = `- date:: ${startedMoment.format(
        "YYYY-MM-DD"
      )} started:: ${startedMoment.toISOString()} focus:: ${focusMinutes} break:: ${breakMinutes}\n`;
      const match = headingRegex.exec(content);
      let updated;
      if (match) {
        const insertIndex = match.index + match[1].length;
        updated = `${content.slice(0, insertIndex)}${line}${content.slice(insertIndex)}`;
      } else {
        updated = `${content.trim()}\n\n## ${ENTRIES_HEADING}\n${line}`;
      }
      await app.vault.modify(file, updated);
    } catch (error) {
      console.error("Pomodoro: could not append entry", error);
      new Notice(t("saveError"));
    }
  };

  let entries = await loadEntries();

  const formatMinutes = (value) => Math.max(0, Math.round(value));

  const statsTitle = statsCard.createEl("h3", { text: t("statsTitle") });
  statsTitle.style.marginTop = "0";

  const statsGrid = statsCard.createEl("div");
  statsGrid.style.display = "grid";
  statsGrid.style.gridTemplateColumns = "repeat(auto-fit, minmax(140px, 1fr))";
  statsGrid.style.gap = ".5rem";

  const totalEl = statsGrid.createEl("div");
  const todayEl = statsGrid.createEl("div");
  const lastEl = statsGrid.createEl("div");

  const updateStats = () => {
    const today = window.moment();
    const todayStr = today.format("YYYY-MM-DD");
    const totalMinutes = entries.reduce((sum, entry) => sum + entry.focusMinutes, 0);
    const todayMinutes = entries
      .filter((entry) => entry.dateStr === todayStr)
      .reduce((sum, entry) => sum + entry.focusMinutes, 0);
    const lastEntry = entries[0] ?? null;

    totalEl.textContent = t("totalLabel", { minutes: formatMinutes(totalMinutes) });
    todayEl.textContent = t("todayLabel", { minutes: formatMinutes(todayMinutes) });
    if (!lastEntry) {
      lastEl.textContent = t("noSessions");
    } else {
      lastEl.textContent = t("lastSessionLabel", {
        minutes: formatMinutes(lastEntry.focusMinutes),
        date: lastEntry.startedAt
          ? lastEntry.startedAt.format("DD MMM YYYY")
          : lastEntry.dateStr ?? "",
        time: lastEntry.startedAt ? lastEntry.startedAt.format("HH:mm") : "",
      });
    }
  };

  const historyTitle = historyCard.createEl("h3", { text: t("historyTitle") });
  historyTitle.style.marginTop = "0";
  const historyList = historyCard.createEl("div");

  const renderHistory = () => {
    while (historyList.firstChild) {
      historyList.firstChild.remove();
    }
    if (!entries.length) {
      historyList.createEl("p", { text: t("noHistory") });
      return;
    }
    const list = historyList.createEl("ul");
    list.style.listStyle = "disc";
    list.style.paddingLeft = "1.5rem";
    entries.slice(0, RECENT_LIMIT).forEach((entry) => {
      const item = list.createEl("li");
      const dateLabel = entry.startedAt
        ? entry.startedAt.format("DD MMM YYYY · HH:mm")
        : entry.dateStr ?? "";
      item.textContent = `${formatMinutes(entry.focusMinutes)} min · ${dateLabel}`;
    });
  };

  updateStats();
  renderHistory();

  const refreshEntries = async () => {
    entries = await loadEntries();
    updateStats();
    renderHistory();
  };

  const timerTitle = timerCard.createEl("h3", { text: t("timerTitle") });
  timerTitle.style.marginTop = "0";
  timerCard.createEl("p", { text: t("timerDescription") });

  const form = timerCard.createEl("div");
  form.style.display = "grid";
  form.style.gridTemplateColumns = "repeat(auto-fit, minmax(120px, 1fr))";
  form.style.gap = ".75rem";

  const createField = (labelText, defaultValue) => {
    const wrapper = form.createEl("label");
    wrapper.style.display = "flex";
    wrapper.style.flexDirection = "column";
    wrapper.style.gap = ".25rem";
    const span = wrapper.createEl("span", { text: labelText });
    const input = wrapper.createEl("input", {
      type: "number",
      value: defaultValue,
    });
    input.min = "1";
    input.step = "1";
    input.required = true;
    input.style.padding = ".4rem";
    input.style.borderRadius = "8px";
    input.style.border = "1px solid var(--background-modifier-border)";
    return input;
  };

  const focusInput = createField(t("focusLabel"), state.focusMinutes ?? 25);
  const breakInput = createField(t("breakLabel"), state.breakMinutes ?? 5);
  const cyclesInput = createField(t("cyclesLabel"), state.cyclesTarget ?? 4);

  const timerDisplay = timerCard.createEl("div");
  timerDisplay.style.fontSize = "2.5rem";
  timerDisplay.style.fontWeight = "700";
  timerDisplay.style.textAlign = "center";
  timerDisplay.style.marginTop = "1rem";
  timerDisplay.textContent = "00:00";

  const phaseLabel = timerCard.createEl("p", { text: t("idleLabel") });
  phaseLabel.style.textAlign = "center";
  phaseLabel.style.margin = "0";

  const buttons = timerCard.createEl("div");
  buttons.style.display = "flex";
  buttons.style.gap = "0.5rem";
  buttons.style.justifyContent = "center";
  buttons.style.marginTop = "1rem";

  const startButton = buttons.createEl("button", { text: t("startButton") });
  const stopButton = buttons.createEl("button", { text: t("stopButton") });
  stopButton.disabled = true;

  globalStore.ui = { timerDisplay, phaseLabel, startButton, stopButton };

  const baseButtonStyles = (button) => {
    button.style.padding = "0.6rem 1.2rem";
    button.style.borderRadius = "999px";
    button.style.border = "none";
    button.style.fontWeight = "600";
    button.style.cursor = "pointer";
  };

  baseButtonStyles(startButton);
  startButton.style.background = "var(--interactive-accent)";
  startButton.style.color = "var(--text-on-accent, #fff)";

  baseButtonStyles(stopButton);
  stopButton.style.background = "var(--background-modifier-border)";

  const formatTimeLeft = (ms) => {
    const safeMs = Math.max(0, ms);
    const totalSeconds = Math.round(safeMs / 1000);
    const minutes = Math.floor(totalSeconds / 60)
      .toString()
      .padStart(2, "0");
    const seconds = (totalSeconds % 60).toString().padStart(2, "0");
    return `${minutes}:${seconds}`;
  };

  const updateTimerUI = () => {
    const ui = globalStore.ui;
    if (!ui) return;
    const { timerDisplay, phaseLabel, startButton, stopButton } = ui;
    if (!state.running || !state.endsAt) {
      timerDisplay.textContent = "00:00";
      phaseLabel.textContent = t("idleLabel");
      startButton.disabled = false;
      stopButton.disabled = true;
      return;
    }
    const remaining = state.endsAt - Date.now();
    timerDisplay.textContent = formatTimeLeft(remaining);
    phaseLabel.textContent = state.phase === "focus" ? t("focusPhase") : t("breakPhase");
    startButton.disabled = true;
    stopButton.disabled = false;
  };

  const clearTimerInterval = () => {
    if (globalStore.intervalId) {
      window.clearInterval(globalStore.intervalId);
      globalStore.intervalId = null;
    }
  };

  const tick = () => {
    if (!state.running || !state.endsAt) {
      updateTimerUI();
      return;
    }
    const remaining = state.endsAt - Date.now();
    if (remaining <= 0) {
      clearTimerInterval();
      state.endsAt = null;
      handlePhaseEnd();
      return;
    }
    updateTimerUI();
  };

  const startTimerLoop = () => {
    clearTimerInterval();
    if (!state.running || !state.endsAt) {
      updateTimerUI();
      return;
    }
    globalStore.intervalId = window.setInterval(tick, 500);
    updateTimerUI();
  };

  const stopTimer = ({ quiet } = { quiet: false }) => {
    state.running = false;
    clearTimerInterval();
    state.phase = "idle";
    state.endsAt = null;
    state.currentFocusStartISO = null;
    updateTimerUI();
    if (!quiet) {
      state.cyclesCompleted = 0;
    }
  };

  const ensureAudioContext = () => {
    if (state.audioContext) return state.audioContext;
    const Ctor = window.AudioContext || window.webkitAudioContext;
    if (!Ctor) return null;
    state.audioContext = new Ctor();
    return state.audioContext;
  };

  const playAlertTone = async () => {
    try {
      const ctx = ensureAudioContext();
      if (!ctx) return;
      await ctx.resume();
      const oscillator = ctx.createOscillator();
      const gain = ctx.createGain();
      oscillator.type = "triangle";
      oscillator.frequency.value = 880;
      oscillator.connect(gain);
      gain.gain.setValueAtTime(0.001, ctx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.2, ctx.currentTime + 0.01);
      gain.connect(ctx.destination);
      oscillator.start();
      gain.gain.exponentialRampToValueAtTime(0.00001, ctx.currentTime + 1);
      oscillator.stop(ctx.currentTime + 1);
    } catch (error) {
      console.warn("Pomodoro: audio blocked", error);
    }
  };

  const pushNotification = (body) => {
    if ("Notification" in window) {
      if (Notification.permission === "granted") {
        new Notification(t("notificationTitle"), { body });
      } else if (Notification.permission === "default") {
        Notification.requestPermission().then((permission) => {
          if (permission === "granted") {
            new Notification(t("notificationTitle"), { body });
          }
        });
      }
    } else {
      new Notice(body);
    }
  };

  const alertPhaseEnd = async ({ body }) => {
    await playAlertTone();
    if ("Notification" in window && Notification.permission === "denied") {
      new Notice(t("notificationPermission"));
    } else {
      pushNotification(body);
    }
  };

  const startPhase = (phase, minutes) => {
    const ui = globalStore.ui;
    const isCurrentRender = globalStore.renderToken === renderToken;
    state.running = true;
    if (isCurrentRender && ui) {
      ui.startButton.disabled = true;
      ui.stopButton.disabled = false;
    }
    if (phase === "focus" && !state.currentFocusStartISO) {
      state.currentFocusStartISO = window.moment().toISOString();
    }
    state.phase = phase;
    state.endsAt = Date.now() + minutes * 60 * 1000;
    if (isCurrentRender) {
      startTimerLoop();
    } else if (typeof globalStore.sync === "function") {
      globalStore.sync();
    }
  };

  const handlePhaseEnd = () => {
    if (state.phase === "focus") {
      state.cyclesCompleted += 1;
      const startedISOString = state.currentFocusStartISO ?? window.moment().toISOString();
      state.currentFocusStartISO = null;
      (async () => {
        await appendEntry({
          focusMinutes: state.focusMinutes,
          breakMinutes: state.breakMinutes,
          startedISOString,
        });
        await refreshEntries();
        await alertPhaseEnd({ body: t("focusDone") });
        if (state.cyclesCompleted >= state.cyclesTarget) {
          new Notice(t("fullSessionDone"));
          stopTimer({ quiet: true });
          state.cyclesCompleted = 0;
          return;
        }
        if (state.breakMinutes > 0) {
          startPhase("break", state.breakMinutes);
        } else {
          state.currentFocusStartISO = window.moment().toISOString();
          startPhase("focus", state.focusMinutes);
        }
      })();
    } else if (state.phase === "break") {
      (async () => {
        await alertPhaseEnd({ body: t("breakDone") });
        state.currentFocusStartISO = window.moment().toISOString();
        startPhase("focus", state.focusMinutes);
      })();
    }
  };

  startButton.addEventListener("click", async () => {
    const focusMinutes = Number(focusInput.value);
    if (!Number.isFinite(focusMinutes) || focusMinutes < 1) {
      new Notice(t("invalidFocus"));
      return;
    }
    const breakMinutes = Number(breakInput.value);
    if (!Number.isFinite(breakMinutes) || breakMinutes < 1) {
      new Notice(t("invalidBreak"));
      return;
    }
    const cycles = Number(cyclesInput.value);
    if (!Number.isFinite(cycles) || cycles < 1) {
      new Notice(t("invalidCycles"));
      return;
    }

    state.focusMinutes = Math.round(focusMinutes);
    state.breakMinutes = Math.round(breakMinutes);
    state.cyclesTarget = Math.round(cycles);
    state.cyclesCompleted = 0;
    state.currentFocusStartISO = window.moment().toISOString();

    ensureAudioContext();
    if ("Notification" in window && Notification.permission === "default") {
      Notification.requestPermission();
    }

    startPhase("focus", state.focusMinutes);
  });

  stopButton.addEventListener("click", () => {
    stopTimer({ quiet: true });
  });

  const bootstrapTimer = () => {
    if (state.running && state.endsAt) {
      if (state.endsAt <= Date.now()) {
        tick();
      } else {
        startTimerLoop();
      }
      return;
    }
    if (state.running && !state.endsAt) {
      state.running = false;
      state.phase = "idle";
    }
    updateTimerUI();
  };

  globalStore.sync = bootstrapTimer;
  bootstrapTimer();
};

run();
```
