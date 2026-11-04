---
layout: page
title: All Campuses
permalink: /AllCampuses/
---

<style>
  .post-header { display: none; }
  #campus-picker {
    display: flex;
    flex-wrap: wrap;
    gap: .6em;
    margin: 1em 0;
  }
  #campus-picker label {
    cursor: pointer;
    padding: .45em 1em;
    border: 1px solid #9ca3af;
    border-radius: 999px;
    background: #f9fafb;
  }
  #campus-picker label:has(input:checked) {
    border-color: #2563eb;
    background: #dbeafe;
    color: #1e3a8a;
    font-weight: 600;
  }
  #campus-picker input {
    position: absolute;
    opacity: 0;
  }
  #campus-picker label:has(input:focus-visible) {
    outline: 2px solid #2563eb;
    outline-offset: 2px;
  }
  #campus-chart {
    width: 100%;
    max-width: 800px;
    height: 500px;
  }
  #campus-chart-message {
    padding: 1em 0;
    color: #6b7280;
  }
  .controls-row {
    display: flex;
    gap: .5em;
    flex-wrap: wrap;
    margin: .5em 0 0;
  }
  #cost-toggle, #sewer-toggle {
    padding: .45em .9em;
    border: 1px solid #6b7280;
    border-radius: 6px;
    background: #f9fafb;
    cursor: pointer;
    font: inherit;
  }
  #sewer-toggle[aria-pressed="true"] {
    border-color: #047857;
    background: #d1fae5;
    color: #065f46;
    font-weight: 600;
  }
  #cost-toggle[aria-pressed="true"] {
    border-color: #b45309;
    background: #fef3c7;
    color: #78350f;
    font-weight: 600;
  }
  #fiscal-year-table {
    border-collapse: collapse;
    margin: 1em 0;
  }
  #fiscal-year-table th,
  #fiscal-year-table td {
    padding: .4em .8em;
    text-align: right;
  }
  #fiscal-year-table th:first-child,
  #fiscal-year-table td:first-child {
    text-align: left;
  }
  #fiscal-year-table tbody tr:nth-child(even) {
    background: #f3f4f6;
  }
  #go-back-btn {
    display: inline-block;
    margin: .5em 0 0;
    padding: .45em .9em;
    border: 1px solid #6b7280;
    border-radius: 6px;
    background: #f9fafb;
    color: #1f2937;
    text-decoration: none;
    font: inherit;
  }
  #go-back-btn:hover {
    background: #e5e7eb;
  }

  /* Floating fiscal year picker that stays beside the chart while scrolling.
     The theme's content column is 740px wide and centered, so anchor the box's
     right edge 16px left of that column instead of the screen edge. */
  #fy-float {
    position: fixed; top: 90px; right: calc(50% + 370px + 16px); z-index: 10;
    background: #ffffff; color: #1f2937;
    border: 1px solid #e3e6ea; border-radius: 6px; padding: 8px 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
  }
  #campus-fy label { display: block; }
  #fy-toggle { display: none; }
  #fy-toggle:focus-visible { outline: 2px solid #2a6fb0; outline-offset: 2px; }
  /* On narrow screens, collapse the picker below the navbar to keep it out of the way. */
  @media (max-width: 1100px) {
    #fy-float { top: 72px; left: 12px; right: auto; padding: 6px; }
    #go-back-btn { margin-top: 3.5em; }
    #fy-toggle {
      display: block; padding: 8px 12px; border: 0; border-radius: 4px;
      background: transparent; color: inherit; font: inherit; cursor: pointer;
    }
    #fy-title, #campus-fy { display: none; }
    #fy-float.is-open #campus-fy { display: block; padding: 0 12px 8px; }
  }
</style>

{% include quick-links.html %}
{% include table-export.html %}

<script src="https://cdn.jsdelivr.net/npm/plotly.js-dist-min@2.35.2/plotly.min.js"></script>

<a href="{{ '/' | relative_url }}" id="go-back-btn">Go Back</a>
<h2>Monthly Usage by Campus</h2>
<p>Select a campus to view its monthly water and sewer metrics.</p>
<div id="campus-picker" role="radiogroup" aria-label="Select campus"></div>
<div class="controls-row">
  <button id="sewer-toggle" type="button" aria-pressed="false">Show sewer usage</button>
  <button id="cost-toggle" type="button" aria-pressed="false">Show monthly costs</button>
