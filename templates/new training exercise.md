# {{EXERCISE_NAME}}

> [!info]
> Track {{EXERCISE_NAME}} volume (load x reps) over time.

```dataviewjs
const run = async () => {
  const exerciseFile = dv.current()?.file;
  const exerciseName = exerciseFile?.name ?? "{{EXERCISE_NAME}}";
  const sessionsHeading = "Sessions";

  const getMutedColor = () => {
    if (typeof window === "undefined" || !window.getComputedStyle) return "#666";
    const value = window.getComputedStyle(document.body).getPropertyValue("--text-muted");
    return value && value.trim().length ? value.trim() : "#666";
  };

  const renderLineChart = (target, labels, values) => {
    const width = 640;
    const height = 280;
    const padding = 36;
    const canvas = target.createEl("canvas");
    canvas.width = width;
    canvas.height = height;
    canvas.style.width = "100%";
    canvas.style.height = "100%";

    const ctx = canvas.getContext("2d");
    ctx.clearRect(0, 0, width, height);

    const maxValue = Math.max(...values);
    const minValue = Math.min(...values);
    const range = maxValue - minValue || 1;
    const steps = Math.max(values.length - 1, 1);
    const points = values.map((value, idx) => {
      const x = padding + (idx / steps) * (width - padding * 2);
      const y =
        height - padding - ((value - minValue) / range) * (height - padding * 2);
      return { x, y, label: labels[idx], value };
    });

    ctx.fillStyle = "rgba(75, 192, 192, 0.15)";
    ctx.beginPath();
    points.forEach((point, idx) => {
      if (idx === 0) ctx.moveTo(point.x, point.y);
      else ctx.lineTo(point.x, point.y);
    });
    ctx.lineTo(points[points.length - 1].x, height - padding);
    ctx.lineTo(points[0].x, height - padding);
    ctx.closePath();
    ctx.fill();

    ctx.strokeStyle = "#4BC0C0";
    ctx.lineWidth = 2;
    ctx.beginPath();
    points.forEach((point, idx) => {
      if (idx === 0) ctx.moveTo(point.x, point.y);
      else ctx.lineTo(point.x, point.y);
    });
    ctx.stroke();

    ctx.fillStyle = "#4BC0C0";
    points.forEach((point) => {
      ctx.beginPath();
      ctx.arc(point.x, point.y, 3, 0, Math.PI * 2);
      ctx.fill();
    });

    const labelColor = getMutedColor();
    ctx.fillStyle = labelColor;
    ctx.font = "12px sans-serif";
    ctx.textAlign = "center";
    const step = Math.max(1, Math.ceil(points.length / 6));
    points.forEach((point, idx) => {
      if (idx % step === 0 || idx === points.length - 1) {
        ctx.fillText(labels[idx], point.x, height - padding + 16);
      }
    });
  };

  if (!exerciseFile) {
    dv.paragraph("Open this exercise note to see logged sets.");
    return;
  }

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
    if (!fields.load || !fields.reps) return null;

    const load = Number(fields.load.replace(/,/g, "."));
    const reps = Number(fields.reps.replace(/,/g, "."));
    const date = fields.date
      ? window.moment(fields.date, "YYYY-MM-DD", true)
      : null;

    return {
      load: isFinite(load) ? load : null,
      reps: isFinite(reps) ? reps : null,
      date,
      dateStr: date && date.isValid() ? date.format("YYYY-MM-DD") : null,
    };
  };

  const content = await dv.io.load(exerciseFile.path);
  const sectionRegex = new RegExp(`##\\s*${sessionsHeading}[^\\n]*\\n([\\s\\S]*?)(?=\\n##\\s+|$)`, "i");
  const match = sectionRegex.exec(content);
  const body = match ? match[1] : "";

  const entries = body
    .split("\n")
    .map(parseLine)
    .filter(Boolean);

  if (entries.length === 0) {
    dv.paragraph("No logged sets yet for this exercise.");
    return;
  }

  const grouped = entries.reduce((acc, entry) => {
    if (!entry.dateStr || entry.load === null || entry.reps === null) return acc;
    const volume = entry.load * entry.reps;
    acc[entry.dateStr] = (acc[entry.dateStr] || 0) + volume;
    return acc;
  }, {});

  const dates = Object.keys(grouped).sort((a, b) => (a < b ? -1 : 1));
  if (dates.length === 0) {
    dv.paragraph("No dated volume entries available.");
    return;
  }

  const container = dv.container.createEl("div", { cls: "exercise-chart" });
  container.style.minHeight = "280px";
  container.style.marginTop = "1rem";
  renderLineChart(container, dates, dates.map((date) => grouped[date]));

  const tableWrapper = dv.container.createEl("div");
  tableWrapper.style.marginTop = "1.5rem";
  const headingEl = tableWrapper.createEl("h3", { text: "Logged sets" });
  headingEl.style.marginBottom = "0.5rem";

  const table = tableWrapper.createEl("table");
  table.style.width = "100%";
  table.style.borderCollapse = "collapse";

  const headerRow = table.createEl("tr");
  ["Date", "Load", "Reps", "Volume"].forEach((label) => {
    const th = headerRow.createEl("th", { text: label });
    th.style.textAlign = "left";
    th.style.borderBottom = "1px solid var(--background-modifier-border)";
    th.style.padding = "0.35rem";
  });

  const sorted = entries
    .filter((entry) => entry.dateStr && entry.load !== null && entry.reps !== null)
    .sort((a, b) => (a.dateStr > b.dateStr ? -1 : 1));

  sorted.forEach((entry) => {
    const tr = table.createEl("tr");
    const cells = [
      entry.dateStr,
      entry.load,
      entry.reps,
      entry.load * entry.reps,
    ];
    cells.forEach((value) => {
      const td = tr.createEl("td", { text: value.toString() });
      td.style.padding = "0.35rem";
      td.style.borderBottom = "1px solid var(--background-modifier-border)";
    });
  });
};

run();
```

## Sessions
<!-- Add lines like `- date:: 2025-11-10 load:: 100 reps:: 5 notes:: optional` -->
