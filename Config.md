# Config

> [!info]
> Choose your preferred UI language. This setting only changes interface labels (not filenames or variables).

```dataviewjs
const run = async () => {
  const settingsPath = "config/settings.md";
  const settingsPage = dv.page("config/settings") ?? {};
  const currentLanguage = (settingsPage.language || "en").toLowerCase();
  const currentCurrency = (settingsPage.currency || "BRL").toUpperCase();

  const translations = {
    en: {
      label: "Language",
      currencyLabel: "Currency",
      currencySaving: "Saving currency...",
      currencyUpdated: "Currency updated! Reload notes to see new symbols.",
      currencySaveFailed: "Could not save currency preference.",
      saving: "Saving...",
      updated: "Updated! Reload notes to see the new language.",
      notFound: "Settings file not found.",
      saveFailed: "Could not save.",
      updateFailed: "Could not update the language.",
    },
    pt: {
      label: "Idioma",
      currencyLabel: "Moeda",
      currencySaving: "Salvando moeda...",
      currencyUpdated: "Moeda atualizada! Recarregue as notas para ver os novos símbolos.",
      currencySaveFailed: "Não foi possível salvar a preferência de moeda.",
      saving: "Salvando...",
      updated: "Atualizado! Recarregue as notas para ver o novo idioma.",
      notFound: "Arquivo de configurações não encontrado.",
      saveFailed: "Não foi possível salvar.",
      updateFailed: "Não foi possível atualizar o idioma.",
    },
  };

  const t = (key) => translations[currentLanguage]?.[key] ?? translations.en[key] ?? key;

  const languages = [
    { value: "en", label: "English" },
    { value: "pt", label: "Português" },
  ];

  const currencies = [
    { value: "BRL", label: "Real (R$)" },
    { value: "USD", label: "Dollar ($)" },
  ];

  const container = dv.container.createEl("div", { cls: "config-panel" });
  container.style.display = "flex";
  container.style.flexDirection = "column";
  container.style.gap = "0.5rem";
  container.style.maxWidth = "360px";

  const label = container.createEl("label", { text: t("label") });
  label.style.fontWeight = "600";

  const select = container.createEl("select");
  select.style.padding = "0.4rem";
  select.style.borderRadius = "6px";
  select.style.border = "1px solid var(--background-modifier-border)";

  languages.forEach((lang) => {
    const option = select.createEl("option", {
      text: lang.label,
      attr: { value: lang.value },
    });
    if (lang.value === currentLanguage) option.selected = true;
  });

  const status = container.createEl("span", { text: " " });
  status.style.fontSize = "0.85rem";
  status.style.color = "var(--text-muted)";

  const currencyLabel = container.createEl("label", { text: t("currencyLabel") });
  currencyLabel.style.fontWeight = "600";
  currencyLabel.style.marginTop = "0.75rem";

  const currencySelect = container.createEl("select");
  currencySelect.style.padding = "0.4rem";
  currencySelect.style.borderRadius = "6px";
  currencySelect.style.border = "1px solid var(--background-modifier-border)";

  currencies.forEach((curr) => {
    const option = currencySelect.createEl("option", {
      text: curr.label,
      attr: { value: curr.value },
    });
    if (curr.value === currentCurrency) option.selected = true;
  });

  const currencyStatus = container.createEl("span", { text: " " });
  currencyStatus.style.fontSize = "0.85rem";
  currencyStatus.style.color = "var(--text-muted)";

  const updateLanguage = async (value) => {
    const file = app.vault.getAbstractFileByPath(settingsPath);
    if (!file) {
      new Notice(t("notFound"));
      return;
    }
    status.textContent = t("saving");
    try {
      await app.fileManager.processFrontMatter(file, (fm) => {
        fm.language = value;
      });
      status.textContent = t("updated");
    } catch (error) {
      console.error(error);
      status.textContent = t("saveFailed");
      new Notice(t("updateFailed"));
    }
  };

  const updateCurrency = async (value) => {
    const file = app.vault.getAbstractFileByPath(settingsPath);
    if (!file) {
      new Notice(t("notFound"));
      return;
    }
    currencyStatus.textContent = t("currencySaving");
    try {
      await app.fileManager.processFrontMatter(file, (fm) => {
        fm.currency = value;
      });
      currencyStatus.textContent = t("currencyUpdated");
    } catch (error) {
      console.error(error);
      currencyStatus.textContent = t("currencySaveFailed");
      new Notice(t("currencySaveFailed"));
    }
  };

  select.onchange = async (event) => {
    const value = event.target.value;
    if (value === currentLanguage) return;
    await updateLanguage(value);
  };

  currencySelect.onchange = async (event) => {
    const value = event.target.value;
    if (value === currentCurrency) return;
    await updateCurrency(value);
  };
};

run();
```
