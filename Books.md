# Books

> [!info]
> Register books, track chapter progress, or log standalone chapters without progress bars.

## Books hub
```dataviewjs
const run = async () => {
  const booksFolder = "books";
  const statsPath = "profile/stats.md";
  const BOOK_XP = 20;
  const settingsPage = dv.page("config/settings") ?? {};
  const language = (settingsPage.language || "en").toLowerCase();

  const translations = {
    en: {
      registerBook: "Register a book",
      bookNamePlaceholder: "Book title",
      totalPlaceholder: "Total chapters",
      addBook: "Add book",
      booksList: "Books",
      noBooks: "No books yet.",
      readChapter: "Read chapter",
      saving: "Saving...",
      invalidBookName: "Provide a book name.",
      invalidChapters: "Provide a valid number of chapters.",
      bookExists: "This book already exists.",
      bookCreated: "Book created.",
      couldNotCreateBook: "Could not create the book.",
      progressLabel: ({ read, total }) => `${read}/${total} chapters`,
      lastChapterLabel: ({ chapter, when }) => `Last chapter ${chapter} · ${when}`,
      allChaptersDone: "All chapters read.",
      chapterLogged: ({ chapter }) => `Chapter ${chapter} logged.`,
      couldNotLogChapter: "Could not log the chapter.",
      standaloneTitle: "Standalone chapter",
      standaloneDescription: "Register a chapter without tracking progress.",
      standaloneBookPlaceholder: "Book name",
      standaloneChapterPlaceholder: "Chapter #",
      standaloneButton: "Register chapter",
      standaloneSaved: "Chapter saved.",
      standaloneError: "Could not save the chapter.",
      noStandalone: "No standalone chapters yet.",
      finishedIn: ({ when }) => `finished in: ${when}`,
    },
    pt: {
      registerBook: "Cadastrar livro",
      bookNamePlaceholder: "Nome do livro",
      totalPlaceholder: "Total de capítulos",
      addBook: "Adicionar livro",
      booksList: "Livros",
      noBooks: "Nenhum livro cadastrado.",
      readChapter: "Ler capítulo",
      saving: "Salvando...",
      invalidBookName: "Informe o nome do livro.",
      invalidChapters: "Informe um total válido de capítulos.",
      bookExists: "Este livro já existe.",
      bookCreated: "Livro cadastrado.",
      couldNotCreateBook: "Não foi possível criar o livro.",
      progressLabel: ({ read, total }) => `${read}/${total} capítulos`,
      lastChapterLabel: ({ chapter, when }) => `Último capítulo ${chapter} · ${when}`,
      allChaptersDone: "Todos os capítulos foram lidos.",
      chapterLogged: ({ chapter }) => `Capítulo ${chapter} registrado.`,
      couldNotLogChapter: "Não foi possível registrar o capítulo.",
      standaloneTitle: "Registrar capítulo avulso",
      standaloneDescription: "Informe o livro e o capítulo para registrar sem progresso.",
      standaloneBookPlaceholder: "Livro",
      standaloneChapterPlaceholder: "Capítulo",
      standaloneButton: "Registrar capítulo",
      standaloneSaved: "Capítulo registrado.",
      standaloneError: "Não foi possível salvar o capítulo.",
      noStandalone: "Nenhum capítulo avulso ainda.",
      finishedIn: ({ when }) => `finished in: ${when}`,
    },
  };

  const t = (key, data = {}) => {
    const value = translations[language]?.[key] ?? translations.en[key] ?? key;
    return typeof value === "function" ? value(data) : value;
  };

  const module = dv.container.createEl("div", { cls: "books-module" });
  module.style.display = "flex";
  module.style.flexDirection = "column";
  module.style.gap = "1.5rem";

  const ensureFolderExists = async () => {
    const folder = app.vault.getAbstractFileByPath(booksFolder);
    if (!folder) {
      try {
        await app.vault.createFolder(booksFolder);
      } catch (error) {
        if (!/exists/i.test(error?.message ?? "")) {
          console.warn("Could not create books folder.", error);
        }
      }
    }
  };

  await ensureFolderExists();

  const incrementBooksXp = async () => {
    const statsFile = app.vault.getAbstractFileByPath(statsPath);
    if (!statsFile) return;
    let content = await dv.io.load(statsPath);
    const xpRegex = /^(\s*-\s*xp:\s*)(\d+)/im;
    if (xpRegex.test(content)) {
      content = content.replace(xpRegex, (_, prefix, value) => {
        const current = Number(value) || 0;
        return `${prefix}${current + BOOK_XP}`;
      });
    } else {
      const suffix = content.endsWith("\n") || content.length === 0 ? "" : "\n";
      content = `${content}${suffix}- xp: ${BOOK_XP}\n`;
    }
    await app.vault.modify(statsFile, content);
  };

  const asString = (value = "") => `${value ?? ""}`;
  const removeDiacritics = (value = "") =>
    asString(value)
      .normalize("NFD")
      .replace(/[\u0300-\u036f]/g, "");
  const sanitize = (value = "") =>
    removeDiacritics(value)
      .replace(/[^a-zA-Z0-9\s-]/g, " ")
      .replace(/\s+/g, " ")
      .trim();
  const slugify = (value = "") =>
    sanitize(value)
      .toLowerCase()
      .replace(/[^a-z0-9\s-]/g, "")
      .replace(/\s+/g, "-")
      .replace(/-+/g, "-");
  const fileSafe = (value = "") => {
    const sanitized = sanitize(value);
    return sanitized ? sanitized.replace(/\s+/g, "_") : "book";
  };
  const escapeYaml = (value = "") =>
    asString(value)
      .replace(/\r?\n/g, " ")
      .replace(/\\/g, "\\\\")
      .replace(/"/g, '\\"');
  const parseChapterCount = (value) => {
    const num = Number(value);
    return Number.isFinite(num) ? Math.max(0, Math.round(num)) : 0;
  };
  const formatDateTime = (value) => {
    if (!value) return "-";
    const m = window.moment(value);
    return m.isValid() ? m.format("YYYY-MM-DD HH:mm") : value;
  };
  const extractFields = (line = "") => {
    const fields = {};
    const regex = /([a-zA-Z]+)::\s*([^#\n]+?)(?=\s+[a-zA-Z]+::|$)/g;
    let match;
    while ((match = regex.exec(line)) !== null) {
      fields[match[1].toLowerCase()] = match[2].trim();
    }
    return fields;
  };
  const parseChaptersSection = (content = "") => {
    const sectionRegex = /##\s*Chapters[^\n]*\n([\s\S]*?)(?=\n##\s+|$)/i;
    const match = sectionRegex.exec(content);
    const body = match ? match[1] : "";
    return body
      .split("\n")
      .map((line) => line.trim())
      .filter((line) => line.startsWith("-"))
      .map((line) => {
        const fields = extractFields(line);
        const chapterNumber = parseChapterCount(fields.chapter);
        const finished = fields.finished ?? fields.date ?? null;
        if (!chapterNumber && !finished) return null;
        return {
          chapter: chapterNumber,
          finished,
        };
      })
      .filter(Boolean);
  };
  const loadBooks = () => {
    const pages = dv
      .pages(`"${booksFolder}"`)
      .where((page) => (page.booktype ?? page.bookType ?? page.file?.frontmatter?.bookType) === "book");
    return pages
      .map((page) => {
        const name =
          page.bookname ?? page.bookName ?? page.file?.frontmatter?.bookName ?? page.file?.name ?? "Book";
        return {
          name,
          totalChapters: parseChapterCount(page.totalchapters ?? page.totalChapters),
          path: page.file?.path,
          slug: slugify(name),
        };
      })
      .array()
      .sort((a, b) => a.name.localeCompare(b.name));
  };
  const loadStandaloneEntries = () => {
    const pages = dv
      .pages(`"${booksFolder}"`)
      .where((page) => (page.booktype ?? page.bookType ?? page.file?.frontmatter?.bookType) === "entry");
    return pages
      .map((page) => {
        const bookName =
          page.bookname ?? page.bookName ?? page.file?.frontmatter?.bookName ?? page.file?.name ?? "Book";
        const chapterNumber = parseChapterCount(page.chapternumber ?? page.chapterNumber);
        const finishedAt = page.finishedat ?? page.finishedAt ?? page.file?.frontmatter?.finishedAt ?? null;
        return {
          bookName,
          chapterNumber,
          finishedAt,
          path: page.file?.path,
          ms: finishedAt ? new Date(finishedAt).getTime() : 0,
        };
      })
      .array()
      .sort((a, b) => (b.ms || 0) - (a.ms || 0));
  };

  let books = loadBooks();
  let standaloneEntries = loadStandaloneEntries();
  const bookCache = new Map();

  const hydrateBook = async (book) => {
    if (!book?.path) return;
    try {
      const content = await dv.io.load(book.path);
      const chapters = parseChaptersSection(content);
      book.chapters = chapters;
      book.readChapters = chapters.length;
      bookCache.set(book.path, { content, chapters });
    } catch (error) {
      console.warn("Could not load book note:", book.path, error);
      book.chapters = [];
      book.readChapters = 0;
      bookCache.set(book.path, { content: "", chapters: [] });
    }
  };

  await Promise.all(books.map(hydrateBook));

  const formsSection = module.createEl("section");
  formsSection.style.display = "flex";
  formsSection.style.flexDirection = "column";
  formsSection.style.gap = "0.75rem";

  const bookFormTitle = formsSection.createEl("h3", { text: t("registerBook") });
  bookFormTitle.style.marginBottom = "0";

  const bookForm = formsSection.createEl("form");
  bookForm.style.display = "flex";
  bookForm.style.flexWrap = "wrap";
  bookForm.style.gap = "0.5rem";

  const bookNameInput = bookForm.createEl("input", {
    attr: { type: "text", placeholder: t("bookNamePlaceholder") },
  });
  bookNameInput.style.flex = "1";

  const bookChaptersInput = bookForm.createEl("input", {
    attr: { type: "number", min: "1", step: "1", placeholder: t("totalPlaceholder") },
  });
  bookChaptersInput.style.width = "160px";

  const bookFormButton = bookForm.createEl("button", { text: t("addBook") });
  bookFormButton.type = "submit";

  const standaloneTitle = formsSection.createEl("h3", { text: t("standaloneTitle") });
  standaloneTitle.style.marginBottom = "0";
  const standaloneDescription = formsSection.createEl("p", { text: t("standaloneDescription") });
  standaloneDescription.style.marginTop = "-0.35rem";
  standaloneDescription.style.fontSize = "0.9rem";
  standaloneDescription.style.color = "var(--text-muted, #888)";

  const standaloneForm = formsSection.createEl("form");
  standaloneForm.style.display = "flex";
  standaloneForm.style.flexWrap = "wrap";
  standaloneForm.style.gap = "0.5rem";

  const standaloneBookInput = standaloneForm.createEl("input", {
    attr: { type: "text", placeholder: t("standaloneBookPlaceholder") },
  });
  standaloneBookInput.style.flex = "1";

  const standaloneChapterInput = standaloneForm.createEl("input", {
    attr: { type: "number", min: "1", step: "1", placeholder: t("standaloneChapterPlaceholder") },
  });
  standaloneChapterInput.style.width = "140px";

  const standaloneButton = standaloneForm.createEl("button", { text: t("standaloneButton") });
  standaloneButton.type = "submit";

  const listsSection = module.createEl("section");
  listsSection.style.display = "flex";
  listsSection.style.flexDirection = "column";
  listsSection.style.gap = "1rem";

  const booksHeader = listsSection.createEl("h3", { text: t("booksList") });
  booksHeader.style.marginBottom = "0";
  const booksListContainer = listsSection.createEl("div", { cls: "books-list" });
  booksListContainer.style.display = "flex";
  booksListContainer.style.flexDirection = "column";
  booksListContainer.style.gap = "0.75rem";

  const standaloneListHeader = listsSection.createEl("h3", { text: t("standaloneTitle") });
  standaloneListHeader.style.marginBottom = "0";
  const standaloneListContainer = listsSection.createEl("div", { cls: "books-standalone" });
  standaloneListContainer.style.display = "flex";
  standaloneListContainer.style.flexDirection = "column";
  standaloneListContainer.style.gap = "0.5rem";

  const renderBooksList = () => {
    booksListContainer.innerHTML = "";
    if (books.length === 0) {
      booksListContainer.createEl("p", { text: t("noBooks") });
      return;
    }

    books.forEach((book) => {
      const read = book.readChapters ?? 0;
      const total = Math.max(book.totalChapters || 0, 0);
      const progress = total > 0 ? Math.min(1, read / total) : 0;
      const percent = Math.round(progress * 100);
      const lastChapter = book.chapters?.[book.chapters.length - 1] ?? null;

      const card = booksListContainer.createEl("div", { cls: "book-card" });
      card.style.border = "1px solid var(--background-modifier-border, #ccc)";
      card.style.borderRadius = "8px";
      card.style.padding = "0.75rem";
      card.style.display = "flex";
      card.style.flexDirection = "column";
      card.style.gap = "0.35rem";

      const titleRow = card.createEl("div");
      titleRow.style.display = "flex";
      titleRow.style.justifyContent = "space-between";
      titleRow.style.alignItems = "center";

      const link = titleRow.createEl("a", {
        text: book.name,
        cls: "internal-link",
      });
      link.href = book.path;

      const progressLabel = titleRow.createEl("span", {
        text: t("progressLabel", { read, total: total || "?" }),
      });
      progressLabel.style.fontSize = "0.9rem";
      progressLabel.style.color = "var(--text-muted, #777)";

      const progressBar = card.createEl("div");
      progressBar.style.width = "100%";
      progressBar.style.height = "8px";
      progressBar.style.borderRadius = "999px";
      progressBar.style.backgroundColor = "var(--background-modifier-border, #e0e0e0)";

      const fill = progressBar.createEl("div");
      fill.style.height = "100%";
      fill.style.borderRadius = "999px";
      fill.style.backgroundColor = "var(--interactive-accent, #6c5ce7)";
      fill.style.width = `${percent}%`;
      fill.style.transition = "width 0.2s ease";

      if (lastChapter?.chapter) {
        card.createEl("p", {
          text: t("lastChapterLabel", {
            chapter: lastChapter.chapter,
            when: formatDateTime(lastChapter.finished),
          }),
        }).style.margin = "0";
      }

      const button = card.createEl("button", { text: t("readChapter") });
      button.disabled = total > 0 ? read >= total : false;

      button.onclick = async () => {
        if (total === 0) {
          new Notice(t("invalidChapters"));
          return;
        }
        if (read >= total) {
          new Notice(t("allChaptersDone"));
          return;
        }
        button.disabled = true;
        const previousText = button.textContent;
        button.textContent = t("saving");
        try {
          const next = await logChapter(book);
          new Notice(t("chapterLogged", { chapter: next }));
        } catch (error) {
          console.error(error);
          new Notice(error?.message ?? t("couldNotLogChapter"));
        } finally {
          renderBooksList();
          // Restore text for the current button instance (even though it's rerendered soon)
          button.textContent = previousText;
          button.disabled = total > 0 ? book.readChapters >= total : false;
        }
      };
    });
  };

  const renderStandaloneEntries = () => {
    standaloneListContainer.innerHTML = "";
    if (standaloneEntries.length === 0) {
      standaloneListContainer.createEl("p", { text: t("noStandalone") });
      return;
    }
    standaloneEntries
      .slice(0, 50)
      .forEach((entry) => {
        const row = standaloneListContainer.createEl("div");
        row.style.display = "flex";
        row.style.flexDirection = "column";
        row.style.border = "1px solid var(--background-modifier-border, #ccc)";
        row.style.borderRadius = "8px";
        row.style.padding = "0.6rem 0.75rem";
        row.style.gap = "0.2rem";

        row.createEl("strong", {
          text: `${entry.bookName} · ${t("standaloneChapterPlaceholder")} ${entry.chapterNumber}`,
        });
        row.createEl("span", {
          text: t("finishedIn", { when: formatDateTime(entry.finishedAt) }),
        }).style.color = "var(--text-muted, #777)";
      });
  };

  const logChapter = async (book) => {
    const cache = bookCache.get(book.path) ?? { content: "", chapters: [] };
    const currentChapters = cache.chapters ?? [];
    const nextChapter = currentChapters.length + 1;
    if (nextChapter > (book.totalChapters || 0)) {
      throw new Error(t("allChaptersDone"));
    }

    const timestamp = window.moment();
    const iso = timestamp.toISOString();
    const line = `- chapter:: ${nextChapter} finished:: ${iso}\n`;
    const file = app.vault.getAbstractFileByPath(book.path);
    if (!file) {
      throw new Error("Book note is missing.");
    }
    let content = cache.content;
    if (content === undefined) {
      content = await app.vault.read(file);
    }

    const sectionRegex = /(##\s*Chapters[^\n]*\n)([\s\S]*?)(?=\n##\s+|$)/i;
    const match = sectionRegex.exec(content);

    if (match) {
      const before = content.slice(0, match.index);
      const after = content.slice(match.index + match[0].length);
      const body = match[2].trimEnd();
      const newBody = `${body ? body + "\n" : ""}${line}`;
      content = before + match[1] + newBody + after;
    } else {
      const trimmed = content.trimEnd();
      const prefix = trimmed.length ? `${trimmed}\n\n` : "";
      content = `${prefix}## Chapters\n${line}`;
    }

    await app.vault.modify(file, content);
    const updatedChapters = [...currentChapters, { chapter: nextChapter, finished: iso }];
    bookCache.set(book.path, { content, chapters: updatedChapters });
    book.chapters = updatedChapters;
    book.readChapters = updatedChapters.length;
    await incrementBooksXp();
    return nextChapter;
  };

  bookForm.onsubmit = async (event) => {
    event.preventDefault();
    const name = bookNameInput.value.trim();
    const totalChapters = parseChapterCount(bookChaptersInput.value);
    if (!name) {
      new Notice(t("invalidBookName"));
      return;
    }
    if (totalChapters <= 0) {
      new Notice(t("invalidChapters"));
      return;
    }
    const slug = slugify(name);
    if (books.some((book) => book.slug === slug)) {
      new Notice(t("bookExists"));
      return;
    }
    bookFormButton.disabled = true;
    const previousText = bookFormButton.textContent;
    bookFormButton.textContent = t("saving");
    try {
      const escapedName = escapeYaml(name);
      const fileBase = fileSafe(name);
      let targetPath = `${booksFolder}/${fileBase}.md`;
      let counter = 1;
      while (app.vault.getAbstractFileByPath(targetPath)) {
        counter += 1;
        targetPath = `${booksFolder}/${fileBase}-${counter}.md`;
      }
      const content = `---
bookType: book
bookName: "${escapedName}"
totalChapters: ${totalChapters}
---

# ${name}

> [!info]
> Use [[Books]] to log each chapter you read for this book.

## Chapters
`;
      await app.vault.create(targetPath, content);
      const newBook = {
        name,
        totalChapters,
        path: targetPath,
        slug,
      };
      books.push(newBook);
      books.sort((a, b) => a.name.localeCompare(b.name));
      await hydrateBook(newBook);
      renderBooksList();
      bookNameInput.value = "";
      bookChaptersInput.value = "";
      new Notice(t("bookCreated"));
    } catch (error) {
      console.error(error);
      new Notice(t("couldNotCreateBook"));
    } finally {
      bookFormButton.disabled = false;
      bookFormButton.textContent = previousText;
    }
  };

  standaloneForm.onsubmit = async (event) => {
    event.preventDefault();
    const bookName = standaloneBookInput.value.trim();
    const chapterNumber = parseChapterCount(standaloneChapterInput.value);
    if (!bookName) {
      new Notice(t("invalidBookName"));
      return;
    }
    if (chapterNumber <= 0) {
      new Notice(t("invalidChapters"));
      return;
    }
    standaloneButton.disabled = true;
    const previousText = standaloneButton.textContent;
    standaloneButton.textContent = t("saving");
    try {
      const timestamp = window.moment();
      const iso = timestamp.toISOString();
      const escapedBook = escapeYaml(bookName);
      const safeName = fileSafe(bookName) || "chapter";
      const stamp = timestamp.format("YYYYMMDD-HHmmss");
      let targetPath = `${booksFolder}/${safeName}-chapter-${chapterNumber}-${stamp}.md`;
      let counter = 1;
      while (app.vault.getAbstractFileByPath(targetPath)) {
        counter += 1;
        targetPath = `${booksFolder}/${safeName}-chapter-${chapterNumber}-${stamp}-${counter}.md`;
      }
      const content = `---
bookType: entry
bookName: "${escapedBook}"
chapterNumber: ${chapterNumber}
finishedAt: ${iso}
---

# ${bookName} · Chapter ${chapterNumber}

finished in: ${timestamp.format("YYYY-MM-DD HH:mm")}
`;
      await app.vault.create(targetPath, content);
      await incrementBooksXp();
      standaloneEntries.unshift({
        bookName,
        chapterNumber,
        finishedAt: iso,
        path: targetPath,
        ms: new Date(iso).getTime(),
      });
      renderStandaloneEntries();
      standaloneBookInput.value = "";
      standaloneChapterInput.value = "";
      new Notice(t("standaloneSaved"));
    } catch (error) {
      console.error(error);
      new Notice(t("standaloneError"));
    } finally {
      standaloneButton.disabled = false;
      standaloneButton.textContent = previousText;
    }
  };

  renderBooksList();
  renderStandaloneEntries();
};

run();
```
