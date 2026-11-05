---
layout: page
title:  "FY Comparison"
---

<style>
  /* Hide the page title heading; page.title is still used by the navbar. */
  .post-header { display: none; }
  /* Fixed height so the chart never resizes (and pushes the table) when the year changes. */
  #usage-chart { max-width: 800px; height: 450px; }
  #campus-picker { margin: 1em 0; }
  #campus-picker select {
    padding: .45em .9em;
    border: 1px solid #9ca3af;
    border-radius: 6px;
    background: #f9fafb;
    font: inherit;
  }
  /* Buttons that switch between water usage, sewer usage and cost. */
  #metric-buttons { display: flex; gap: .5em; flex-wrap: wrap; margin: .5em 0 1em; }
  #metric-buttons button {
    padding: .45em .9em;
    border: 1px solid #6b7280;
    border-radius: 6px;
    background: #f9fafb;
    color: #1f2937;
    cursor: pointer;
    font: inherit;
  }
  #metric-buttons button[aria-pressed="true"] {
    border-color: #1d4ed8;
    background: #dbeafe;
    color: #1e3a8a;
    font-weight: 600;
  }
  #metric-buttons button:focus-visible { outline: 2px solid #2a6fb0; outline-offset: 2px; }
  /* Floating fiscal year picker that stays beside the chart while scrolling.
     The theme's content column is 740px wide and centered, so anchor the box's
     right edge 16px left of that column instead of the screen edge. */
  #fy-float {
    position: fixed; top: 90px; right: calc(50% + 370px + 16px); z-index: 10;
    background: #ffffff; color: #1f2937;
    border: 1px solid #e3e6ea; border-radius: 6px; padding: 8px 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
  }
  #usage-fy label { display: block; }
  #fy-toggle { display: none; }
  #fy-toggle:focus-visible { outline: 2px solid #2a6fb0; outline-offset: 2px; }
  /* On narrow screens, collapse the picker below the navbar to keep it out of the way. */
  @media (max-width: 1100px) {
    #fy-float { top: 72px; left: 12px; right: auto; padding: 6px; }
    #usage-heading { margin-top: 100px; }
    #fy-toggle {
      display: block; padding: 8px 12px; border: 0; border-radius: 4px;
      background: transparent; color: inherit; font: inherit; cursor: pointer;
    }
    #fy-title, #usage-fy { display: none; }
    #fy-float.is-open #usage-fy { display: block; padding: 0 12px 8px; }
  }
  #fy-totals-table, #usage-table {
    border-collapse: collapse; margin-top: .5em;
  }
  #fy-totals-table th, #fy-totals-table td, #usage-table th, #usage-table td {
    padding: 2px 12px; text-align: right;
  }
  /* Make "Show data table" look like a button, with an arrow that flips when open. */
  details > summary {
    display: inline-block; list-style: none; cursor: pointer; user-select: none;
    padding: 6px 14px; border: 1px solid currentColor; border-radius: 6px;
    background: rgba(127, 127, 127, 0.12); font-weight: 600;
  }
  details > summary::-webkit-details-marker { display: none; }
  details > summary::before { content: "\25B8"; display: inline-block; margin-right: 8px; transition: transform .15s; }
  details[open] > summary::before { transform: rotate(90deg); }
  details > summary:hover { background: rgba(127, 127, 127, 0.25); }
  details > summary:focus-visible { outline: 2px solid #2a6fb0; outline-offset: 2px; }
</style>

<script src="https://cdn.jsdelivr.net/npm/plotly.js-dist-min@2.35.2/plotly.min.js"></script>

{% include quick-links.html %}
{% include table-export.html %}

<h1>Fiscal Year Comparison</h1>

<div id="campus-picker">
  <label for="campus-select">Campus: </label>
  <select id="campus-select"></select>
</div>

<div id="metric-buttons" role="group" aria-label="Choose what to compare">
  <button type="button" data-metric="water" aria-pressed="true">Water Usage</button>
  <button type="button" data-metric="sewer" aria-pressed="false">Sewer Usage</button>
  <button type="button" data-metric="cost" aria-pressed="false">Cost (Water + Sewer)</button>
</div>

<h3 id="usage-heading">Total Water Usage by Fiscal Year</h3>
<details id="fy-totals-details" open>
  <summary id="fy-totals-summary">Hide fiscal year table</summary>
  <table id="fy-totals-table">
    <thead><tr><th>Fiscal Year</th><th id="totals-header">Total (kgal)</th></tr></thead>
    <tbody></tbody>
  </table>
</details>
<script>
  // Shown by default; the button text flips between "Hide" and "Show" as the table is closed and opened.
  (() => {
    const details = document.getElementById("fy-totals-details");
    const summary = document.getElementById("fy-totals-summary");
    details.addEventListener("toggle", () => {
      summary.textContent = details.open ? "Hide fiscal year table" : "Show fiscal year table";
    });
  })();
</script>

<h3 id="chart-heading">Monthly Water Usage</h3>
<div id="fy-float">
  <button id="fy-toggle" type="button" aria-expanded="false" aria-controls="usage-fy">Select FY</button>
  <div id="fy-title"><strong>Fiscal years</strong></div>
  <div id="usage-fy"></div>
</div>
<script>
  (() => {
    const picker = document.getElementById("fy-float");
    const toggle = document.getElementById("fy-toggle");
    toggle.addEventListener("click", () => {
      const open = toggle.getAttribute("aria-expanded") !== "true";
      picker.classList.toggle("is-open", open);
      toggle.setAttribute("aria-expanded", String(open));
    });
  })();
</script>
<div id="chart-msg">Loading campus data...</div>
<div id="usage-chart"></div>
<details>
  <summary>Show data table</summary>
  <table id="usage-table"></table>
</details>

<script>
  // Fiscal year runs July -> June.
  const MONTHS = ["Jul","Aug","Sep","Oct","Nov","Dec","Jan","Feb","Mar","Apr","May","Jun"];
  const monthIndex = m => (m + 5) % 12; // calendar month (1-12) -> position in fiscal year (0-11)

  // What the page can compare. Usage is in thousand gallons (kgal); cost is water + sewer charges.
  const METRICS = {
    water: { label: "Water Usage", money: false },
    sewer: { label: "Sewer Usage", money: false },
    cost:  { label: "Cost (Water + Sewer)", money: true }
  };
  let selectedMetric = "water";
  const metric = () => METRICS[selectedMetric];

  // 1,234 for usage, $1,234.56 for cost
  const formatValue = value => metric().money
    ? value.toLocaleString("en-US", { style: "currency", currency: "USD" })
    : value.toLocaleString("en-US", { maximumFractionDigits: 0 });

  // Minimal CSV parser that handles quoted fields containing commas.
  function parseCSV(text) {
    const rows = [];
    let row = [], field = "", inQuotes = false;
    for (let i = 0; i < text.length; i++) {
      const c = text[i];
      if (inQuotes) {
        if (c === '"' && text[i + 1] === '"') { field += '"'; i++; }
        else if (c === '"') inQuotes = false;
        else field += c;
      } else if (c === '"') inQuotes = true;
      else if (c === ",") { row.push(field); field = ""; }
      else if (c === "\n" || c === "\r") {
        if (c === "\r" && text[i + 1] === "\n") i++;
        row.push(field); field = "";
        rows.push(row); row = [];
      } else field += c;
    }
    if (field !== "" || row.length) { row.push(field); rows.push(row); }
    return rows;
  }

  // byCampus[campus][fy] = { water: [12 months], sewer: [12 months], cost: [12 months] }
  // (month 0 = July). A month with no value stays null.
  const byCampus = {};
  let selectedCampus;
  const chartEl = document.getElementById("usage-chart");
  const campusSelect = document.getElementById("campus-select");

  // The selected campus's data: data[fy][metric][monthPosition]
  const campusData = () => byCampus[selectedCampus] || {};

  // The 12 monthly values of a fiscal year for the selected campus and metric.
  const valuesFor = fy => (campusData()[fy] || {})[selectedMetric] || [];

  // Fiscal years that have some data for the selected campus and metric, oldest first.
  function campusFYs() {
    return Object.keys(campusData()).filter(fy => valuesFor(fy).some(value => value)).sort();
  }

  // How many months of a fiscal year have data for the selected campus and metric.
  function monthsWithData(fy) {
    return valuesFor(fy).filter(value => value).length;
  }

  // Fiscal years currently checked, oldest first so the order is stable.
  function selectedFYs() {
    return [...document.querySelectorAll('input[name="usage-fy"]:checked')].map(i => i.value).sort();
  }

  // Each fiscal year keeps its own color no matter which others are checked, or which campus is shown.
  const PALETTE = ["#3b82c4", "#e08a2e", "#3da36b", "#9b6bc9", "#d65a7a", "#8a8f98", "#c9a227", "#2aa5a5", "#a65d3f", "#5b6b2a"];
  let allFYs = []; // every fiscal year in the file, oldest first
  const colorFor = fy => PALETTE[allFYs.indexOf(fy) % PALETTE.length];

  // Download a PNG that includes value labels, units and a title. The export is drawn
  // in an off-screen chart so the on-screen chart never flickers.
  async function savePngWithValues(gd, title, filename) {
    const W = 1100, H = 650;
    const tmp = document.createElement("div");
    tmp.style.cssText = "position:fixed; left:-10000px; top:0; width:" + W + "px; height:" + H + "px;";
    document.body.append(tmp);
    try {
      const exportData = gd.data.map((trace, i) => Object.assign({}, trace, {
        mode: "lines+markers+text",
        texttemplate: metric().money ? "$%{y:,.0f}" : "%{y:,.0f}",
        // alternate above/below so labels from different years overlap less
        textposition: i % 2 ? "bottom center" : "top center"
      }));
      const exportLayout = Object.assign({}, gd.layout, {
        width: W,
        height: H,
        title: { text: title },
        paper_bgcolor: "#ffffff",   // the on-screen chart is transparent, which looks bad in a PNG
        margin: { t: 60, r: 24, b: 80, l: 80 },
        xaxis: Object.assign({}, gd.layout.xaxis, { title: { text: "Month (fiscal year, Jul - Jun)" } }),
        yaxis: Object.assign({}, gd.layout.yaxis, {
          title: { text: metric().label + (metric().money ? " (USD)" : " (kgal)") }
        })
      });
      await Plotly.newPlot(tmp, exportData, exportLayout, { staticPlot: true });
      await Plotly.downloadImage(tmp, { format: "png", width: W, height: H, scale: 2, filename: filename });
    } finally {
      Plotly.purge(tmp);
      tmp.remove();
    }
  }

  // Draw one line per entry in series ([{ name, vals, color }]) into the chart element.
  function drawLines(el, series) {
   // dark mode looks weird
    // const dark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    const ink = "#1f2937";
    const grid = "#3a3f47";
    const money = metric().money;

    // null values leave a gap in the line instead of being drawn as zero.
    Plotly.react(el, series.map(s => ({
      x: MONTHS,
      y: s.vals,
      type: "scatter",
      mode: "lines+markers",
      name: s.name,
      line: { color: s.color, width: 2, dash: s.dash || "solid" },
      marker: { size: 8 },
      connectgaps: false,
      hovertemplate: money ? "$%{y:,.2f}" : "%{y:,.0f} kgal"
    })), {
      height: 450,
      margin: { t: 20, r: 24, b: 40, l: 70 },
      paper_bgcolor: "rgba(0,0,0,0)",
      plot_bgcolor: "rgba(0,0,0,0)",
      font: { color: ink },
      hoverlabel: { bgcolor: "#ffffff", bordercolor: "#d1d5db", font: { color: "#1f2937" } },
      hovermode: "x unified",
      showlegend: series.length > 1,
      legend: { orientation: "h", y: -0.12 },
      xaxis: { categoryorder: "array", categoryarray: MONTHS, gridcolor: grid, fixedrange: true },
      yaxis: {
        title: money ? "USD" : "kgal",
        tickprefix: money ? "$" : "",
        rangemode: "tozero",
        gridcolor: grid,
        fixedrange: true
      }
    }, {
      responsive: true,
      displaylogo: false,
      // replace the stock camera button so downloads include titles, units and values
      modeBarButtonsToRemove: ["toImage"],
      modeBarButtonsToAdd: [{
        name: "Download PNG with values",
        title: "Download PNG with values",
        icon: Plotly.Icons.camera,
        click: gd => {
          const title = "Monthly " + metric().label + " - " + selectedCampus;
          return savePngWithValues(gd, title, title.replace(/[^\w]+/g, "_"));
        }
      }]
    });
  }

  // One line per selected fiscal year for the selected campus and metric.
  function render() {
    const fys = selectedFYs();
    const table = document.getElementById("usage-table");
    table.replaceChildren();
    const chartMsg = document.getElementById("chart-msg");
    document.getElementById("chart-heading").textContent = "Monthly " + metric().label + " - " + selectedCampus;

    if (!fys.length) {
      Plotly.purge(chartEl);
      chartMsg.textContent = "Select at least one fiscal year.";
      return;
    }
    chartMsg.textContent = "";
    drawLines(chartEl, fys.map(fy => ({ name: fy.toUpperCase(), vals: valuesFor(fy), color: colorFor(fy) })));

    // Table view: one column per selected fiscal year
    table.insertAdjacentHTML("beforeend",
      "<tr><th>Month</th>" + fys.map(fy => "<th>" + fy.toUpperCase() + "</th>").join("") + "</tr>");
    MONTHS.forEach((m, i) => {
      table.insertAdjacentHTML("beforeend",
        "<tr><td>" + m + "</td>" + fys.map(fy => {
          const value = valuesFor(fy)[i];
          return "<td>" + (value == null ? "–" : formatValue(value)) + "</td>";
        }).join("") + "</tr>");
    });
  }

  // One checkbox per fiscal year this campus has data for, newest first. Years that were
  // already checked stay checked when switching campuses or metrics; otherwise the newest two
  // full fiscal years (12 months of data) start checked, so a partial year doesn't look sparse.
  function buildOptions(previousFys = []) {
    const container = document.getElementById("usage-fy");
    const available = campusFYs();
    const kept = previousFys.filter(fy => available.includes(fy));
    const full = available.filter(fy => monthsWithData(fy) >= 12);
    const checked = kept.length ? kept : (full.length ? full : available).slice(-2);
    container.replaceChildren();

    // "All" checks or clears every year; it is checked whenever every year is checked.
    // It has its own name so selectedFYs() doesn't count it as a fiscal year.
    const allLabel = document.createElement("label");
    const allInput = document.createElement("input");
    allInput.type = "checkbox"; allInput.name = "usage-fy-all";
    allLabel.append(allInput, " All");
    container.append(allLabel);
    const yearInputs = () => [...container.querySelectorAll('input[name="usage-fy"]')];
    const syncAll = () => { allInput.checked = yearInputs().every(i => i.checked); };
    allInput.addEventListener("change", () => {
      yearInputs().forEach(i => { i.checked = allInput.checked; });
      render();
    });

    available.slice().reverse().forEach(fy => {
      const label = document.createElement("label");
      const input = document.createElement("input");
      input.type = "checkbox"; input.name = "usage-fy"; input.value = fy;
      input.checked = checked.includes(fy);
      input.addEventListener("change", () => { syncAll(); render(); });
      label.append(input, " " + fy.toUpperCase());
      container.append(label);
    });
    syncAll();
  }

  function renderFYTotals() {
    document.getElementById("usage-heading").textContent =
      selectedCampus + " - Total " + metric().label + " by Fiscal Year";
    document.getElementById("totals-header").textContent = metric().money ? "Total (USD)" : "Total (kgal)";
    const body = document.querySelector("#fy-totals-table tbody");
    body.replaceChildren();
    campusFYs().reverse().forEach(fy => {
      const row = document.createElement("tr");
      const total = valuesFor(fy).reduce((sum, value) => sum + (value || 0), 0);
      [fy.toUpperCase(), formatValue(total)].forEach(value => {
        const cell = document.createElement("td");
        cell.textContent = value;
        row.append(cell);
      });
      body.append(row);
    });
  }

  function setupCampusPicker() {
    const campuses = Object.keys(byCampus).sort();
    campuses.forEach(campus => {
      const option = document.createElement("option");
      option.value = campus;
      option.textContent = campus;
      campusSelect.append(option);
    });
    // Default to Manoa ("Manoa", "UH Manoa", "Mānoa"...); fall back to the first campus.
    const plain = name => name.normalize("NFD").replace(/[\u0300-\u036f]/g, "");
    selectedCampus = campuses.find(c => /manoa/i.test(plain(c))) || campuses[0];
    campusSelect.value = selectedCampus;

    campusSelect.addEventListener("change", () => {
      const previousFys = selectedFYs();
      selectedCampus = campusSelect.value;
      buildOptions(previousFys);
      renderFYTotals();
      render();
    });
  }

  // Water usage / sewer usage / cost buttons.
  function setupMetricButtons() {
    const buttons = document.querySelectorAll("#metric-buttons button");
    buttons.forEach(button => {
      button.addEventListener("click", () => {
        selectedMetric = button.dataset.metric;
        buttons.forEach(b => b.setAttribute("aria-pressed", String(b === button)));
        const previousFys = selectedFYs();
        buildOptions(previousFys);
        renderFYTotals();
        render();
      });
    });
  }

  // "$1,234.56" -> 1234.56, blank -> NaN
  const toNumber = text => parseFloat((text || "").replace(/[$,]/g, ""));
  // Add a value to a month slot, ignoring blanks.
  function addValue(values, position, value) {
    if (Number.isFinite(value)) values[position] = (values[position] || 0) + value;
  }

  fetch("{{ '/Data/campus_water.csv' | relative_url }}")
    .then(r => {
      if (!r.ok) throw new Error("Campus water data could not be loaded.");
      return r.text();
    })
    .then(text => {
      const rows = parseCSV(text);
      const header = rows[0];
      const ci = header.indexOf("campus");
      const di = header.indexOf("date");
      const fi = header.indexOf("fiscal_year");
      const wui = header.indexOf("water_usage");
      const sui = header.indexOf("sewer_usage");
      const wci = header.indexOf("water_charges");
      const sci = header.indexOf("sewer_charges");
      if ([ci, di, fi, wui, sui, wci, sci].some(index => index < 0)) {
        throw new Error("Campus water data is missing a required column.");
      }
      const fySet = new Set();
      for (const r of rows.slice(1)) {
        const campus = (r[ci] || "").trim();
        const fy = (r[fi] || "").trim().toLowerCase();
        const month = parseInt((r[di] || "").split("-")[1], 10); // dates look like 2022-07-01
        if (!campus || !fy || isNaN(month)) continue;

        byCampus[campus] = byCampus[campus] || {};
        byCampus[campus][fy] = byCampus[campus][fy] || {
          water: new Array(12).fill(null),
          sewer: new Array(12).fill(null),
          cost: new Array(12).fill(null)
        };
        const entry = byCampus[campus][fy];
        const pos = monthIndex(month);

        addValue(entry.water, pos, toNumber(r[wui]));
        addValue(entry.sewer, pos, toNumber(r[sui]));
        // cost = water charges + sewer charges (whichever of the two are filled in)
        const charges = [toNumber(r[wci]), toNumber(r[sci])].filter(Number.isFinite);
        if (charges.length) addValue(entry.cost, pos, charges.reduce((sum, value) => sum + value, 0));
        fySet.add(fy);
      }
      if (!Object.keys(byCampus).length) throw new Error("No usable campus water records were found.");
      allFYs = [...fySet].sort();

      setupCampusPicker();
      setupMetricButtons();
      buildOptions();
      renderFYTotals();
      render();
    })
    .catch(err => {
      console.error(err);
      document.getElementById("chart-msg").textContent = "Failed to load chart: " + err.message;
    });
</script>