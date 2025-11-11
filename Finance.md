---
properties:
  file.folder:
    displayName: Month
  statement:
    displayName: Statement
  spotify:
    displayName: Spotify
  mariana:
    displayName: Mariana
  salary:
    displayName: Salary

views:
  - type: table
    name: Finance - Table
    filters:
      and:
        - file.folder.startsWith("finance/")
    groupBy: file.folder
    order:
      - file.folder
    sort:
      - property: file.folder
        direction: DESC
    columnSize:
      file.folder: 150
      statement: 100
      spotify: 100
      mariana: 100
      salary: 100
---

# Finance

> [!info]
> Use the form below to create a finance month with the correct `finance/<year>/<Month>.md` path.

```dataviewjs
const run = async () => {
  const templatePath = "templates/new finance month.md";
  const container = dv.container.createEl("div", { cls: "finance-month-creator" });
  container.style.border = "1px solid var(--background-modifier-border)";
  container.style.borderRadius = "10px";
  container.style.padding = "1rem";
  container.style.marginBottom = "1rem";
  container.style.display = "flex";
  container.style.flexDirection = "column";
  container.style.gap = "0.5rem";

  container.createEl("strong", { text: "New finance month" });

  const form = container.createEl("form");
  form.style.display = "flex";
  form.style.flexWrap = "wrap";
  form.style.gap = "0.5rem";

  const today = window.moment();
  const currentYear = today.year();
  const years = [currentYear - 1, currentYear, currentYear + 1, currentYear + 2];
  const months = window.moment.months();

  const yearSelect = form.createEl("select");
  yearSelect.style.flex = "0 0 140px";
  years.forEach((year) => {
    yearSelect.createEl("option", { text: year, value: year });
  });
  yearSelect.value = currentYear;

  const monthSelect = form.createEl("select");
  monthSelect.style.flex = "1 1 200px";
  months.forEach((month) => {
    monthSelect.createEl("option", { text: month, value: month });
  });
  monthSelect.value = today.format("MMMM");

  const submitBtn = form.createEl("button", { text: "Create month" });
  submitBtn.type = "submit";
  submitBtn.style.cursor = "pointer";

  const loadTemplate = (() => {
    let cache = null;
    let tried = false;
    return async () => {
      if (tried) return cache;
      tried = true;
      try {
        cache = await dv.io.load(templatePath);
      } catch (error) {
        console.warn("Could not load finance template.", error);
        cache = null;
      }
      return cache;
    };
  })();

  const ensureFolder = async (folderPath) => {
    if (app.vault.getAbstractFileByPath(folderPath)) return;
    await app.vault.createFolder(folderPath);
  };

  const buildContent = async (monthName, year) => {
    const template = await loadTemplate();
    if (!template) {
      return `# ${monthName} ${year}\n\n## Expenses\n\n## Income\n`;
    }
    return template
      .replace(/<%\s*tp\.file\.title\s*%>/g, monthName)
      .replace(/<%\s*moment\(\)\.format\('YYYY'\)\s*%>/g, year);
  };

  form.onsubmit = async (event) => {
    event.preventDefault();
    const year = Number(yearSelect.value);
    const monthName = monthSelect.value;
    const folderPath = `finance/${year}`;
    const targetPath = `${folderPath}/${monthName}.md`;
    submitBtn.disabled = true;
    const prevText = submitBtn.textContent;
    submitBtn.textContent = "Creating...";
    try {
      await ensureFolder(folderPath);
      if (app.vault.getAbstractFileByPath(targetPath)) {
        new Notice(`Month already exists: ${targetPath}`);
        app.workspace.openLinkText(targetPath, "", false);
      } else {
        const content = await buildContent(monthName, year);
        await app.vault.create(targetPath, content);
        new Notice(`Created ${targetPath}`);
        app.workspace.openLinkText(targetPath, "", false);
      }
    } catch (error) {
      console.error(error);
      new Notice("Could not create the month.");
    } finally {
      submitBtn.disabled = false;
      submitBtn.textContent = prevText;
    }
  };
};

