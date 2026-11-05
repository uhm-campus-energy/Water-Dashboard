---
layout: page
title:  "Account Comparison"
---

<style>
  /* Hide the page title heading; page.title is still used by the navbar. */
  .post-header { display: none; }
  /* Fixed height so the chart never resizes (and pushes the table) when the selection changes. */
  #acct-chart { max-width: 800px; height: 450px; }
  #acct-msg { padding: 1em 0; color: #6b7280; }
  #acct-input { width: 100%; max-width: 400px; padding: 4px 8px; }
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
    #page-title { margin-top: 70px; }
    #fy-toggle {
      display: block; padding: 8px 12px; border: 0; border-radius: 4px;
      background: transparent; color: inherit; font: inherit; cursor: pointer;
    }
    #fy-title, #usage-fy { display: none; }
    #fy-float.is-open #usage-fy { display: block; padding: 0 12px 8px; }
  }
  #acct-fy-totals-table, #acct-usage-table { border-collapse: collapse; margin-top: .5em; }
  #acct-fy-totals-table th, #acct-fy-totals-table td,
  #acct-usage-table th, #acct-usage-table td { padding: 2px 12px; text-align: right; }
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

<h1 id="page-title">UH Manoa Account Comparison</h1>

<div id="metric-buttons" role="group" aria-label="Choose what to compare">
  <button type="button" data-metric="water" aria-pressed="true">Water Usage</button>
  <button type="button" data-metric="sewer" aria-pressed="false">Sewer Usage</button>
  <button type="button" data-metric="cost" aria-pressed="false">Cost (Water + Sewer)</button>
</div>

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

<div>Uses the fiscal years selected on the left. Type to search, then pick one account to compare its fiscal years.</div>
<input id="acct-input" list="acct-list" placeholder="Search for an account..." autocomplete="off">
<datalist id="acct-list"></datalist>
<div id="acct-msg"></div>
<h3 id="acct-fy-heading" hidden></h3>
<details id="acct-fy-details" open hidden>
  <summary id="acct-fy-summary">Hide fiscal year table</summary>
  <table id="acct-fy-totals-table">
    <thead><tr><th>Fiscal Year</th><th id="acct-totals-header">Total (kgal)</th></tr></thead>
    <tbody></tbody>
  </table>
