---
layout: page
title:  "Compare Accounts"
---

<style>
  /* Hide the page title heading; page.title is still used by the navbar. */
  .post-header { display: none; }
  #acct-chart { max-width: 100%; height: 450px; }
  #acct-msg { padding: 1em 0; color: #6b7280; }
  #acct-filter { width: 100%; max-width: 400px; padding: 4px 8px; }
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
  /* Select all / clear button. */
  #all-accts-btn {
    margin: .5em 0; padding: .45em .9em;
    border: 1px solid #6b7280; border-radius: 6px;
    background: #f9fafb; color: #1f2937; cursor: pointer; font: inherit;
  }
  #all-accts-btn[aria-pressed="true"] {
    border-color: #1d4ed8; background: #dbeafe; color: #1e3a8a; font-weight: 600;
  }
  #all-accts-btn:focus-visible { outline: 2px solid #2a6fb0; outline-offset: 2px; }
  #acct-count { margin-left: .75em; color: #6b7280; }
  /* Scrollable checklist of accounts. */
  #acct-list-box {
    max-height: 240px; overflow-y: auto; margin: .5em 0 1em; padding: 6px 10px;
    border: 1px solid #e3e6ea; border-radius: 6px; background: #ffffff; color: #1f2937;
  }
  #acct-list-box label { display: block; padding: 1px 0; cursor: pointer; }
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
    #page-title { margin-top: 70px; }
    #fy-toggle {
      display: block; padding: 8px 12px; border: 0; border-radius: 4px;
      background: transparent; color: inherit; font: inherit; cursor: pointer;
    }
    #fy-title, #usage-fy { display: none; }
    #fy-float.is-open #usage-fy { display: block; padding: 0 12px 8px; }
  }
  .table-scroll { overflow-x: auto; }
  #acct-totals-table, #acct-usage-table { border-collapse: collapse; margin-top: .5em; }
  #acct-totals-table th, #acct-totals-table td,
  #acct-usage-table th, #acct-usage-table td { padding: 2px 12px; text-align: right; }
  #acct-totals-table th:first-child, #acct-totals-table td:first-child { text-align: left; }
  /* Make the "Show ... table" toggles look like buttons, with an arrow that flips when open. */
  details > summary {
    display: inline-block; list-style: none; cursor: pointer; user-select: none;
    padding: 6px 14px; border: 1px solid currentColor; border-radius: 6px;
    background: rgba(127, 127, 127, 0.12); font-weight: 600; margin-top: .5em;
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

<h1 id="page-title">UH Manoa Compare Accounts</h1>

<div id="metric-buttons" role="group" aria-label="Choose what to compare">
  <button type="button" data-metric="water" aria-pressed="true">Water Usage</button>
  <button type="button" data-metric="sewer" aria-pressed="false">Sewer Usage</button>
  <button type="button" data-metric="cost" aria-pressed="false">Cost (Water + Sewer)</button>
</div>

<div id="fy-float">
  <button id="fy-toggle" type="button" aria-expanded="false" aria-controls="usage-fy">Select FY</button>
  <div id="fy-title"><strong>Fiscal year</strong></div>
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

<div>
  <button type="button" id="all-accts-btn" aria-pressed="false">Select All Accounts</button>
  <span id="acct-count"></span>
</div>
<div>Pick one fiscal year on the left, then check the accounts you want to compare. 
<br>Use the search bar below to search for a certain account in the check list.</div>
<input id="acct-filter" placeholder="Search the account list..." autocomplete="off">
<div id="acct-list-box"></div>
<div id="acct-msg"></div>
<h3 id="acct-usage-heading" hidden></h3>
<div>Each account is its own line.</div>
<div id="acct-chart"></div>
<details>
  <summary>Show totals table</summary>
  <div class="table-scroll"><table id="acct-totals-table"></table></div>
</details>
<details>
  <summary>Show data table</summary>
  <div class="table-scroll"><table id="acct-usage-table"></table></div>
</details>

<script>
  // Fiscal year runs July -> June.
  const MONTHS = ["Jul","Aug","Sep","Oct","Nov","Dec","Jan","Feb","Mar","Apr","May","Jun"];
  const monthIndex = m => (m + 5) % 12; // calendar month (1-12) -> position in fiscal year (0-11)

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

  // What the page can compare. Usage is in thousand gallons (kgal); cost is water + sewer charges.
  const METRICS = {
    water: { label: "Water Usage", money: false },
    sewer: { label: "Sewer Usage", money: false },
    cost:  { label: "Cost (Water + Sewer)", money: true }
  };
  let selectedMetric = "water";
  const metric = () => METRICS[selectedMetric];

  const money = v => v.toLocaleString("en-US", { style: "currency", currency: "USD" });
  // 1,234 for usage, $1,234.56 for cost
  const formatValue = v => metric().money ? money(v) : v.toLocaleString();
  // "$1,234.56" -> 1234.56, "1,234" -> 1234, blank -> NaN
  const toNumber = s => parseFloat((s || "").replace(/[$,]/g, ""));

  const records = []; // every usable CSV row: { fy, month, desc, waterUsage, sewerUsage, waterCharges, sewerCharges }
  let byAcct = {};    // byAcct[description][fy][monthPosition] = summed value for the selected metric
  const fyList = new Set();
  const monthsInFY = {}; // monthsInFY[fy] = Set of calendar months that appear anywhere in that year
  let allFYs = [];       // every fiscal year in the file, oldest first
  let selectedFY = null; // the one fiscal year being compared
  const selectedAccts = new Set(); // account descriptions that are checked

  const acctEl = document.getElementById("acct-chart");
  const acctMsg = document.getElementById("acct-msg");
  const acctFilter = document.getElementById("acct-filter");
  const acctListBox = document.getElementById("acct-list-box");
  const acctCount = document.getElementById("acct-count");
  const allBtn = document.getElementById("all-accts-btn");
  const acctUsageHeading = document.getElementById("acct-usage-heading");
  const acctUsageTable = document.getElementById("acct-usage-table");
  const acctTotalsTable = document.getElementById("acct-totals-table");

  // Value for one record under the selected metric; null when it has no value (blank in the CSV).
  function amountFor(rec) {
    let value;
    if (selectedMetric === "water") value = rec.waterUsage;
    else if (selectedMetric === "sewer") value = rec.sewerUsage;
    else value = (Number.isNaN(rec.waterCharges) && Number.isNaN(rec.sewerCharges))
      ? NaN
      : (rec.waterCharges || 0) + (rec.sewerCharges || 0);
    return Number.isNaN(value) ? null : value;
  }

  // Accounts that have data for the selected fiscal year and metric, alphabetical.
  function listedAccounts() {
    return Object.keys(byAcct)
      .filter(desc => byAcct[desc][selectedFY] && byAcct[desc][selectedFY].some(v => v))
      .sort((a, b) => a.localeCompare(b));
  }

  // Checked accounts that have data right now.
  const plottedAccounts = () => listedAccounts().filter(desc => selectedAccts.has(desc));

  // Rebuild the per-account monthly values from the records for the selected metric.
  function rebuildData() {
    byAcct = {};
    records.forEach(rec => {
      const { fy, month, desc } = rec;
      const amount = amountFor(rec);
      if (amount === null) return;
      byAcct[desc] = byAcct[desc] || {};
      byAcct[desc][fy] = byAcct[desc][fy] || new Array(12).fill(null);
      const pos = monthIndex(month);
      byAcct[desc][fy][pos] = (byAcct[desc][fy][pos] || 0) + amount;
    });
  }

  // A color for each line: the palette for a few lines, evenly spaced hues for many.
  const PALETTE = ["#3b82c4", "#e08a2e", "#3da36b", "#9b6bc9", "#d65a7a", "#8a8f98", "#c9a227", "#2aa5a5", "#a65d3f", "#5b6b2a"];
  function colorFor(i, n) {
    return n <= PALETTE.length ? PALETTE[i] : "hsl(" + Math.round(i * 360 / n) + ", 65%, 45%)";
  }

  // y-axis range shared by water usage and sewer usage for the checked accounts in the selected year,
  // so switching between the two keeps the same scale and the difference is easy to see.
  function usageAxisRange(accounts) {
    const sums = {};
    records.forEach(rec => {
      if (rec.fy !== selectedFY || !accounts.includes(rec.desc)) return;
      const pos = monthIndex(rec.month);
      [["water", rec.waterUsage], ["sewer", rec.sewerUsage]].forEach(([type, value]) => {
        if (Number.isNaN(value)) return;
        const key = rec.desc + "|" + type + "|" + pos;
        sums[key] = (sums[key] || 0) + value;
      });
    });
    const values = Object.values(sums);
    const low = Math.min(0, ...values);
    const high = Math.max(0, ...values);
    return [low, high + (high - low) * 0.05 || 1];
  }

  // Rough height (px) a bottom legend needs when its entries wrap across rows of the given width.
  function legendHeight(names, width) {
    let rows = 1, used = 0;
    names.forEach(name => {
      const w = name.length * 7 + 50; // text + line marker + padding
      if (used + w > width && used > 0) { rows++; used = 0; }
      used += w;
    });
    return rows * 20 + 10;
  }

  // Download a PNG that includes units and a title (and value labels when there are only a few lines).
  // The export is drawn in an off-screen chart so the on-screen chart never flickers.
  async function savePngWithValues(gd, title, filename) {
    const W = 1100;
    // make the picture taller so the legend at the bottom has room
    const legendH = gd.data.length > 1 ? legendHeight(gd.data.map(t => t.name), W - 140) : 0;
    const H = 650 + legendH;
    const many = gd.data.length > 6;
    const tmp = document.createElement("div");
    tmp.style.cssText = "position:fixed; left:-10000px; top:0; width:" + W + "px; height:" + H + "px;";
    document.body.append(tmp);
    try {
      const exportData = gd.data.map((trace, i) => Object.assign({}, trace, {
        mode: many ? "lines+markers" : "lines+markers+text",
        texttemplate: metric().money ? "%{y:$,.0f}" : "%{y:,.0f}",
        textposition: i % 2 ? "bottom center" : "top center"
      }));
      const exportLayout = Object.assign({}, gd.layout, {
        width: W,
        height: H,
        title: { text: title },
        paper_bgcolor: "#ffffff",
        margin: { t: 60, r: 24, b: 80 + legendH, l: 80 },
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

  // Draw one line per entry in series ([{ name, vals, color }]).
  function drawLines(series, usageRange) {
    const ink = "#1f2937";
    const grid = "#3a3f47";
    const isMoney = metric().money;
    const closest = series.length > 6; // many lines: show one line at a time
    // The legend sits under the chart. Grow the chart by however many rows the legend needs,
    // so the plot area itself always stays the same size.
    const legendH = series.length > 1 ? legendHeight(series.map(s => s.name), (acctEl.clientWidth || 740) - 94) : 0;
    acctEl.style.height = (450 + legendH) + "px";

    // null values leave a gap in the line instead of being drawn as zero.
    Plotly.react(acctEl, series.map(s => ({
      x: MONTHS,
      y: s.vals,
      type: "scatter",
      mode: "lines+markers",
      name: s.name,
      line: { color: s.color, width: series.length > 10 ? 1.5 : 2 },
      marker: { size: series.length > 10 ? 5 : 8 },
      connectgaps: false,
      hovertemplate: closest
        ? "<b>%{fullData.name}</b><br>%{x}: " + (isMoney ? "%{y:$,.2f}" : "%{y:,} kgal") + "<extra></extra>"
        : (isMoney ? "%{y:$,.2f}" : "%{y:,} kgal"),
    })), {
      height: 450 + legendH,
      margin: { t: 20, r: 24, b: 40 + legendH, l: 70 },
      paper_bgcolor: "rgba(0,0,0,0)",
      plot_bgcolor: "rgba(0,0,0,0)",
      font: { color: ink },
      hoverlabel: { bgcolor: "#ffffff", bordercolor: "#d1d5db", font: { color: "#1f2937" } },
      // a unified tooltip with many lines is unreadable, so show one line at a time
      hovermode: closest ? "closest" : "x unified",
      showlegend: series.length > 1,
      legend: { orientation: "h", x: 0, xanchor: "left", y: -0.1, yanchor: "top" },
      xaxis: { categoryorder: "array", categoryarray: MONTHS, gridcolor: grid, fixedrange: true },
      yaxis: {
        title: isMoney ? "USD" : "kgal",
        tickprefix: isMoney ? "$" : "",
        rangemode: "tozero",
        gridcolor: grid,
        fixedrange: true,
        ...(usageRange ? { range: usageRange } : {})
      }
    }, {
      responsive: true,
      displaylogo: false,
      // replace the stock camera button so downloads include titles and units
      modeBarButtonsToRemove: ["toImage"],
      modeBarButtonsToAdd: [{
        name: "Download PNG with values",
        title: "Download PNG with values",
        icon: Plotly.Icons.camera,
        click: gd => {
          const title = acctUsageHeading.textContent;
          return savePngWithValues(gd, title, title.replace(/[^\w]+/g, "_"));
        }
      }]
    });
  }

  function makeRow(cells, tag) {
    const row = document.createElement("tr");
    cells.forEach(value => {
      const cell = document.createElement(tag || "td");
      cell.textContent = value;
      row.append(cell);
    });
    return row;
  }

  // Keep the button text and count in step with the checked accounts.
  function updateControls() {
    const listed = listedAccounts();
    const checked = listed.filter(desc => selectedAccts.has(desc)).length;
    const allSelected = listed.length > 0 && checked === listed.length;
    allBtn.textContent = allSelected ? "Clear All Accounts" : "Select All Accounts";
    allBtn.setAttribute("aria-pressed", String(allSelected));
    acctCount.textContent = checked + " of " + listed.length + " selected";
  }

  // The chart, the totals table and the data table for the checked accounts.
  function render() {
    Plotly.purge(acctEl);
    acctUsageTable.replaceChildren();
    acctTotalsTable.replaceChildren();
    updateControls();

    const accounts = plottedAccounts();
    if (!accounts.length) {
      acctUsageHeading.hidden = true;
      acctMsg.textContent = "Check one or more accounts above, or use Select All Accounts.";
      return;
    }

    const label = metric().label;
    const isMoney = metric().money;
    const unit = isMoney ? " (USD)" : " (kgal)";
    acctUsageHeading.textContent = "Monthly " + label + " - " + selectedFY.toUpperCase() +
      " (" + accounts.length + (accounts.length === 1 ? " account)" : " accounts)");
    acctUsageHeading.hidden = false;
    acctMsg.textContent = "";

    const series = accounts.map((desc, i) => ({
      name: desc, vals: byAcct[desc][selectedFY], color: colorFor(i, accounts.length)
    }));
    drawLines(series, isMoney ? null : usageAxisRange(accounts));

    // Totals table: biggest first, with a grand total row.
    const totals = series.map(s => ({ name: s.name, total: s.vals.reduce((sum, v) => sum + (v || 0), 0) }))
      .sort((a, b) => b.total - a.total);
    acctTotalsTable.append(makeRow(["Account", "Total" + unit], "th"));
    totals.forEach(t => acctTotalsTable.append(makeRow([t.name, formatValue(t.total)])));
    if (totals.length > 1) {
      acctTotalsTable.append(makeRow(["All selected", formatValue(totals.reduce((sum, t) => sum + t.total, 0))]));
    }

    // Data table: one column per account.
    acctUsageTable.append(makeRow(["Month", ...series.map(s => s.name + unit)], "th"));
    MONTHS.forEach((month, i) => {
      acctUsageTable.append(makeRow([month, ...series.map(s => s.vals[i] == null ? "–" : formatValue(s.vals[i]))]));
    });
  }

  // The scrollable checklist of accounts, narrowed by the filter box.
  function renderAccountList() {
    const needle = acctFilter.value.trim().toLowerCase();
    acctListBox.replaceChildren();
    const shown = listedAccounts().filter(desc => desc.toLowerCase().includes(needle));
    if (!shown.length) {
      acctListBox.textContent = "No accounts match.";
      return;
    }
    shown.forEach(desc => {
      const label = document.createElement("label");
      const input = document.createElement("input");
      input.type = "checkbox";
      input.value = desc;
      input.checked = selectedAccts.has(desc);
      input.addEventListener("change", () => {
        if (input.checked) selectedAccts.add(desc); else selectedAccts.delete(desc);
        render();
      });
      label.append(input, " " + desc);
      acctListBox.append(label);
    });
  }

  // Radio buttons: one fiscal year at a time, newest first.
  function buildFYOptions() {
    const container = document.getElementById("usage-fy");
    container.replaceChildren();
    allFYs.slice().reverse().forEach(fy => {
      const label = document.createElement("label");
      const input = document.createElement("input");
      input.type = "radio"; input.name = "usage-fy"; input.value = fy;
      input.checked = fy === selectedFY;
      input.addEventListener("change", () => {
        selectedFY = fy;
        renderAccountList();
        render();
      });
      label.append(input, " " + fy.toUpperCase());
      container.append(label);
    });
  }

  // Select All Accounts / Clear All Accounts (applies to every account with data this year).
  function setupAllButton() {
    allBtn.addEventListener("click", () => {
      const listed = listedAccounts();
      const allSelected = listed.length > 0 && listed.every(desc => selectedAccts.has(desc));
      listed.forEach(desc => { if (allSelected) selectedAccts.delete(desc); else selectedAccts.add(desc); });
      acctListBox.querySelectorAll("input[type=checkbox]").forEach(i => { i.checked = selectedAccts.has(i.value); });
      render();
    });
  }

  // Water usage / sewer usage / cost buttons. Checked accounts stay checked.
  function setupMetricButtons() {
    const buttons = document.querySelectorAll("#metric-buttons button");
    buttons.forEach(button => {
      button.addEventListener("click", () => {
        selectedMetric = button.dataset.metric;
        buttons.forEach(b => b.setAttribute("aria-pressed", String(b === button)));
        rebuildData();
        renderAccountList();
        render();
      });
    });
  }

  fetch("{{ '/Data/water_view_new.csv' | relative_url }}")
    .then(r => r.text())
    .then(text => {
      const rows = parseCSV(text);
      const header = rows[0];
      const wui = header.indexOf("water_usage");
      const sui = header.indexOf("sewer_usage");
      const wci = header.indexOf("water_charges_adjusted");
      const sci = header.indexOf("sewer_charges_adjusted");
      const fi = header.indexOf("fiscal_year");
      const di = header.indexOf("date");
      const ai = header.indexOf("description");
      if ([wui, sui, wci, sci, fi, di, ai].some(index => index < 0)) {
        throw new Error("The data file is missing a required column.");
      }
      for (const r of rows.slice(1)) {
        const fy = (r[fi] || "").trim().toLowerCase();
        const desc = (r[ai] || "").trim();
        const month = parseInt((r[di] || "").split("-")[1], 10); // dates look like 2022-7-18
        if (!fy || !desc || isNaN(month)) continue;
        records.push({
          fy, month, desc,
          waterUsage: toNumber(r[wui]),
          sewerUsage: toNumber(r[sui]),
          waterCharges: toNumber(r[wci]),
          sewerCharges: toNumber(r[sci])
        });
        fyList.add(fy);
        monthsInFY[fy] = monthsInFY[fy] || new Set();
        monthsInFY[fy].add(month);
      }
      allFYs = [...fyList].sort();
      // Start on the newest fiscal year that has all 12 months, so a partial year doesn't look sparse.
      const fullYears = allFYs.filter(fy => monthsInFY[fy].size >= 12);
      selectedFY = (fullYears.length ? fullYears : allFYs).slice(-1)[0];

      setupMetricButtons();
      setupAllButton();
      acctFilter.addEventListener("input", renderAccountList);
      rebuildData();
      buildFYOptions();
      renderAccountList();
      render();
    })
    .catch(err => {
      console.error(err);
      acctMsg.textContent = "Failed to load chart: " + err.message;
    });
</script>