run();
```

```dataviewjs
const run = async () => {
  const cardsFolder = "finance/cards";
  const recurrencesPath = "finance/recurrences.md";
  const MONTH_FORMAT = "YYYY-MM";

  const slugify = (value = "") =>
    value
      .toString()
      .normalize("NFD")
      .replace(/[\u0300-\u036f]/g, "")
      .replace(/[^a-zA-Z0-9]+/g, "-")
      .replace(/-+/g, "-")
      .replace(/^-+|-+$/g, "")
      .toLowerCase();

  const sanitizeFileName = (value = "card") =>
    value
      .toString()
      .trim()
      .replace(/[\\/:*?"<>|]/g, "-")
      .replace(/\s+/g, " ")
      .trim() || "card";

  const parseFields = (line = "") => {
    const fields = {};
    const regex = /([a-zA-Z]+)::\s*([^#\n]+?)(?=\s+[a-zA-Z]+::|$)/g;
    let match;
    while ((match = regex.exec(line)) !== null) {
      fields[match[1].toLowerCase()] = match[2].trim();
    }
    return fields;
  };

  const parseSectionLines = (content = "", heading = "") => {
    if (!heading) return [];
    const sectionRegex = new RegExp(`##\\s*${heading}[^\\n]*\\n([\\s\\S]*?)(?=\\n##\\s+|$)`, "i");
    const match = sectionRegex.exec(content);
    const body = match ? match[1] : "";
    return body
      .split("\n")
      .map((line) => line.trim())
      .filter((line) => line.startsWith("-"))
      .map(parseFields);
  };

  const ensureFolder = async (folderPath) => {
    if (app.vault.getAbstractFileByPath(folderPath)) return;
    try {
      await app.vault.createFolder(folderPath);
    } catch (error) {
      if (!/exists/i.test(error?.message ?? "")) {
        console.warn("Could not create folder:", folderPath, error);
      }
    }
  };

  const ensureRecurrenceFile = async () => {
    const file = app.vault.getAbstractFileByPath(recurrencesPath);
    if (file) return;
    await app.vault.create(
      recurrencesPath,
      "# Recurring Expenses\n\n> [!info]\n> Managed via Finance dashboard. Edit with caution.\n\n## Recurrences\n"
    );
  };

  const insertLine = async (path, heading, line) => {
    const file = app.vault.getAbstractFileByPath(path);
    if (!file) return;
    const content = await app.vault.read(file);
    const sectionRegex = new RegExp(`(##\\s*${heading}[^\\n]*\\n)([\\s\\S]*?)(?=\\n##\\s+|$)`, "i");
    const match = sectionRegex.exec(content);
    const lineWithBreak = line.endsWith("\n") ? line : `${line}\n`;

    if (match) {
      const before = content.slice(0, match.index);
      const afterContent = content.slice(match.index + match[0].length);
      const body = match[2].trimEnd();
      const newBody = `${body ? body + "\n" : ""}${lineWithBreak}`;
      const updated = before + match[1] + newBody + afterContent;
      await app.vault.modify(file, updated);
    } else {
      const updated = `${content.trim()}\n\n## ${heading}\n${lineWithBreak}`;
      await app.vault.modify(file, updated);
    }
  };

  const parseCharge = (fields, slug, fallbackName) => {
    const rawAmount = fields.amount ?? fields.value ?? "0";
    const amount = Number(rawAmount.toString().replace(",", "."));
    if (!isFinite(amount) || amount === 0) return null;
    const rawMonth = fields.month ?? fields.statement ?? "";
    const monthMoment = window.moment(rawMonth, ["YYYY-MM", "MMM YYYY", "MMMM YYYY"], true);
    if (!monthMoment?.isValid()) return null;
    const id = fields.id || `${slug}-${monthMoment.format(MONTH_FORMAT)}-${Math.random().toString(36).slice(2, 6)}`;
    return {
      id,
      amount,
      monthKey: monthMoment.format(MONTH_FORMAT),
      category: fields.category || fallbackName,
      note: fields.note || fields.description || "",
      tag: `card-${slug}-${slugify(id)}`,
    };
  };

  const loadCardsData = async (dvApi) => {
    await ensureFolder(cardsFolder);
    const pages = dvApi
      .pages(`"${cardsFolder}"`)
      .where((p) => p.file?.path?.startsWith(`${cardsFolder}/`))
      .array();
    const cards = [];
    const chargesByMonth = new Map();

    for (const page of pages) {
      const name = page.cardname ?? page.cardName ?? page.file.name.replace(".md", "");
      const slug = slugify(name) || `card-${cards.length + 1}`;
      const limit = Number(page.limit ?? page.file.frontmatter?.limit ?? 0) || 0;
      const closeDay = Number(page.closeday ?? page.closeDay ?? 5) || 5;
      const dueDay = Number(page.dueday ?? page.dueDay ?? closeDay + 5) || closeDay + 5;
      let content = "";
      try {
        content = await dvApi.io.load(page.file.path);
      } catch (error) {
        console.warn("Could not load card file:", page.file.path, error);
      }
      const charges = parseSectionLines(content, "Charges")
        .map((fields) => parseCharge(fields, slug, name))
        .filter(Boolean);

      charges.forEach((charge) => {
        const bucket = chargesByMonth.get(charge.monthKey) ?? [];
        bucket.push({
          cardName: name,
          cardSlug: slug,
          cardLimit: limit,
          charge,
        });
        chargesByMonth.set(charge.monthKey, bucket);
      });

      cards.push({
        name,
        slug,
        limit,
        closeDay,
        dueDay,
        path: page.file.path,
        charges,
      });
    }

    return { cards, chargesByMonth };
  };

  const loadRecurrences = async (dvApi) => {
    await ensureRecurrenceFile();
    let content = "";
    try {
      content = await dvApi.io.load(recurrencesPath);
    } catch (error) {
      console.warn("Could not read recurrences file.", error);
      return [];
    }
    return parseSectionLines(content, "Recurrences").map((fields) => {
      const value = Number((fields.value ?? "0").toString().replace(",", ".")) || 0;
      const startMoment = fields.start
        ? window.moment(fields.start, ["YYYY-MM", "MMM YYYY"], true)
        : null;
      const endMoment = fields.end
        ? window.moment(fields.end, ["YYYY-MM", "MMM YYYY"], true)
        : null;
      return {
        id: fields.id || `rec-${Math.random().toString(36).slice(2, 8)}`,
        title: fields.title || fields.name || "Recurring expense",
        category: fields.category || "recurring",
        value,
        note: fields.note || "",
        active: (fields.active ?? "true").toString().toLowerCase() !== "false",
        start: fields.start || "",
        end: fields.end || "",
        startMoment: startMoment?.isValid() ? startMoment : null,
        endMoment: endMoment?.isValid() ? endMoment : null,
      };
    });
  };

  const saveRecurrences = async (entries) => {
    await ensureRecurrenceFile();
    const header =
      "# Recurring Expenses\n\n> [!info]\n> Managed via Finance dashboard. Edit with caution.\n\n## Recurrences\n";
    const lines = entries
      .map((entry) => {
        const parts = [
          `id:: ${entry.id}`,
          `title:: ${entry.title}`,
          `category:: ${entry.category}`,
          `value:: ${Number(entry.value).toFixed(2)}`,
          `start:: ${entry.start || ""}`,
        ];
        if (entry.end) parts.push(`end:: ${entry.end}`);
        parts.push(`active:: ${entry.active !== false}`);
        if (entry.note) parts.push(`note:: ${entry.note}`);
        return `- ${parts.join(" ")}`;
      })
      .join("\n");
    const file = app.vault.getAbstractFileByPath(recurrencesPath);
    await app.vault.modify(file, `${header}${lines ? `${lines}\n` : ""}`);
  };

  const financePages = dv
    .pages('"finance"')
    .where((p) => {
      const parts = p.file.folder.split("/");
      return parts.length === 2 && parts[0] === "finance" && !isNaN(Number(parts[1]));
    })
    .array();

  const loadCategories = (() => {
    const cache = {};
    return async (type) => {
      if (cache[type]) return cache[type];
      try {
        const file = app.vault.getAbstractFileByPath("config/consts.md");
        if (!file) return (cache[type] = []);
        const content = await app.vault.read(file);
        const match = content.match(new RegExp(`##\\s*${type}[\\s\\S]*?(?=\\n##|$)`, "i"));
        if (!match) return (cache[type] = []);
        const items = match[0]
          .split("\n")
          .map((line) => line.trim())
          .filter((line) => line.startsWith("-"))
          .map((line) => line.replace(/^-\s*/, "").trim())
          .filter(Boolean);
        return (cache[type] = items);
      } catch {
        return (cache[type] = []);
      }
    };
  })();

  const cardsData = await loadCardsData(dv);
  const recurrenceEntries = await loadRecurrences(dv);

  const ensureExpenseLine = async (path, tag, line) => {
    let content;
    try {
      content = await dv.io.load(path);
    } catch {
      return;
    }
    if (tag && content.includes(`#${tag}`)) return;
    await insertLine(path, "Expenses", line);
  };

  const shouldApplyRecurrence = (entry, monthMoment) => {
    if (!entry.active) return false;
    if (entry.startMoment && monthMoment.isBefore(entry.startMoment, "month")) return false;
    if (entry.endMoment && monthMoment.isAfter(entry.endMoment, "month")) return false;
    return true;
  };

  for (const page of financePages) {
    const [_, year] = page.file.folder.split("/");
    const monthName = page.file.name.replace(".md", "");
    const monthMoment = window.moment(`${monthName} ${year}`, "MMMM YYYY", true);
    if (!monthMoment?.isValid()) continue;
    const monthKey = monthMoment.format(MONTH_FORMAT);

    const charges = cardsData.chargesByMonth.get(monthKey) ?? [];
    const recs = recurrenceEntries.filter((entry) => shouldApplyRecurrence(entry, monthMoment));
    if (!charges.length && !recs.length) continue;

    for (const item of charges) {
      const amountStr = item.charge.amount.toFixed(2);
      const noteTag = item.charge.note ? ` #${slugify(item.charge.note)}` : "";
      const category = item.charge.category || `${item.cardName} card`;
      const line = `- ${category}:: ${amountStr} #card-bill #${item.charge.tag}${noteTag}`;
      await ensureExpenseLine(page.file.path, item.charge.tag, line);
    }

    for (const entry of recs) {
      const recTag = `recurrence-${entry.id}-${monthKey}`;
      const amountStr = Number(entry.value).toFixed(2);
      const titleTag = entry.title ? ` #${slugify(entry.title)}` : "";
      const line = `- ${entry.category}:: ${amountStr} #recurring #${recTag}${titleTag}`;
      await ensureExpenseLine(page.file.path, recTag, line);
    }
  }

  window.financeUtils = {
    cardsFolder,
    recurrencesPath,
    slugify,
    sanitizeFileName,
    ensureFolder,
    insertLine,
    loadCardsData: (dvApi) => loadCardsData(dvApi),
    loadRecurrences: (dvApi) => loadRecurrences(dvApi),
    saveRecurrences,
    loadCategories,
  };
};

run();
```

## Credit cards

```dataviewjs
const run = async () => {
  const toolkit = window.financeUtils;
  if (!toolkit) {
    dv.paragraph("Reload this dashboard to initialize credit card tools.");
    return;
  }

  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();
  const currency = (settingsPage.currency || "BRL").toUpperCase();

  const translations = {
    en: {
      sectionTitle: "Credit cards",
      addCardTitle: "Add card",
      cardName: "Card name",
      limit: "Limit",
      closeDay: "Closing day",
      dueDay: "Due day",
      saveCard: "Save card",
      noCards: "No cards yet. Add one to start tracking statements.",
      utilization: ({ spent, limit }) => `${spent} outstanding of ${limit}`,
      upcomingTitle: "Upcoming statements",
      upcomingEmpty: "No future charges yet.",
      newCharge: "Add charge",
      statementMonth: "Statement month",
      amount: "Amount",
      categoryPlaceholder: "Category (optional)",
      categorySelectPlaceholder: "Select category (optional)",
      loadingCategories: "Loading categories...",
      notePlaceholder: "Note (optional)",
      saveCharge: "Log charge",
      saving: "Saving...",
      invalidCard: "Provide a name and positive limit.",
      invalidCharge: "Provide a month and a non-zero amount.",
      cardSaved: "Card saved.",
      chargeSaved: "Charge saved.",
    },
    pt: {
      sectionTitle: "Cartões de crédito",
      addCardTitle: "Adicionar cartão",
      cardName: "Nome do cartão",
      limit: "Limite",
      closeDay: "Fechamento",
      dueDay: "Vencimento",
      saveCard: "Salvar cartão",
      noCards: "Nenhum cartão cadastrado. Adicione um para acompanhar as faturas.",
      utilization: ({ spent, limit }) => `${spent} em aberto de ${limit}`,
      upcomingTitle: "Próximas faturas",
      upcomingEmpty: "Nenhuma despesa futura.",
      newCharge: "Adicionar gasto",
      statementMonth: "Mês da fatura",
      amount: "Valor",
      categoryPlaceholder: "Categoria (opcional)",
      categorySelectPlaceholder: "Selecione a categoria (opcional)",
      loadingCategories: "Carregando categorias...",
      notePlaceholder: "Observação (opcional)",
      saveCharge: "Registrar gasto",
      saving: "Salvando...",
      invalidCard: "Informe um nome e um limite válido.",
      invalidCharge: "Informe o mês e um valor diferente de zero.",
      cardSaved: "Cartão salvo.",
      chargeSaved: "Gasto registrado.",
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  const formatCurrency = (value) => {
    const symbols = {
      BRL: { prefix: "R$", locale: "pt-BR" },
      USD: { prefix: "$", locale: "en-US" },
    };
    const cfg = symbols[currency] ?? symbols.BRL;
    const formatter = new Intl.NumberFormat(cfg.locale, {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
    });
    return `${cfg.prefix} ${formatter.format(value)}`;
  };

  const root = dv.container.createEl("div", { cls: "cards-manager" });
  root.style.display = "flex";
  root.style.flexDirection = "column";
  root.style.gap = "1.25rem";

  const today = window.moment();
  const currentMonthKey = today.format("YYYY-MM");

  const render = async () => {
    root.innerHTML = "";
    const data = await toolkit.loadCardsData(dv);
    const cards = data.cards.sort((a, b) => a.name.localeCompare(b.name));

    const title = root.createEl("h2", { text: t("sectionTitle") });
    title.style.margin = "0";

    const addCardSection = root.createEl("section");
    addCardSection.style.border = "1px solid var(--background-modifier-border)";
    addCardSection.style.borderRadius = "10px";
    addCardSection.style.padding = "1rem";

    addCardSection.createEl("h3", { text: t("addCardTitle") }).style.marginTop = "0";

    const cardForm = addCardSection.createEl("form");
    cardForm.style.display = "grid";
    cardForm.style.gridTemplateColumns = "repeat(auto-fit, minmax(180px, 1fr))";
    cardForm.style.gap = "0.5rem";

    const nameInput = cardForm.createEl("input", {
      attr: { type: "text", placeholder: t("cardName"), required: "true" },
    });
    const limitInput = cardForm.createEl("input", {
      attr: { type: "number", min: "0", step: "0.01", placeholder: t("limit"), required: "true" },
    });
    const closeInput = cardForm.createEl("input", {
      attr: { type: "number", min: "1", max: "31", placeholder: t("closeDay") },
    });
    const dueInput = cardForm.createEl("input", {
      attr: { type: "number", min: "1", max: "31", placeholder: t("dueDay") },
    });
    const cardButton = cardForm.createEl("button", { text: t("saveCard") });
    cardButton.type = "submit";
    cardButton.style.cursor = "pointer";

    const createCardFile = async ({ name, limit, closeDay, dueDay }) => {
      await toolkit.ensureFolder(toolkit.cardsFolder);
      const baseName = toolkit.sanitizeFileName(name);
      const safeName = baseName || `card-${Date.now()}`;
      const ext = ".md";
      let targetPath = `${toolkit.cardsFolder}/${safeName}${ext}`;
      let suffix = 1;
      while (app.vault.getAbstractFileByPath(targetPath)) {
        targetPath = `${toolkit.cardsFolder}/${safeName}-${suffix}${ext}`;
        suffix += 1;
      }
      const escapedName = name.replace(/"/g, '\\"');
      const content = `---
cardName: "${escapedName}"
limit: ${limit}
closeDay: ${closeDay}
dueDay: ${dueDay}
---
# ${name}

> [!info]
> Charges are managed from [[Finance]].

## Charges
`;
      await app.vault.create(targetPath, content);
    };

    cardForm.onsubmit = async (event) => {
      event.preventDefault();
      const name = nameInput.value.trim();
      const limit = Number((limitInput.value || "").replace(",", "."));
      const closeDay = Number(closeInput.value || 5);
      const dueDay = Number(dueInput.value || closeDay + 5);
      if (!name || !isFinite(limit) || limit <= 0) {
        new Notice(t("invalidCard"));
        return;
      }
      cardButton.disabled = true;
      const prev = cardButton.textContent;
      cardButton.textContent = t("saving");
      try {
        await createCardFile({ name, limit, closeDay, dueDay });
        new Notice(t("cardSaved"));
        nameInput.value = "";
        limitInput.value = "";
        closeInput.value = "";
        dueInput.value = "";
        await render();
      } catch (error) {
        console.error(error);
        new Notice("Could not save card.");
      } finally {
        cardButton.disabled = false;
        cardButton.textContent = prev;
      }
    };

    if (cards.length === 0) {
      root.createEl("p", { text: t("noCards") });
      return;
    }

    const cardsGrid = root.createEl("div");
    cardsGrid.style.display = "grid";
    cardsGrid.style.gridTemplateColumns = "repeat(auto-fit, minmax(260px, 1fr))";
    cardsGrid.style.gap = "1rem";

    const maybeInsertChargeIntoMonth = async (card, charge) => {
      const monthMoment = window.moment(charge.monthKey, "YYYY-MM", true);
      if (!monthMoment?.isValid()) return;
      const year = monthMoment.format("YYYY");
      const monthName = monthMoment.format("MMMM");
      const monthPath = `finance/${year}/${monthName}.md`;
      const file = app.vault.getAbstractFileByPath(monthPath);
      if (!file) return;
      let content;
      try {
        content = await dv.io.load(monthPath);
      } catch {
        return;
      }
      if (content.includes(`#${charge.tag}`)) return;
      const noteTag = charge.note ? ` #${toolkit.slugify(charge.note)}` : "";
      const line = `- ${charge.category || card.name}:: ${charge.amount.toFixed(2)} #card-bill #${charge.tag}${noteTag}`;
      await toolkit.insertLine(monthPath, "Expenses", line);
    };

    const appendCharge = async (card, payload) => {
      const monthMoment = window.moment(payload.month, "YYYY-MM", true);
      if (!monthMoment?.isValid()) {
        throw new Error("Invalid month");
      }
      const amount = Number(payload.amount);
      if (!isFinite(amount) || amount === 0) {
        throw new Error("Invalid amount");
      }
      const id = `${card.slug}-${Date.now().toString(36)}`;
      const lineParts = [
        `id:: ${id}`,
        `month:: ${payload.month}`,
        `amount:: ${amount.toFixed(2)}`,
        `category:: ${payload.category || card.name}`,
      ];
      if (payload.note) lineParts.push(`note:: ${payload.note}`);
      lineParts.push(`created:: ${window.moment().toISOString()}`);
      await toolkit.insertLine(card.path, "Charges", `- ${lineParts.join(" ")}`);
      await maybeInsertChargeIntoMonth(card, {
        id,
        monthKey: payload.month,
        amount,
        category: payload.category || card.name,
        note: payload.note || "",
        tag: `card-${card.slug}-${toolkit.slugify(id)}`,
      });
    };

    const formatMonthLabel = (monthKey) => {
      const m = window.moment(monthKey, "YYYY-MM", true);
      return m.isValid() ? m.format(language === "pt" ? "MMM YYYY" : "MMM YYYY") : monthKey;
    };

    cards.forEach((card) => {
      const cardBox = cardsGrid.createEl("div");
      cardBox.style.border = "1px solid var(--background-modifier-border)";
      cardBox.style.borderRadius = "10px";
      cardBox.style.padding = "1rem";
      cardBox.style.display = "flex";
      cardBox.style.flexDirection = "column";
      cardBox.style.gap = "0.75rem";

      const header = cardBox.createEl("div");
      header.style.display = "flex";
      header.style.justifyContent = "space-between";
      header.style.alignItems = "baseline";

      const nameEl = header.createEl("h3", { text: card.name });
      nameEl.style.margin = "0";
      header.createEl("span", { text: formatCurrency(card.limit) });

      const isUnpaidCharge = (charge) => {
        const monthMoment = window.moment(charge.monthKey, "YYYY-MM", true);
        if (!monthMoment?.isValid()) return true;
        return monthMoment.isSameOrAfter(today, "month");
      };

      const outstandingSpent = card.charges
        .filter(isUnpaidCharge)
        .reduce((sum, charge) => sum + charge.amount, 0);
      const spentLabel = cardBox.createEl("span", {
        text: t("utilization", {
          spent: formatCurrency(outstandingSpent),
          limit: formatCurrency(card.limit),
        }),
      });
      spentLabel.style.fontSize = "0.85rem";
      spentLabel.style.color = "var(--text-muted)";

      const track = cardBox.createEl("div");
      track.style.width = "100%";
      track.style.height = "8px";
      track.style.borderRadius = "999px";
      track.style.background = "var(--background-modifier-border)";

      const fill = track.createEl("div");
      fill.style.height = "100%";
      fill.style.borderRadius = "999px";
      fill.style.background = "var(--interactive-accent)";
      const percent =
        card.limit > 0 ? Math.min(100, Math.round((outstandingSpent / card.limit) * 100)) : 0;
      fill.style.width = `${percent}%`;

      const upcomingWrapper = cardBox.createEl("div");
      const upcomingTitle = upcomingWrapper.createEl("strong", { text: t("upcomingTitle") });
      upcomingTitle.style.display = "block";
      upcomingTitle.style.marginBottom = "0.25rem";
      const futureMap = new Map();
      card.charges.forEach((charge) => {
        if (!charge.monthKey) return;
        const total = futureMap.get(charge.monthKey) || 0;
        futureMap.set(charge.monthKey, total + charge.amount);
      });
      const upcoming = [...futureMap.entries()]
        .sort((a, b) => a[0].localeCompare(b[0]))
        .slice(0, 5);
      if (upcoming.length === 0) {
        upcomingWrapper.createEl("span", { text: t("upcomingEmpty") }).style.fontSize = "0.85rem";
      } else {
        const list = upcomingWrapper.createEl("ul");
        list.style.margin = "0";
        list.style.paddingLeft = "1.1rem";
        upcoming.forEach(([monthKey, total]) => {
          const li = list.createEl("li");
          li.textContent = `${formatMonthLabel(monthKey)} • ${formatCurrency(total)}`;
        });
      }

      const chargeForm = cardBox.createEl("form");
      chargeForm.style.display = "flex";
      chargeForm.style.flexDirection = "column";
      chargeForm.style.gap = "0.35rem";

      chargeForm.createEl("strong", { text: t("newCharge") });

      const monthInput = chargeForm.createEl("input", {
        attr: { type: "month", value: currentMonthKey, placeholder: t("statementMonth") },
      });
      const amountInput = chargeForm.createEl("input", {
        attr: { type: "number", step: "0.01", placeholder: t("amount"), required: "true" },
      });
      const categoryField = chargeForm.createEl("div");
      categoryField.style.display = "flex";
      categoryField.style.flexDirection = "column";
      categoryField.style.gap = "0.25rem";

      const categorySelect = categoryField.createEl("select");
      categorySelect.style.padding = "0.35rem 0.5rem";
      categorySelect.disabled = true;
      categorySelect.createEl("option", { text: t("loadingCategories"), value: "" }).selected = true;

      let placeholderOption;
      let getCategoryValue = () => "";
      let resetCategoryField = () => {};

      const enableCategorySelect = (options) => {
        categorySelect.innerHTML = "";
        placeholderOption = categorySelect.createEl("option", {
          text: t("categorySelectPlaceholder"),
          value: "",
        });
        placeholderOption.selected = true;
        options.forEach((cat) => categorySelect.createEl("option", { text: cat, value: cat }));
        categorySelect.disabled = false;
        getCategoryValue = () => categorySelect.value || "";
        resetCategoryField = () => {
          categorySelect.value = "";
          placeholderOption.selected = true;
        };
      };

      const fallbackToInput = () => {
        categorySelect.remove();
        const fallbackInput = categoryField.createEl("input", {
          attr: { type: "text", placeholder: t("categoryPlaceholder") },
        });
        getCategoryValue = () => fallbackInput.value.trim();
        resetCategoryField = () => {
          fallbackInput.value = "";
        };
      };

      (async () => {
        try {
          const categories = (await toolkit.loadCategories?.("expenseCategories")) ?? [];
          if (categories.length) {
            enableCategorySelect(categories);
          } else {
            fallbackToInput();
          }
        } catch (error) {
          console.warn("Could not load categories for credit card charges.", error);
          fallbackToInput();
        }
      })();

      const noteInput = chargeForm.createEl("input", {
        attr: { type: "text", placeholder: t("notePlaceholder") },
      });
      const submitCharge = chargeForm.createEl("button", { text: t("saveCharge") });
      submitCharge.type = "submit";
      submitCharge.style.cursor = "pointer";

      chargeForm.onsubmit = async (event) => {
        event.preventDefault();
        const monthValue = monthInput.value || currentMonthKey;
        const amountValue = Number((amountInput.value || "").replace(",", "."));
        if (!monthValue || !isFinite(amountValue) || amountValue === 0) {
          new Notice(t("invalidCharge"));
          return;
        }
        submitCharge.disabled = true;
        const prevLabel = submitCharge.textContent;
        submitCharge.textContent = t("saving");
        try {
          const categoryValue = getCategoryValue();
          await appendCharge(card, {
            month: monthValue,
            amount: amountValue,
            category: categoryValue,
            note: noteInput.value.trim(),
          });
          new Notice(t("chargeSaved"));
          amountInput.value = "";
          resetCategoryField();
          noteInput.value = "";
          await render();
        } catch (error) {
          console.error(error);
          new Notice("Could not add charge.");
        } finally {
          submitCharge.disabled = false;
          submitCharge.textContent = prevLabel;
        }
      };
    });
  };

  await render();
};

run();
```

## Recurring expenses

```dataviewjs
const run = async () => {
  const toolkit = window.financeUtils;
  if (!toolkit) {
    dv.paragraph("Reload this dashboard to initialize recurring expenses.");
    return;
  }

  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();
  const currency = (settingsPage.currency || "BRL").toUpperCase();

  const translations = {
    en: {
      sectionTitle: "Recurring expenses",
      addRecurrence: "Add recurrence",
      titleLabel: "Title",
      categoryLabel: "Category",
      valueLabel: "Value",
      startLabel: "Start month",
      noteLabel: "Note (optional)",
      saveButton: "Save recurrence",
      saving: "Saving...",
      noItems: "No recurrences yet.",
      statusActive: "Active",
      statusPaused: "Paused",
      pause: "Pause",
      resume: "Resume",
      invalid: "Provide a title and value.",
      saved: "Recurrence saved.",
      updated: "Recurrence updated.",
    },
    pt: {
      sectionTitle: "Despesas recorrentes",
      addRecurrence: "Adicionar recorrência",
      titleLabel: "Título",
      categoryLabel: "Categoria",
      valueLabel: "Valor",
      startLabel: "Mês inicial",
      noteLabel: "Observação (opcional)",
      saveButton: "Salvar recorrência",
      saving: "Salvando...",
      noItems: "Nenhuma recorrência ainda.",
      statusActive: "Ativa",
      statusPaused: "Pausada",
      pause: "Pausar",
      resume: "Retomar",
      invalid: "Informe um título e um valor válido.",
      saved: "Recorrência salva.",
      updated: "Recorrência atualizada.",
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  const formatCurrency = (value) => {
    const symbols = {
      BRL: { prefix: "R$", locale: "pt-BR" },
      USD: { prefix: "$", locale: "en-US" },
    };
    const cfg = symbols[currency] ?? symbols.BRL;
    const formatter = new Intl.NumberFormat(cfg.locale, {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
    });
    return `${cfg.prefix} ${formatter.format(value)}`;
  };

  const serializeEntries = (entries) =>
    entries.map((entry) => ({
      id: entry.id,
      title: entry.title,
      category: entry.category,
      value: entry.value,
      start: entry.start || entry.startMoment?.format("YYYY-MM") || "",
      end: entry.end || entry.endMoment?.format("YYYY-MM") || "",
      active: entry.active !== false,
      note: entry.note || "",
    }));

  const root = dv.container.createEl("div", { cls: "recurrence-manager" });
  root.style.display = "flex";
  root.style.flexDirection = "column";
  root.style.gap = "1rem";

  const render = async () => {
    root.innerHTML = "";
    const entries = await toolkit.loadRecurrences(dv);

    const title = root.createEl("h2", { text: t("sectionTitle") });
    title.style.margin = "0";

    const formSection = root.createEl("section");
    formSection.style.border = "1px solid var(--background-modifier-border)";
    formSection.style.borderRadius = "10px";
    formSection.style.padding = "1rem";

    formSection.createEl("h3", { text: t("addRecurrence") }).style.marginTop = "0";

    const form = formSection.createEl("form");
    form.style.display = "grid";
    form.style.gridTemplateColumns = "repeat(auto-fit, minmax(180px, 1fr))";
    form.style.gap = "0.5rem";

    const titleInput = form.createEl("input", {
      attr: { type: "text", placeholder: t("titleLabel"), required: "true" },
    });
    const categoryInput = form.createEl("input", {
      attr: { type: "text", placeholder: t("categoryLabel"), value: "recurring" },
    });
    const valueInput = form.createEl("input", {
      attr: { type: "number", step: "0.01", placeholder: t("valueLabel"), required: "true" },
    });
    const startInput = form.createEl("input", {
      attr: { type: "month", placeholder: t("startLabel"), value: window.moment().format("YYYY-MM") },
    });
    const noteInput = form.createEl("input", {
      attr: { type: "text", placeholder: t("noteLabel") },
    });
    const saveButton = form.createEl("button", { text: t("saveButton") });
    saveButton.type = "submit";
    saveButton.style.cursor = "pointer";

    form.onsubmit = async (event) => {
      event.preventDefault();
      const titleValue = titleInput.value.trim();
      const categoryValue = categoryInput.value.trim() || "recurring";
      const valueNumber = Number((valueInput.value || "").replace(",", "."));
      const startValue = startInput.value || window.moment().format("YYYY-MM");
      if (!titleValue || !isFinite(valueNumber) || valueNumber === 0) {
        new Notice(t("invalid"));
        return;
      }
      saveButton.disabled = true;
      const prev = saveButton.textContent;
      saveButton.textContent = t("saving");
      try {
        entries.push({
          id: `rec-${Date.now().toString(36)}`,
          title: titleValue,
          category: categoryValue,
          value: valueNumber,
          start: startValue,
          end: "",
          active: true,
          note: noteInput.value.trim(),
        });
        await toolkit.saveRecurrences(serializeEntries(entries));
        new Notice(t("saved"));
        titleInput.value = "";
        valueInput.value = "";
        noteInput.value = "";
        await render();
      } catch (error) {
        console.error(error);
        new Notice("Could not save recurrence.");
      } finally {
        saveButton.disabled = false;
        saveButton.textContent = prev;
      }
    };

    if (entries.length === 0) {
      root.createEl("p", { text: t("noItems") });
      return;
    }

    const list = root.createEl("div");
    list.style.display = "grid";
    list.style.gridTemplateColumns = "repeat(auto-fit, minmax(240px, 1fr))";
    list.style.gap = "1rem";

    entries
      .sort((a, b) => a.title.localeCompare(b.title))
      .forEach((entry) => {
        const card = list.createEl("div");
        card.style.border = "1px solid var(--background-modifier-border)";
        card.style.borderRadius = "8px";
        card.style.padding = "0.85rem";
        card.style.display = "flex";
        card.style.flexDirection = "column";
        card.style.gap = "0.4rem";

        const heading = card.createEl("div");
        heading.style.display = "flex";
        heading.style.justifyContent = "space-between";
        heading.style.alignItems = "baseline";

        const titleEl = heading.createEl("strong", { text: entry.title });
        const statusEl = heading.createEl("span", {
          text: entry.active ? t("statusActive") : t("statusPaused"),
        });
        statusEl.style.fontSize = "0.8rem";
        statusEl.style.color = entry.active ? "var(--text-accent)" : "var(--text-muted)";

        card.createEl("span", { text: `${entry.category} • ${formatCurrency(entry.value)}` });
        card.createEl("span", {
          text: `${t("startLabel")}: ${entry.start || window.moment().format("YYYY-MM")}`,
        }).style.fontSize = "0.85rem";

        if (entry.note) {
          const noteEl = card.createEl("span", { text: entry.note });
          noteEl.style.fontSize = "0.85rem";
          noteEl.style.color = "var(--text-muted)";
        }

        const toggle = card.createEl("button", {
          text: entry.active ? t("pause") : t("resume"),
        });
        toggle.style.cursor = "pointer";
        toggle.onclick = async () => {
          entry.active = !entry.active;
          try {
            await toolkit.saveRecurrences(serializeEntries(entries));
            new Notice(t("updated"));
            await render();
          } catch (error) {
            console.error(error);
            new Notice("Could not update recurrence.");
          }
        };
      });
  };

  await render();
};

run();
```


## Summary
```dataviewjs
const run = async () => {
  const pages = dv.pages('"finance"').where(p => {
    const parts = p.file.folder.split('/');
    return parts.length === 2 && parts[0] === 'finance' && !isNaN(parts[1]);
  });

  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();
  const currency = (settingsPage.currency || "BRL").toUpperCase();

  const translations = {
    en: {
      missingMonth: "❌ Create `finance/2025/November.md` with `- cat:: value`",
      yearLabel: "Year:",
      openCurrentMonth: ({ month }) => `Open ${month}`,
      noDataYear: "No financial data for this year.",
      summaryTitle: ({ year }) => `Summary ${year}`,
      trendTitle: ({ year }) => `Trend ${year}`,
      financesTitle: ({ year }) => `Finances ${year}`,
      monthHeader: "Month",
      expensesLabel: "Expenses",
      incomeLabel: "Income",
      balanceLabel: "Balance",
      noEntries: "No entries yet.",
      categoryLabel: "Category",
      selectCategory: "Select category",
      amountPlaceholder: "Amount",
      tagPlaceholder: "Optional tag (e.g., rent)",
      addIncome: "Add income",
      addExpenses: "Add expense",
      saving: "Saving...",
      invalidEntry: "Select a category and enter a non-zero amount.",
      entryAdded: "Entry added! Reload to see the update.",
      couldNotSave: ({ reason }) => `Could not save entry (${reason}).`,
      openMonth: "Open month",
    },
    pt: {
      missingMonth: "❌ Crie `finance/2025/November.md` com linhas `- cat:: valor`",
      yearLabel: "Ano:",
      openCurrentMonth: ({ month }) => `Abrir ${month}`,
      noDataYear: "Nenhum dado financeiro para este ano.",
      summaryTitle: ({ year }) => `Resumo ${year}`,
      trendTitle: ({ year }) => `Tendência ${year}`,
      financesTitle: ({ year }) => `Finanças ${year}`,
      monthHeader: "Mês",
      expensesLabel: "Despesas",
      incomeLabel: "Receitas",
      balanceLabel: "Saldo",
      noEntries: "Sem lançamentos.",
      categoryLabel: "Categoria",
      selectCategory: "Selecione a categoria",
      amountPlaceholder: "Valor",
      tagPlaceholder: "Tag opcional (ex.: aluguel)",
      addIncome: "Adicionar receita",
      addExpenses: "Adicionar despesa",
      saving: "Salvando...",
      invalidEntry: "Escolha uma categoria e informe um valor diferente de zero.",
      entryAdded: "Lançamento adicionado! Recarregue para ver a atualização.",
      couldNotSave: ({ reason }) => `Não foi possível salvar (${reason}).`,
      openMonth: "Abrir mês",
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  if (pages.length === 0) {
    dv.paragraph(t("missingMonth"));
    return;
  }

  const currencySymbols = {
    BRL: { prefix: "R$", format: (value) => Number(value || 0).toFixed(2).replace('.', ',') },
    USD: { prefix: "$", format: (value) => Number(value || 0).toFixed(2) },
  };
  const currencyConfig = currencySymbols[currency] || currencySymbols.BRL;
  const formatCurrency = (value) => `${currencyConfig.prefix} ${currencyConfig.format(value)}`;
  const makeTagSlug = (value) =>
    value
      .toLowerCase()
      .normalize('NFD')
      .replace(/[\u0300-\u036f]/g, '')
      .replace(/[^a-z0-9-]+/g, '-')
      .replace(/-+/g, '-')
      .replace(/^-+|-+$/g, '');

  const loadCategories =
    window.financeUtils?.loadCategories ||
    (() => {
      const cache = {};
      return async (type) => {
        if (cache[type]) return cache[type];
        try {
          const file = app.vault.getAbstractFileByPath('config/consts.md');
          if (!file) return (cache[type] = []);
          const content = await app.vault.read(file);
          const match = content.match(new RegExp(`##\\s*${type}[\\s\\S]*?(?=\\n##|$)`, 'i'));
          if (!match) return (cache[type] = []);
          const items = match[0]
            .split('\n')
            .map(line => line.trim())
            .filter(line => line.startsWith('-'))
            .map(line => line.replace(/^-\s*/, '').trim())
            .filter(Boolean);
          return (cache[type] = items);
        } catch {
          return (cache[type] = []);
        }
      };
    })();

  const parseSection = (content, heading) => {
    const after = content.split(new RegExp(`##\\s*${heading}`, 'i'))[1];
    if (!after) return { total: 0, items: [] };
    const section = after.split(/\n##\s+/)[0];
    const items = [];
    let total = 0;
    section.split('\n').forEach(rawLine => {
      const line = rawLine.trim();
      if (!line) return;
      const match = line.match(/^-?\s*([^:]+)::\s*([\d.,]+)(?:\s*#(.+))?/);
      if (!match) return;
      const [, catRaw, valueRaw, noteRaw] = match;
      const value = parseFloat(valueRaw.replace(',', '.'));
      if (isNaN(value)) return;
      total += value;
      items.push({
        category: catRaw.trim(),
        value,
        note: (noteRaw || '').trim()
      });
    });
    return { total, items };
  };

  const insertLine = async (path, heading, line) => {
    const file = app.vault.getAbstractFileByPath(path);
    if (!file) throw new Error('File not found');
    const content = await app.vault.read(file);
    const sectionRegex = new RegExp(`(##\\s*${heading}[^\\n]*\\n)([\\s\\S]*?)(?=\\n##\\s+|$)`, 'i');
    const match = sectionRegex.exec(content);
    const lineWithBreak = line.endsWith('\n') ? line : `${line}\n`;

    if (match) {
      const before = content.slice(0, match.index);
      const afterContent = content.slice(match.index + match[0].length);
      const body = match[2].trimEnd();
      const newBody = `${body ? body + '\n' : ''}${lineWithBreak}`;
      const updated = before + match[1] + newBody + afterContent;
      await app.vault.modify(file, updated);
    } else {
      const updated = `${content.trim()}\n\n## ${heading}\n${lineWithBreak}`;
      await app.vault.modify(file, updated);
    }
  };

  const years = [...new Set(pages.map(p => p.file.folder.split('/')[1]))].map(Number).sort((a, b) => b - a);
  const controls = dv.container.createEl('div');
  controls.style.display = 'flex';
  controls.style.gap = '0.75rem';
  controls.style.flexWrap = 'wrap';
  controls.style.alignItems = 'center';
  controls.style.margin = '0.5rem 0 1rem';

  const yearLabel = controls.createEl('label', { text: t('yearLabel') });
  yearLabel.style.fontWeight = '600';

  const yearSelect = controls.createEl('select');
  yearSelect.style.padding = '0.25rem 0.5rem';
  years.forEach(y => yearSelect.createEl('option', { text: y, value: y }));
  yearSelect.value = years[0];

  const today = window.moment();
  const currentYear = today.format('YYYY');
  const currentMonthName = today.format('MMMM');
  const currentPath = `finance/${currentYear}/${currentMonthName}.md`;

  const openMonthBtn = controls.createEl('button', { text: t('openCurrentMonth', { month: currentMonthName }) });
  openMonthBtn.className = 'mod-cta';
  openMonthBtn.style.cursor = 'pointer';
  openMonthBtn.onclick = () => app.workspace.openLinkText(currentPath, '', false);

  const dataContainer = dv.container.createEl('div');

  const loadChartJs = (() => {
    let promise;
    return () => {
      if (window.Chart) return Promise.resolve();
      if (!promise) {
        promise = new Promise((resolve, reject) => {
          const script = document.createElement('script');
          script.src = 'https://cdn.jsdelivr.net/npm/chart.js';
          script.onload = resolve;
          script.onerror = reject;
          document.head.appendChild(script);
        });
      }
      return promise;
    };
  })();

  const renderYear = async (year) => {
    dataContainer.innerHTML = '';
    const yearPages = pages.where(p => p.file.folder === `finance/${year}`);

    const monthEntries = [];
    for (const p of yearPages) {
      const month = p.file.name.replace('.md', '');
      const content = await dv.io.load(p.file.path);
      const expenses = parseSection(content, 'Expenses');
      const income = parseSection(content, 'Income');
      monthEntries.push({
        month,
        file: p.file,
        expenses,
        income
      });
    }

    if (monthEntries.length === 0) {
      dataContainer.createEl('p', { text: t('noDataYear') });
      return;
    }

    monthEntries.sort((a, b) => new Date(`${b.month} 1, ${year}`) - new Date(`${a.month} 1, ${year}`));
    const months = monthEntries.map(m => m.month);

    const summaryTable = dataContainer.createEl('div');
    summaryTable.createEl('h3', { text: t('summaryTitle', { year }) });
    const rows = monthEntries.map(entry => [
      entry.month,
      formatCurrency(entry.expenses.total),
      formatCurrency(entry.income.total),
      formatCurrency(entry.income.total - entry.expenses.total)
    ]);
    dv.table([t('monthHeader'), t('expensesLabel'), t('incomeLabel'), t('balanceLabel')], rows, summaryTable);

    const chartWrapper = dataContainer.createEl('div');
    chartWrapper.createEl('h3', { text: t('trendTitle', { year }) });
    const chartDiv = chartWrapper.createEl('div', { attr: { style: 'height:400px; margin:20px 0;' } });
    loadChartJs().then(() => {
      const canvas = document.createElement('canvas');
      chartDiv.appendChild(canvas);
      new Chart(canvas, {
        type: 'line',
        data: {
          labels: months,
          datasets: [
            {
              label: t('expensesLabel'),
              data: monthEntries.map(m => m.expenses.total),
              borderColor: '#FF6384',
              backgroundColor: '#FF638433',
              tension: 0.3
            },
            {
              label: t('incomeLabel'),
              data: monthEntries.map(m => m.income.total),
              borderColor: '#36A2EB',
              backgroundColor: '#36A2EB33',
              tension: 0.3
            },
            {
              label: t('balanceLabel'),
              data: monthEntries.map(m => m.income.total - m.expenses.total),
              borderColor: '#4BC0C0',
              backgroundColor: '#4BC0C033',
              tension: 0.3
            }
          ]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          interaction: { mode: 'index', intersect: false },
          plugins: {
            title: { display: true, text: t('financesTitle', { year }) },
            legend: { position: 'bottom' }
          },
          scales: {
            y: {
              beginAtZero: true,
              ticks: { callback: (value) => `${currencyConfig.prefix} ${value}` }
            }
          }
        }
      });
    });

    const monthsGrid = dataContainer.createEl('div');
    monthsGrid.style.display = 'flex';
    monthsGrid.style.flexDirection = 'column';
    monthsGrid.style.gap = '1rem';
    monthsGrid.style.marginTop = '1rem';

    const renderList = (items, parent) => {
      if (items.length === 0) {
        parent.createEl('p', { text: t('noEntries') }).style.color = 'var(--text-muted)';
        return;
      }
      const list = parent.createEl('ul');
      list.style.margin = '0';
      list.style.paddingLeft = '1.1rem';
      items.forEach(item => {
        const li = list.createEl('li');
        const note = item.note ? ` • #${item.note}` : '';
        li.textContent = `${item.category}: ${formatCurrency(item.value)}${note}`;
      });
    };

    const createEntryForm = (parent, heading, filePath) => {
      const form = parent.createEl('form');
      form.style.display = 'flex';
      form.style.flexDirection = 'column';
      form.style.gap = '0.35rem';
      form.style.marginTop = '0.5rem';

      const nameLabel = form.createEl('label', { text: t('categoryLabel') });
      nameLabel.style.fontWeight = '600';
      const nameInput = form.createEl('select');
      nameInput.required = true;
      const placeholder = nameInput.createEl('option', { text: t('selectCategory'), value: '' });
      placeholder.disabled = true;
      placeholder.selected = true;

      loadCategories(heading === 'Income' ? 'incomeCategories' : 'expenseCategories').then(list => {
        const targetList = Array.isArray(list) && list.length ? list : ['general'];
        if (targetList.length) {
          placeholder.selected = false;
        }
        targetList.forEach(cat => {
          nameInput.createEl('option', { text: cat, value: cat });
        });
      });
      const valueInput = form.createEl('input', { attr: { type: 'number', step: '0.01', placeholder: t('amountPlaceholder'), required: 'true' } });
      const noteInput = form.createEl('input', { attr: { type: 'text', placeholder: t('tagPlaceholder') } });
      const submitBtn = form.createEl('button', { text: heading === 'Income' ? t('addIncome') : t('addExpenses') });
      submitBtn.type = 'submit';
      submitBtn.style.cursor = 'pointer';

      form.onsubmit = async (event) => {
        event.preventDefault();
        const category = nameInput.value.trim();
        const value = parseFloat((valueInput.value || '').replace(',', '.'));
        if (!category || isNaN(value) || value === 0) {
          new Notice(t('invalidEntry'));
          return;
        }
        submitBtn.disabled = true;
        submitBtn.textContent = t('saving');

        const tag = noteInput.value.trim();
        const safeTag = tag ? makeTagSlug(tag) : '';
        const amountStr = Number(value).toFixed(2);
        const line = `- ${category}:: ${amountStr}${safeTag ? ` #${safeTag}` : ''}`;

        try {
          await insertLine(filePath, heading, line);
          new Notice(t('entryAdded'));
          nameInput.value = '';
          valueInput.value = '';
          noteInput.value = '';
        } catch (error) {
          console.error(error);
          const reason = error?.message || 'unknown error';
          new Notice(t('couldNotSave', { reason }));
        } finally {
          submitBtn.disabled = false;
          submitBtn.textContent = heading === 'Income' ? t('addIncome') : t('addExpenses');
        }
      };
    };

    monthEntries.forEach(entry => {
      const card = monthsGrid.createEl('div');
      card.style.border = '1px solid var(--background-modifier-border)';
      card.style.borderRadius = '10px';
      card.style.padding = '1rem';
      card.style.display = 'flex';
      card.style.flexDirection = 'column';
      card.style.gap = '0.75rem';

      const cardHeader = card.createEl('div');
      cardHeader.style.display = 'flex';
      cardHeader.style.justifyContent = 'space-between';
      cardHeader.style.alignItems = 'center';

      const title = cardHeader.createEl('h3', { text: entry.month });
      title.style.margin = '0';

      const openButton = cardHeader.createEl('button', { text: t('openMonth') });
      openButton.style.cursor = 'pointer';
      openButton.onclick = () => app.workspace.openLinkText(entry.file.path, '', false);

      const columns = card.createEl('div');
      columns.style.display = 'grid';
      columns.style.gridTemplateColumns = 'repeat(auto-fit, minmax(240px, 1fr))';
      columns.style.gap = '1rem';

      const buildColumn = (heading, data) => {
        const col = columns.createEl('div');
        const colHeader = col.createEl('div');
        colHeader.style.display = 'flex';
        colHeader.style.justifyContent = 'space-between';
        colHeader.style.alignItems = 'baseline';

        const headingLabel = heading === 'Income' ? t('incomeLabel') : t('expensesLabel');
        const colTitle = colHeader.createEl('h4', { text: headingLabel });
        colTitle.style.margin = '0';
        colHeader.createEl('span', { text: formatCurrency(data.total) });

        renderList(data.items, col);
        createEntryForm(col, heading, entry.file.path);
      };

      buildColumn('Income', entry.income);
      buildColumn('Expenses', entry.expenses);
    });
  };

  renderYear(years[0]);
  yearSelect.onchange = () => renderYear(yearSelect.value);
};

run();
```