</div>
<div id="fy-float">
  <button id="fy-toggle" type="button" aria-expanded="false" aria-controls="campus-fy">Select FY</button>
  <div id="fy-title"><strong>Fiscal years</strong></div>
  <div id="campus-fy"></div>
</div>
<div id="campus-chart-message" role="status">Loading campus data...</div>
<div id="campus-chart" aria-label="Monthly water and sewer usage line graph"></div>

<h2 id="fiscal-year-heading">Fiscal Year Summary</h2>
<table id="fiscal-year-table">
  <thead>
    <tr>
      <th>Fiscal Year</th>
      <th>Water Usage (thousand gallons)</th>
      <th>Sewer Usage (thousand gallons)</th>
      <th>Water Charges</th>
      <th>Sewer Charges</th>
    </tr>
  </thead>
  <tbody></tbody>
</table>

<script>
  (() => {
    const picker = document.getElementById("campus-picker");
    const chart = document.getElementById("campus-chart");
    const message = document.getElementById("campus-chart-message");
    const sewerToggle = document.getElementById("sewer-toggle");
    const costToggle = document.getElementById("cost-toggle");
    const fiscalHeading = document.getElementById("fiscal-year-heading");
    const fiscalTableHead = document.querySelector("#fiscal-year-table thead");
    const fiscalTableBody = document.querySelector("#fiscal-year-table tbody");
    const fyFloat = document.getElementById("fy-float");
    const fyToggle = document.getElementById("fy-toggle");
    const fyContainer = document.getElementById("campus-fy");
    // Monthly totals are stored as store[campus][fiscalYear][month] = value, so every
    // month stays tied to the fiscal year of the row it came from.
    const campusWaterData = {};
    const campusSewerData = {};
    const monthlyWaterCostData = {};
    const monthlySewerCostData = {};
    const fiscalData = {};
    let selectedCampus;

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
          row.push(field); rows.push(row); row = []; field = "";
        } else field += c;
      }
      if (field !== "" || row.length) { row.push(field); rows.push(row); }
      return rows;
    }

    function monthKey(date) {
      const parsed = new Date(date);
      if (Number.isNaN(parsed.getTime())) return null;
      return parsed.toISOString().slice(0, 7);
    }

    function parseAmount(value) {
      const amount = Number((value || "").replace(/[$,]/g, ""));
      return Number.isFinite(amount) ? amount : 0;
    }

    function formatNumber(value) {
      return value.toLocaleString("en-US", { maximumFractionDigits: 0 });
    }

    function formatCurrency(value) {
      return value.toLocaleString("en-US", {
        style: "currency",
        currency: "USD"
      });
    }

    // Add an amount to store[campus][fiscalYear][month].
    function addMonthly(store, campus, fiscalYear, month, amount) {
      store[campus] = store[campus] || {};
      store[campus][fiscalYear] = store[campus][fiscalYear] || {};
      store[campus][fiscalYear][month] = (store[campus][fiscalYear][month] || 0) + amount;
    }

    // The selected campus's months for the given fiscal years, oldest first: [{ month, value }].
    // Pass one store, or an array of stores to add them together (e.g. water + sewer charges).
    function monthlySeries(stores, fys) {
      const byMonth = {};
      (Array.isArray(stores) ? stores : [stores]).forEach(store => {
        const byFY = store[selectedCampus] || {};
        fys.forEach(fy => {
          Object.entries(byFY[fy] || {}).forEach(([month, value]) => {
            byMonth[month] = (byMonth[month] || 0) + value;
          });
        });
      });
      return Object.entries(byMonth)
        .sort(([monthA], [monthB]) => monthA.localeCompare(monthB))
        .map(([month, value]) => ({ month, value }));
    }

    // The selected bubble: one fiscal year, or every fiscal year this campus has data for.
    function selectedFYs() {
      const chosen = document.querySelector('input[name="campus-fy"]:checked');
      if (!chosen) return [];
      return chosen.value === "all" ? campusFYs() : [chosen.value];
    }

    // Only the fiscal years that have data for the selected campus, oldest first.
    // A year counts only if it has some non-zero usage or charges, so blank or
    // all-zero placeholder rows in the CSV don't add a bubble.
    function campusFYs() {
      const records = fiscalData[selectedCampus] || {};
      return Object.keys(records)
        .filter(fy => {
          const r = records[fy];
          return r.waterUsage || r.sewerUsage || r.water || r.sewer;
        })
        .sort();
    }

    function renderChart() {
      const fys = selectedFYs();
      if (!fys.length) {
        Plotly.purge(chart);
        message.textContent = "Select at least one fiscal year.";
        return;
      }
      message.textContent = "";
      let costAxisRange;

      const waterRecords = monthlySeries(campusWaterData, fys);
      const traces = [
        {
          x: waterRecords.map(record => record.month),
          y: waterRecords.map(record => record.value),
          type: "scatter",
          mode: "lines+markers",
          name: "Water usage",
          line: { color: "#2563eb", width: 2 },
          marker: { color: "#2563eb", size: 6 },
          hovertemplate: "%{y:,.0f} thousand gallons<extra>Water usage</extra>"
        }
      ];

      if (sewerToggle.getAttribute("aria-pressed") === "true") {
        const sewerRecords = monthlySeries(campusSewerData, fys);
        traces.push({
          x: sewerRecords.map(record => record.month),
          y: sewerRecords.map(record => record.value),
          type: "scatter",
          mode: "lines+markers",
          name: "Sewer usage",
          line: { color: "#059669", width: 2 },
          marker: { color: "#059669", size: 6 },
          hovertemplate: "%{y:,.0f} thousand gallons<extra>Sewer usage</extra>"
        });
      }

      if (costToggle.getAttribute("aria-pressed") === "true") {
        // Water charges only, unless sewer usage is on the graph: then water + sewer charges.
        const showSewer = sewerToggle.getAttribute("aria-pressed") === "true";
        const costLabel = showSewer ? "Monthly cost (water + sewer)" : "Monthly cost (water)";
        const costRecords = monthlySeries(
          showSewer ? [monthlyWaterCostData, monthlySewerCostData] : [monthlyWaterCostData], fys);
        // Fix the cost axis using both water-only and water + sewer values, so it
        // doesn't rescale when sewer usage is toggled on or off.
        const waterOnlyValues = monthlySeries([monthlyWaterCostData], fys).map(r => r.value);
        const combinedValues = monthlySeries([monthlyWaterCostData, monthlySewerCostData], fys).map(r => r.value);
        const allCostValues = [0, ...waterOnlyValues, ...combinedValues];
        const costLow = Math.min(...allCostValues);
        const costHigh = Math.max(...allCostValues);
        const costPad = (costHigh - costLow) * 0.05 || 1;
        costAxisRange = [costLow < 0 ? costLow - costPad : 0, costHigh + costPad];
        traces.push({
          x: costRecords.map(record => record.month),
          y: costRecords.map(record => record.value),
          type: "scatter",
          mode: "lines+markers",
          name: costLabel,
          yaxis: "y2",
          line: { color: "#d97706", width: 2 },
          marker: { color: "#d97706", size: 6 },
          hovertemplate: "$%{y:,.2f}<extra>" + costLabel + "</extra>"
        });
      }

      Plotly.react(chart, traces, {
        title: selectedCampus + " monthly usage",
        xaxis: { title: "Month", type: "date", hoverformat: "%B %Y" },
        yaxis: { title: "Usage (thousand gallons)", rangemode: "tozero" },
        yaxis2: {
          title: {text: "Monthly charges (USD)", standoff: 10},
          overlaying: "y",
          side: "right",
          rangemode: "tozero",
          showgrid: false,
          automargin: true,
          ...(costAxisRange ? { range: costAxisRange } : {})
        },
        legend: { orientation: "h", x: 0.5, xanchor: "center", y: -0.25, yanchor: "top" },
        margin: { t: 60, r: 20, b: 110, l: 75 },
        hovermode: "x unified"
      }, { responsive: true, displaylogo: false });
    }

    // Fill the summary table: a header row, then one row of cell values per entry in rows.
    function setSummaryTable(firstColumn, rows) {
      const headRow = document.createElement("tr");
      [
        firstColumn,
        "Water Usage (thousand gallons)",
        "Sewer Usage (thousand gallons)",
        "Water Charges",
        "Sewer Charges"
      ].forEach(text => {
        const th = document.createElement("th");
        th.textContent = text;
        headRow.append(th);
      });
      fiscalTableHead.replaceChildren(headRow);

      fiscalTableBody.replaceChildren();
      rows.forEach(values => {
        const row = document.createElement("tr");
        values.forEach(value => {
          const cell = document.createElement("td");
          cell.textContent = value;
          row.append(cell);
        });
        fiscalTableBody.append(row);
      });
    }

    // "2025-07" -> "Jul 2025"
    function formatMonth(month) {
      return new Date(month + "-01").toLocaleDateString("en-US", {
        month: "short", year: "numeric", timeZone: "UTC"
      });
    }

    // One fiscal year selected: a row per month. "All" selected: a row per fiscal year.
    function renderFiscalYearTable() {
      const chosen = document.querySelector('input[name="campus-fy"]:checked');
      const fys = selectedFYs();

      if (chosen && chosen.value !== "all" && fys.length === 1) {
        const fy = fys[0];
        const monthsOf = store => (store[selectedCampus] || {})[fy] || {};
        const water = monthsOf(campusWaterData);
        const sewer = monthsOf(campusSewerData);
        const waterCharges = monthsOf(monthlyWaterCostData);
        const sewerCharges = monthsOf(monthlySewerCostData);
        const months = [...new Set([
          ...Object.keys(water), ...Object.keys(sewer),
          ...Object.keys(waterCharges), ...Object.keys(sewerCharges)
        ])].sort();

        fiscalHeading.textContent = selectedCampus + " " + fy.toUpperCase() + " monthly summary";
        setSummaryTable("Month", months.map(month => [
          formatMonth(month),
          formatNumber(water[month] || 0),
          formatNumber(sewer[month] || 0),
          formatCurrency(waterCharges[month] || 0),
          formatCurrency(sewerCharges[month] || 0)
        ]));
        return;
      }

      const records = fiscalData[selectedCampus] || {};
      fiscalHeading.textContent = selectedCampus + " fiscal year summary";
      setSummaryTable("Fiscal Year", Object.keys(records)
        .filter(fy => fys.includes(fy)).sort().reverse()
        .map(fiscalYear => [
          fiscalYear.toUpperCase(),
          formatNumber(records[fiscalYear].waterUsage),
          formatNumber(records[fiscalYear].sewerUsage),
          formatCurrency(records[fiscalYear].water),
          formatCurrency(records[fiscalYear].sewer)
        ]));
    }

    function renderAll() {
      renderChart();
      renderFiscalYearTable();
    }

    // One bubble (radio button) for "All" plus one per fiscal year that has data for the
    // selected campus, newest first. When switching campuses the current choice is kept if
    // the new campus has it ("All" always carries over); otherwise the newest year is selected.
    function buildFYOptions() {
      const available = campusFYs();
      const previous = document.querySelector('input[name="campus-fy"]:checked');
      const previousValue = previous ? previous.value : null;
      const selected = previousValue === "all" || available.includes(previousValue)
        ? previousValue
        : available[available.length - 1];
      fyContainer.replaceChildren();

      const options = available.length ? ["all", ...available.slice().reverse()] : [];
      options.forEach(value => {
        const label = document.createElement("label");
        const input = document.createElement("input");
        input.type = "radio"; input.name = "campus-fy"; input.value = value;
        input.checked = value === selected;
        input.addEventListener("change", renderAll);
        label.append(input, " " + (value === "all" ? "All" : value.toUpperCase()));
        fyContainer.append(label);
      });
    }

    function renderPicker(campuses) {
      campuses.forEach((campus, index) => {
        const label = document.createElement("label");
        const input = document.createElement("input");
        input.type = "radio";
        input.name = "campus";
        input.value = campus;
        input.checked = index === 0;
        input.addEventListener("change", () => {
          selectedCampus = input.value;
          buildFYOptions();
          renderAll();
        });
        label.append(input, campus);
        picker.append(label);
      });
      selectedCampus = campuses[0];
    }

    sewerToggle.addEventListener("click", () => {
      const isPressed = sewerToggle.getAttribute("aria-pressed") === "true";
      sewerToggle.setAttribute("aria-pressed", String(!isPressed));
      sewerToggle.textContent = isPressed ? "Show sewer usage" : "Hide sewer usage";
      renderChart();
    });

    costToggle.addEventListener("click", () => {
      const isPressed = costToggle.getAttribute("aria-pressed") === "true";
      costToggle.setAttribute("aria-pressed", String(!isPressed));
      costToggle.textContent = isPressed ? "Show monthly charges" : "Hide monthly charges";
      renderChart();
    });

    // On narrow screens the fiscal year list is collapsed behind a "Select FY" button.
    fyToggle.addEventListener("click", () => {
      const open = fyToggle.getAttribute("aria-expanded") !== "true";
      fyFloat.classList.toggle("is-open", open);
      fyToggle.setAttribute("aria-expanded", String(open));
    });

    fetch("{{ '/Data/campus_water.csv' | relative_url }}")
      .then(response => {
        if (!response.ok) throw new Error("Campus water data could not be loaded.");
        return response.text();
      })
      .then(text => {
        const rows = parseCSV(text);
        if (!rows.length) throw new Error("Campus water data is empty.");
        const header = rows[0];
        const campusIndex = header.indexOf("campus");
        const dateIndex = header.indexOf("date");
        const waterUsageIndex = header.indexOf("water_usage");
        const sewerUsageIndex = header.indexOf("sewer_usage");
        const waterChargesIndex = header.indexOf("water_charges");
        const sewerChargesIndex = header.indexOf("sewer_charges");
        const fiscalYearIndex = header.indexOf("fiscal_year");
        
        if ([campusIndex, dateIndex, waterUsageIndex, sewerUsageIndex, waterChargesIndex, sewerChargesIndex, fiscalYearIndex]
          .some(index => index < 0)) {
          throw new Error("Campus water data is missing a required column.");
        }

        rows.slice(1).forEach(row => {
          const campus = (row[campusIndex] || "").trim();
          const month = monthKey(row[dateIndex]);
          const rawWaterUsage = Number((row[waterUsageIndex] || "").replace(/,/g, ""));
          const rawSewerUsage = Number((row[sewerUsageIndex] || "").replace(/,/g, ""));
          const fiscalYear = (row[fiscalYearIndex] || "").trim().toLowerCase();
        

          const waterUsage = Number.isFinite(rawWaterUsage) ? rawWaterUsage : 0;
          const sewerUsage = Number.isFinite(rawSewerUsage) ? rawSewerUsage : 0;
          const waterCharges = parseAmount(row[waterChargesIndex]);
          const sewerCharges = parseAmount(row[sewerChargesIndex]);

          addMonthly(campusWaterData, campus, fiscalYear, month, waterUsage);
          addMonthly(campusSewerData, campus, fiscalYear, month, sewerUsage);
          addMonthly(monthlyWaterCostData, campus, fiscalYear, month, waterCharges);
          addMonthly(monthlySewerCostData, campus, fiscalYear, month, sewerCharges);

          fiscalData[campus] = fiscalData[campus] || {};
          fiscalData[campus][fiscalYear] = fiscalData[campus][fiscalYear] || {
            waterUsage: 0, sewerUsage: 0, water: 0, sewer: 0
          };
          fiscalData[campus][fiscalYear].waterUsage += waterUsage;
          fiscalData[campus][fiscalYear].sewerUsage += sewerUsage;
          fiscalData[campus][fiscalYear].water += waterCharges;
          fiscalData[campus][fiscalYear].sewer += sewerCharges;
        });

        const campuses = Object.keys(campusWaterData).sort();
        if (!campuses.length) throw new Error("No usable campus water records were found.");

        renderPicker(campuses);
        buildFYOptions();
        message.textContent = "";
        renderAll();
      })
      .catch(error => {
        message.textContent = "Unable to load campus water data: " + error.message;
      });
  })();
</script>