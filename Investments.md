# Investments

> [!info]
> Manage your investments and track monthly contributions.

```meta-bind-button
label: New investment
icon: plus
style: primary
class: ""
cssStyle: ""
backgroundImage: ""
tooltip: ""
id: ""
hidden: false
actions:
  - type: templaterCreateNote
    templateFile: templates/new investment.md
    folderPath: investments
    fileName: investment
    openNote: true
    openIfAlreadyExists: true

```

```dataviewjs
const run = async () => {
  const folder = "investments";
  const pages = dv
    .pages(`"${folder}"`)
    .where((p) => p.file.path.startsWith(`${folder}/`));

  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();
  const currency = (settingsPage.currency || "BRL").toUpperCase();

  const translations = {
    en: {
      noInvestments: "❌ No investments found. Create notes under `investments/`.",
      noMovements: "❌ No movements found.",
      chartTitle: "Contribution trend (last 12 months)",
      totalInvested: "Total invested:",
      lastContribution: "Last contribution:",
      onDate: ({ date }) => `on ${date}`,
      newTotalPlaceholder: ({ prefix }) => `New total value (${prefix})`,
      optionalTagPlaceholder: "Optional tag (e.g., bonus)",
      addContribution: "Add contribution",
      enterValue: "Enter the new total value.",
      valueMatches: "The value you entered already matches the current total.",
      saving: "Saving...",
      contributionAdded: "Contribution added! Reload the view to see the result.",
      couldNotSave: ({ reason }) => `Could not save the contribution (${reason}).`,
    },
    pt: {
      noInvestments: "❌ Nenhum investimento encontrado. Crie notas em `investments/`.",
      noMovements: "❌ Nenhum movimento registrado.",
      chartTitle: "Tendência de aportes (últimos 12 meses)",
      totalInvested: "Total investido:",
      lastContribution: "Último aporte:",
      onDate: ({ date }) => `em ${date}`,
      newTotalPlaceholder: ({ prefix }) => `Novo valor total (${prefix})`,
      optionalTagPlaceholder: "Tag opcional (ex.: bônus)",
      addContribution: "Adicionar aporte",
      enterValue: "Informe o novo valor total.",
      valueMatches: "O valor informado já corresponde ao total atual.",
      saving: "Salvando...",
      contributionAdded: "Aporte registrado! Recarregue para ver o resultado.",
      couldNotSave: ({ reason }) => `Não foi possível salvar o aporte (${reason}).`,
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  const currencySymbols = {
    BRL: {
      prefix: "R$",
      format: (value) => Number(value).toFixed(2).replace(".", ","),
    },
    USD: {
      prefix: "$",
      format: (value) => Number(value).toFixed(2),
    },
  };

  const currencyConfig = currencySymbols[currency] || currencySymbols.BRL;
  const formatCurrency = (value) => `${currencyConfig.prefix} ${currencyConfig.format(value)}`;
  const formatLedgerAmount = (value) => {
    if (!isFinite(value)) return "0";
    const fixed = Number(value).toFixed(2);
    return fixed.replace(/\.00$/, "");
  };

  if (pages.length === 0) {
    dv.paragraph(t("noInvestments"));
    return;
  }

  const today = window.moment();
  const todayStr = today.format("YYYY-MM-DD");

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

  const makeTagSlug = (value) =>
    value
      .toLowerCase()
      .normalize("NFD")
      .replace(/[\u0300-\u036f]/g, "")
      .replace(/[^a-z0-9-]+/g, "-")
      .replace(/-+/g, "-")
      .replace(/^-+|-+$/g, "");

  const extractSectionLines = (content) => {
    const sectionRegex = /##\s*Movements[^\n]*\n([\s\S]*?)(?=\n##\s+|$)/i;
    const match = sectionRegex.exec(content);
    const target = match ? match[1] : content;
    return target.split("\n");
  };

  const parseMovements = (content) => {
    return extractSectionLines(content)
      .map((line) => line.trim())
      .filter((line) => line.startsWith("-"))
      .map((line) => {
        const body = line.replace(/^-\s*/, "");
        const amountMatch = body.match(/^(-?[\d.,]+)/);
        if (!amountMatch) return null;
        const rawAmount = amountMatch[0];
        const amount = parseFloat(rawAmount.replace(",", "."));
        if (isNaN(amount)) return null;

        const tags = [...line.matchAll(/#([^\s#]+)/g)].map((m) => m[1]);
        const dateTag = tags.find((tag) => /^\d{4}-\d{2}-\d{2}$/.test(tag));
        const date = dateTag ? window.moment(dateTag, "YYYY-MM-DD", true) : null;

        return {
          amount,
          tags,
          date: date && date.isValid() ? date : null,
          raw: line,
          isInitial: tags.includes("initial"),
        };
      })
      .filter(Boolean);
  };

  const normalizePath = (rawPath) => {
    if (!rawPath) return null;
    let path = rawPath.trim();
    if (path.endsWith("/")) path = path.slice(0, -1);
    if (!path.endsWith(".md")) path = `${path}.md`;
    return path;
  };

  const ensureFile = (rawPath) => {
    const path = normalizePath(rawPath);
    if (!path) return null;
    const file = app.vault.getAbstractFileByPath(path);
    return file && typeof file === "object" && "extension" in file ? file : null;
  };

  const insertMovement = async (path, line) => {
    const tFile = ensureFile(path);
    if (!tFile) throw new Error("File not found");
    const content = await app.vault.read(tFile);
    const sectionRegex = /(##\s*Movements[^\n]*\n)([\s\S]*?)(?=\n##\s+|$)/i;
    const match = sectionRegex.exec(content);
    const lineWithBreak = line.endsWith("\n") ? line : `${line}\n`;

    if (match) {
      const before = content.slice(0, match.index);
      const after = content.slice(match.index + match[0].length);
      const body = match[2].trimEnd();
      const newBody = `${body ? body + "\n" : ""}${lineWithBreak}`;
      const updated = before + match[1] + newBody + after;
      await app.vault.modify(tFile, updated);
    } else {
      const nextContent = `${content.trim()}\n\n## Movements\n${lineWithBreak}`;
      await app.vault.modify(tFile, nextContent);
    }
  };

  const investments = [];
  for (const page of pages) {
    const content = await dv.io.load(page.file.path);
    const movements = parseMovements(content);
    const total = movements.reduce((sum, move) => sum + move.amount, 0);
    investments.push({ page, movements, total });
  }

  if (investments.length === 0) {
    dv.paragraph(t("noMovements"));
    return;
  }

  const chartWrapper = dv.container.createEl("div", { cls: "investment-chart" });
  chartWrapper.style.minHeight = "320px";
  chartWrapper.style.marginBottom = "1.5rem";
  const canvas = chartWrapper.createEl("canvas");

  const months = [];
  const labels = [];
  const startMonth = today.clone().startOf("month").subtract(11, "months");
  for (let i = 0; i < 12; i++) {
    const month = startMonth.clone().add(i, "months");
    months.push(month);
    labels.push(month.format("MMM YYYY"));
  }

  const datasets = investments.map((inv, idx) => {
    const palette = [
      "#36A2EB",
      "#FF6384",
      "#FFCE56",
      "#4BC0C0",
      "#9966FF",
      "#FF9F40",
      "#60A917",
      "#E64A19",
    ];

    const sorted = inv.movements
      .filter((m) => m.date)
      .sort((a, b) => a.date.valueOf() - b.date.valueOf());

    let idxPointer = 0;
    let running = 0;
    while (
      idxPointer < sorted.length &&
      sorted[idxPointer].date.isBefore(months[0], "month")
    ) {
      running += sorted[idxPointer].amount;
      idxPointer++;
    }

    const data = months.map((month) => {
      while (
        idxPointer < sorted.length &&
        (sorted[idxPointer].date.isSame(month, "month") ||
          sorted[idxPointer].date.isBefore(month, "month"))
      ) {
        running += sorted[idxPointer].amount;
        idxPointer++;
      }
      return Number(running.toFixed(2));
    });

    return {
      label: inv.page.file.name,
      data,
      borderColor: palette[idx % palette.length],
      backgroundColor: palette[idx % palette.length] + "40",
      tension: 0.2,
    };
  });

  loadChartJs().then(() => {
    const ctx = canvas.getContext("2d");
    new Chart(ctx, {
      type: "line",
      data: { labels, datasets },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        interaction: { mode: "index", intersect: false },
        stacked: false,
        plugins: {
          title: {
            display: true,
            text: t("chartTitle"),
          },
          legend: { position: "bottom" },
        },
        scales: {
          y: {
            ticks: {
              callback: (value) => `${currencyConfig.prefix} ${Number(value).toFixed(0)}`,
            },
          },
        },
      },
    });
  });

  const cardsWrapper = dv.container.createEl("div", { cls: "investment-cards" });
  cardsWrapper.style.display = "grid";
  cardsWrapper.style.gridTemplateColumns = "repeat(auto-fit, minmax(260px, 1fr))";
  cardsWrapper.style.gap = "1rem";

  investments.forEach((inv) => {
    const card = cardsWrapper.createEl("div", { cls: "investment-card" });
    card.style.border = "1px solid var(--background-modifier-border)";
    card.style.borderRadius = "8px";
    card.style.padding = "1rem";
    card.style.display = "flex";
    card.style.flexDirection = "column";
    card.style.gap = "0.75rem";

    card.createEl("h3", { text: inv.page.file.name });
    card.createEl("p", {
      text: `${t("totalInvested")} ${formatCurrency(inv.total)}`,
    });

    const lastMove = inv.movements
      .filter((m) => m.date)
      .sort((a, b) => b.date.valueOf() - a.date.valueOf())[0];
    if (lastMove) {
      card.createEl("p", {
        text: `${t("lastContribution")} ${formatCurrency(lastMove.amount)} ${t("onDate", { date: lastMove.date.format("DD/MM/YYYY") })}`,
      });
    }

    const list = card.createEl("ul");
    list.style.margin = "0";
    list.style.paddingLeft = "1.2rem";
    inv.movements
      .slice(-5)
      .reverse()
      .forEach((move) => {
        const li = list.createEl("li");
        const dateStr = move.date ? move.date.format("DD/MM/YYYY") : "n/a";
        li.textContent = `${formatCurrency(move.amount)} • ${dateStr}`;
      });

    const form = card.createEl("form");
    form.style.display = "flex";
    form.style.flexDirection = "column";
    form.style.gap = "0.35rem";

    const valueInput = form.createEl("input", {
      attr: {
        type: "number",
        step: "0.01",
        placeholder: t("newTotalPlaceholder", { prefix: currencyConfig.prefix }),
      },
    });
    const noteInput = form.createEl("input", {
      attr: { type: "text", placeholder: t("optionalTagPlaceholder") },
    });
    const submitBtn = form.createEl("button", {
      text: t("addContribution"),
      attr: { type: "submit" },
    });

    form.onsubmit = async (event) => {
      event.preventDefault();
      const parsedInput = parseFloat((valueInput.value || "").replace(",", "."));
      if (isNaN(parsedInput)) {
        new Notice(t("enterValue"));
        return;
      }

      const currentTotal = inv.movements.reduce((sum, move) => sum + move.amount, 0);
      const delta = parseFloat((parsedInput - currentTotal).toFixed(2));
      if (Math.abs(delta) < 0.00001) {
        new Notice(t("valueMatches"));
        return;
      }

      submitBtn.disabled = true;
      submitBtn.textContent = t("saving");

      const amountStr = formatLedgerAmount(delta);
      const tags = [`#${todayStr}`];
      const note = noteInput.value.trim();
      if (note) {
        const tag = makeTagSlug(note);
        if (tag) tags.push(`#${tag}`);
      }

      const line = `- ${amountStr} ${tags.join(" ")}`;

      try {
        await insertMovement(inv.page.file.path, line);
        new Notice(t("contributionAdded"));
        valueInput.value = "";
        noteInput.value = "";
      } catch (error) {
        console.error(error);
        const reason = error?.message || "unknown error";
        new Notice(t("couldNotSave", { reason }));
      } finally {
        submitBtn.disabled = false;
        submitBtn.textContent = t("addContribution");
      }
    };
  });
};

run();
```
