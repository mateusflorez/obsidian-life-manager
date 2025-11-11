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
> Click **+ New finance month** to create a fresh month from the template.

```meta-bind-button
label: New finance month
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
    templateFile: templates/new finance month.md
    folderPath: finance/2025
    fileName: month
    openNote: true
    openIfAlreadyExists: true

```

```dataviewjs
const run = async () => {
  const pages = dv.pages('"finance"').where(p => {
    const parts = p.file.folder.split('/');
    return parts.length === 2 && parts[0] === 'finance' && !isNaN(parts[1]);
  });

  if (pages.length === 0) {
    dv.paragraph("❌ Create `finance/2025/November.md` with `- cat:: value`");
    return;
  }

  const formatCurrency = (value) => `R$ ${Number(value || 0).toFixed(2).replace('.', ',')}`;
  const makeTagSlug = (value) =>
    value
      .toLowerCase()
      .normalize('NFD')
      .replace(/[\u0300-\u036f]/g, '')
      .replace(/[^a-z0-9-]+/g, '-')
      .replace(/-+/g, '-')
      .replace(/^-+|-+$/g, '');

  const loadCategories = (() => {
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

  const yearLabel = controls.createEl('label', { text: 'Year:' });
  yearLabel.style.fontWeight = '600';

  const yearSelect = controls.createEl('select');
  yearSelect.style.padding = '0.25rem 0.5rem';
  years.forEach(y => yearSelect.createEl('option', { text: y, value: y }));
  yearSelect.value = years[0];

  const today = window.moment();
  const currentYear = today.format('YYYY');
  const currentMonthName = today.format('MMMM');
  const currentPath = `finance/${currentYear}/${currentMonthName}.md`;

  const openMonthBtn = controls.createEl('button', { text: `Open ${currentMonthName}` });
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
      dataContainer.createEl('p', { text: 'No financial data for this year.' });
      return;
    }

    monthEntries.sort((a, b) => new Date(`${b.month} 1, ${year}`) - new Date(`${a.month} 1, ${year}`));
    const months = monthEntries.map(m => m.month);

    const summaryTable = dataContainer.createEl('div');
    summaryTable.createEl('h3', { text: `Summary ${year}` });
    const rows = monthEntries.map(entry => [
      entry.month,
      formatCurrency(entry.expenses.total),
      formatCurrency(entry.income.total),
      formatCurrency(entry.income.total - entry.expenses.total)
    ]);
    dv.table(['Month', 'Expenses', 'Income', 'Balance'], rows, summaryTable);

    const chartWrapper = dataContainer.createEl('div');
    chartWrapper.createEl('h3', { text: `Trend ${year}` });
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
              label: 'Expenses',
              data: monthEntries.map(m => m.expenses.total),
              borderColor: '#FF6384',
              backgroundColor: '#FF638433',
              tension: 0.3
            },
            {
              label: 'Income',
              data: monthEntries.map(m => m.income.total),
              borderColor: '#36A2EB',
              backgroundColor: '#36A2EB33',
              tension: 0.3
            },
            {
              label: 'Balance',
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
            title: { display: true, text: `Finances ${year}` },
            legend: { position: 'bottom' }
          },
          scales: {
            y: {
              beginAtZero: true,
              ticks: { callback: (value) => `R$ ${value}` }
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
        parent.createEl('p', { text: 'No entries yet.' }).style.color = 'var(--text-muted)';
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

      const nameLabel = form.createEl('label', { text: 'Category' });
      nameLabel.style.fontWeight = '600';
      const nameInput = form.createEl('select');
      nameInput.required = true;
      const placeholder = nameInput.createEl('option', { text: 'Select category', value: '' });
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
      const valueInput = form.createEl('input', { attr: { type: 'number', step: '0.01', placeholder: 'Amount', required: 'true' } });
      const noteInput = form.createEl('input', { attr: { type: 'text', placeholder: 'Optional tag (e.g., rent)' } });
      const submitBtn = form.createEl('button', { text: `Add ${heading.toLowerCase()}` });
      submitBtn.type = 'submit';
      submitBtn.style.cursor = 'pointer';

      form.onsubmit = async (event) => {
        event.preventDefault();
        const category = nameInput.value.trim();
        const value = parseFloat((valueInput.value || '').replace(',', '.'));
        if (!category || isNaN(value) || value === 0) {
          new Notice('Select a category and enter a non-zero amount.');
          return;
        }
        submitBtn.disabled = true;
        submitBtn.textContent = 'Saving...';

        const tag = noteInput.value.trim();
        const safeTag = tag ? makeTagSlug(tag) : '';
        const amountStr = Number(value).toFixed(2);
        const line = `- ${category}:: ${amountStr}${safeTag ? ` #${safeTag}` : ''}`;

        try {
          await insertLine(filePath, heading, line);
          new Notice(`${heading} added! Reload to see the update.`);
          nameInput.value = '';
          valueInput.value = '';
          noteInput.value = '';
        } catch (error) {
          console.error(error);
          const reason = error?.message || 'unknown error';
          new Notice(`Could not save entry (${reason}).`);
        } finally {
          submitBtn.disabled = false;
          submitBtn.textContent = `Add ${heading.toLowerCase()}`;
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

      const openButton = cardHeader.createEl('button', { text: 'Open month' });
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

        const colTitle = colHeader.createEl('h4', { text: heading });
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
