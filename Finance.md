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
(() => {
  const pages = dv.pages('"finance"').where(p => {
    const parts = p.file.folder.split('/');
    return parts.length === 2 && parts[0] === 'finance' && !isNaN(parts[1]);
  });

  if (pages.length === 0) {
    dv.paragraph("❌ Create `finance/2025/November.md` with `- cat:: value`");
    return;
  }

  const years = [...new Set(pages.map(p => p.file.folder.split('/')[1]))].map(Number).sort((a, b) => b - a);

  const yearSelect = dv.container.createEl('select');
  years.forEach(y => yearSelect.createEl('option', { text: y, value: y }));
  yearSelect.value = years[0];

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

  const parseSectionSum = (content, heading) => {
    const after = content.split(new RegExp(`##\\s*${heading}`, 'i'))[1];
    if (!after) return 0;
    const section = after.split(/\n##\s+/)[0];
    return section.split('\n').reduce((total, rawLine) => {
      const line = rawLine.trim();
      if (!line) return total;
      const match = line.match(/^-?\s*([^:]+)::\s*([\d.,]+)/);
      if (!match) return total;
      const value = parseFloat(match[2].replace(',', '.'));
      return isNaN(value) ? total : total + value;
    }, 0);
  };

  const renderYear = async (year) => {
    dataContainer.innerHTML = '';
    const yearPages = pages.where(p => p.file.folder === `finance/${year}`);
    const monthData = {};

    for (const p of yearPages) {
      const month = p.file.name.replace('.md', '');
      const content = await dv.io.load(p.file.path);
      const exp = parseSectionSum(content, 'Expenses');
      const rec = parseSectionSum(content, 'Income');
      if (exp === 0 && rec === 0) continue;
      monthData[month] = { exp, rec, res: rec - exp };
    }

    const months = Object.keys(monthData).sort((a, b) =>
      new Date(`${b} 1, ${year}`) - new Date(`${a} 1, ${year}`)
    );

    if (months.length === 0) {
      dataContainer.createEl('p', { text: 'Nenhum dado financeiro para este ano.' });
      return;
    }

    const summary = months.map(m => ({ month: m, ...monthData[m] }));

    // TABLE
    dataContainer.createEl('h3', { text: `Summary ${year}` });
    const tableEl = dataContainer.createEl('div');
    dv.table(
      ['Month', 'Expenses', 'Income', 'Balance'],
      summary.map(s => [
        s.month,
        `R$ ${s.exp.toFixed(2).replace('.', ',')}`,
        `R$ ${s.rec.toFixed(2).replace('.', ',')}`,
        `R$ ${s.res.toFixed(2).replace('.', ',')}`
      ]),
      tableEl
    );

    // CHART
    dataContainer.createEl('h3', { text: `Chart ${year}` });
    const chartDiv = dataContainer.createEl('div', { attr: { style: 'height:400px; margin:20px 0;' } });
    loadChartJs().then(() => {
      const canvas = document.createElement('canvas');
      chartDiv.appendChild(canvas);
      new Chart(canvas, {
        type: 'bar',
        data: {
          labels: months,
          datasets: [
            { label: 'Expenses', data: summary.map(s => s.exp), backgroundColor: '#FF6384CC', borderColor: '#FF6384' },
            { label: 'Income', data: summary.map(s => s.rec), backgroundColor: '#36A2EBCC', borderColor: '#36A2EB' },
            { label: 'Balance', data: summary.map(s => s.res), backgroundColor: '#4BC0C0CC', borderColor: '#4BC0C0' }
          ]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          scales: { y: { beginAtZero: true, title: { display: true, text: 'R$' } } },
          plugins: { title: { display: true, text: `Finances ${year}` } }
        }
      });
    });
  };

  renderYear(years[0]);
  yearSelect.onchange = () => renderYear(yearSelect.value);
})();
```
```dataviewjs
(() => {
  // 1. Fetch finance/Year/*.md notes
  const pages = dv.pages('"finance"').where(p => {
    const parts = p.file.folder.split('/');
    return parts.length === 2 && parts[0] === 'finance' && !isNaN(parts[1]);
  });

  if (pages.length === 0) {
    dv.paragraph("❌ Create `finance/2025/November.md` with `- cat:: value #title`");
    return;
  }

  // 2. Year select
  const years = [...new Set(pages.map(p => p.file.folder.split('/')[1]))].map(Number).sort((a, b) => b - a);
  const yearSelect = dv.container.createEl('select');
  years.forEach(y => yearSelect.createEl('option', { text: y, value: y }));
  yearSelect.value = years[0];

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

  // 3. Render per year
  const renderYear = async (year) => {
    dataContainer.innerHTML = '';
    const yearPages = pages.where(p => p.file.folder === `finance/${year}`);

    const monthMap = {};

    for (const p of yearPages) {
      const month = p.file.name.replace('.md', '');
      monthMap[month] = monthMap[month] || { cats: {} };

      const content = await dv.io.load(p.file.path);

      const afterExpenses = content.split(/##\s*Expenses/i)[1];
      if (!afterExpenses) continue;
      const expensesSection = afterExpenses.split(/\n##\s+/)[0];
      const lines = expensesSection.split('\n');

      for (const rawLine of lines) {
        const line = rawLine.trim();
        if (!line) continue;

        const match = line.match(/^-?\s*([^:]+)::\s*([\d.,]+)(?:\s*#(.+))?/);
        if (!match) continue;

        const [, catRaw, valStr, titleRaw] = match;
        const cat = catRaw.trim();
        const value = parseFloat(valStr.replace(',', '.'));
        if (isNaN(value)) continue;

        const catData = monthMap[month].cats[cat] = monthMap[month].cats[cat] || { total: 0, items: [] };
        catData.total += value;
        catData.items.push({ title: (titleRaw || '-').trim(), value });
      }

      if (Object.keys(monthMap[month].cats).length === 0) {
        delete monthMap[month];
      }
    }

    const months = Object.keys(monthMap).sort((a, b) =>
      new Date(`${b} 1, ${year}`) - new Date(`${a} 1, ${year}`)
    );

    if (months.length === 0) {
      dataContainer.createEl('p', { text: 'No expenses found for this year.' });
      return;
    }

    months.forEach(month => {
      const cats = monthMap[month].cats;

      // PIE CHART (expenses only)
      dataContainer.createEl('h3', { text: `${month}:` });
      const pieDiv = dataContainer.createEl('div', { attr: { style: 'height:300px; margin:15px 0;' } });

      const labels = Object.keys(cats);
      const data = labels.map(c => cats[c].total);
      const palette = ['#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF', '#FF9F40', '#C9CBCF'];

      loadChartJs().then(() => {
        const canvas = document.createElement('canvas');
        pieDiv.appendChild(canvas);
        new Chart(canvas, {
          type: 'pie',
          data: {
            labels,
            datasets: [{
              data,
              backgroundColor: labels.map((_, idx) => palette[idx % palette.length] + 'CC')
            }]
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: { title: { display: true, text: `Expenses - ${month}` }, legend: { position: 'right' } }
          }
        });
      });

      // TABLE (expenses only)
      const tableDiv = dataContainer.createEl('div');
      const rows = [];
      Object.keys(cats).sort().forEach(cat => {
        rows.push([cat, `R$ ${cats[cat].total.toFixed(2).replace('.', ',')}`]);
        cats[cat].items.forEach(i => {
          rows.push([`   ${i.title}`, `R$ ${i.value.toFixed(2).replace('.', ',')}`]);
        });
      });

      dv.table(['Category', 'Total'], rows, tableDiv);
    });
  };

  renderYear(years[0]);
  yearSelect.onchange = () => renderYear(yearSelect.value);
})();
```