</details>
<h3 id="acct-usage-heading" hidden></h3>
<div id="acct-chart"></div>
<details>
  <summary>Show data table</summary>
  <table id="acct-usage-table"></table>
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
  let byAcct = {}; // byAcct[description][fy][monthPosition] = summed value for the selected metric
  const fyList = new Set();
  let selectedAcct = null; // the one account being viewed
  const acctEl = document.getElementById("acct-chart");
  const acctInput = document.getElementById("acct-input");
  const acctMsg = document.getElementById("acct-msg");
  const acctFYHeading = document.getElementById("acct-fy-heading");
  const acctUsageHeading = document.getElementById("acct-usage-heading");
  const acctFYTotals = document.getElementById("acct-fy-details"); // wraps the fiscal year table (shown by default)
  // The button text flips between "Hide" and "Show" as the table is closed and opened.
  acctFYTotals.addEventListener("toggle", () => {
    document.getElementById("acct-fy-summary").textContent =
      acctFYTotals.open ? "Hide fiscal year table" : "Show fiscal year table";
  });
  const acctFYTotalsBody = document.querySelector("#acct-fy-totals-table tbody");
  const acctUsageTable = document.getElementById("acct-usage-table");

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

  // The selected account's data: data[fy][monthPosition] (empty if it has none for this metric).
  const accountData = () => (selectedAcct && byAcct[selectedAcct]) || {};

  // Fiscal years to offer: those the selected account has data for, or every year if none is selected yet.
  function availableFYs() {
    if (!selectedAcct) return [...fyList].sort();
    const data = accountData();
    return Object.keys(data).filter(fy => data[fy].some(value => value)).sort();
  }

  // How many months of a fiscal year the selected account has data for.
  function monthsWithData(fy) {
    return (accountData()[fy] || []).filter(value => value).length;
  }

  // Fiscal years currently checked, oldest first so the order is stable.
  function selectedFYs() {
    return [...document.querySelectorAll('input[name="usage-fy"]:checked')].map(i => i.value).sort();
  }

  // Each fiscal year keeps its own color no matter which others are checked.
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
        texttemplate: metric().money ? "%{y:$,.0f}" : "%{y:,.0f}",
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

  // y-axis range shared by water usage and sewer usage for the selected account and fiscal years,
  // so switching between the two keeps the same scale and the difference is easy to see.
  function usageAxisRange(fys) {
    const sums = {}; // monthly totals per usage type, fiscal year and month
    records.forEach(rec => {
      if (rec.desc !== selectedAcct || !fys.includes(rec.fy)) return;
      const pos = monthIndex(rec.month);
      [["water", rec.waterUsage], ["sewer", rec.sewerUsage]].forEach(([type, value]) => {
        if (Number.isNaN(value)) return;
        const key = type + "|" + rec.fy + "|" + pos;
        sums[key] = (sums[key] || 0) + value;
      });
    });
    const values = Object.values(sums);
    const low = Math.min(0, ...values);
    const high = Math.max(0, ...values);
    return [low, high + (high - low) * 0.05 || 1];
  }

  // Draw one line per entry in series ([{ name, vals, color }]) into the chart element.
  // usageRange is a fixed [low, high] y-axis range for usage charts (null lets the axis fit itself).
  function drawLines(series, usageRange) {
    // dark mode looks weird
    // const dark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    const ink = "#1f2937";
    const grid = "#3a3f47";
    const isMoney = metric().money;

    // null values leave a gap in the line instead of being drawn as zero.
    Plotly.react(acctEl, series.map(s => ({
      x: MONTHS,
      y: s.vals,
      type: "scatter",
      mode: "lines+markers",
      name: s.name,
      line: { color: s.color, width: 2 },
      marker: { size: 8 },
      connectgaps: false,
      hovertemplate: isMoney ? "%{y:$,.2f}" : "%{y:,} kgal"
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
      // replace the stock camera button so downloads include titles, units and values
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

  // One line per selected fiscal year for the selected account and metric.
  function render() {
    Plotly.purge(acctEl);
    acctUsageTable.replaceChildren();
    acctFYTotalsBody.replaceChildren();

    if (!selectedAcct) {
      acctFYHeading.hidden = true;
      acctFYTotals.hidden = true;
      acctUsageHeading.hidden = true;
      acctMsg.textContent = "Search for an account and pick it to compare its fiscal years.";
      return;
    }

    const label = metric().label;
    const isMoney = metric().money;
    acctFYHeading.textContent = selectedAcct + " - Total " + label + " by Fiscal Year";
    acctUsageHeading.textContent = "Monthly " + label + " - " + selectedAcct;
    document.getElementById("acct-totals-header").textContent = isMoney ? "Total (USD)" : "Total (kgal)";
    acctFYHeading.hidden = false;
    acctFYTotals.hidden = false;
    acctUsageHeading.hidden = false;

    const data = accountData();
    if (!availableFYs().length) {
      acctMsg.textContent = selectedAcct + " has no " + label.toLowerCase() + " data.";
      return;
    }

    // Total for every fiscal year this account has data for, newest first.
    availableFYs().reverse().forEach(fy => {
      const row = document.createElement("tr");
      const total = data[fy].reduce((sum, value) => sum + (value || 0), 0);
      [fy.toUpperCase(), formatValue(total)].forEach(value => {
        const cell = document.createElement("td");
        cell.textContent = value;
        row.append(cell);
      });
      acctFYTotalsBody.append(row);
    });

    const series = selectedFYs().filter(fy => data[fy]).map(fy => ({
      name: fy.toUpperCase(), vals: data[fy], color: colorFor(fy)
    }));
    if (!series.length) { acctMsg.textContent = "Select at least one fiscal year."; return; }
    acctMsg.textContent = "";
    drawLines(series, isMoney ? null : usageAxisRange(selectedFYs()));

    // Table view: one column per selected fiscal year
    const header = document.createElement("tr");
    ["Month", ...series.map(s => s.name + (isMoney ? " (USD)" : " (kgal)"))].forEach(value => {
      const th = document.createElement("th");
      th.textContent = value;
      header.append(th);
    });
    acctUsageTable.append(header);
    MONTHS.forEach((month, i) => {
      const row = document.createElement("tr");
      const monthCell = document.createElement("td");
      monthCell.textContent = month;
      row.append(monthCell);
      series.forEach(s => {
        const cell = document.createElement("td");
        cell.textContent = s.vals[i] == null ? "–" : formatValue(s.vals[i]);
        row.append(cell);
      });
      acctUsageTable.append(row);
    });
  }

  // One checkbox per fiscal year the selected account has data for, newest first. Years that were
  // already checked stay checked (e.g. across a metric or account change); otherwise the newest two
  // full fiscal years (12 months of data) start checked, so a partial year doesn't look sparse.
  function buildOptions(previousFys = []) {
    const container = document.getElementById("usage-fy");
    const available = availableFYs();
    const kept = previousFys.filter(fy => available.includes(fy));
    const full = selectedAcct ? available.filter(fy => monthsWithData(fy) >= 12) : available;
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

  // Pick the account typed in the search box (fires on picking from the list, Enter, or leaving the box).
  function addAccount() {
    const typed = acctInput.value.trim().toLowerCase();
    if (!typed) return;
    const desc = Object.keys(byAcct).find(d => d.toLowerCase() === typed);
    if (!desc) { acctMsg.textContent = "No account matches \"" + acctInput.value + "\"."; return; }
    const previousFys = selectedFYs();
    selectedAcct = desc;
    acctInput.value = "";
    buildOptions(previousFys);
    render();
  }

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

  function renderAccountOptions() {
    const list = document.getElementById("acct-list");
    list.replaceChildren();
    Object.keys(byAcct).sort((a, b) => a.localeCompare(b)).forEach(desc => {
      const option = document.createElement("option");
      option.value = desc;
      list.append(option);
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
        rebuildData();
        renderAccountOptions();
        buildOptions(previousFys);
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
      }
      allFYs = [...fyList].sort();

      setupMetricButtons();
      acctInput.addEventListener("change", addAccount);
      rebuildData();
      renderAccountOptions();
      buildOptions();
      render();
    })
    .catch(err => {
      console.error(err);
      acctMsg.textContent = "Failed to load chart: " + err.message;
    });
</script>