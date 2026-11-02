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
    const fiscalTableBody = document.querySelector("#fiscal-year-table tbody");
    const fyFloat = document.getElementById("fy-float");
    const fyToggle = document.getElementById("fy-toggle");
    const fyContainer = document.getElementById("campus-fy");
    const campusWaterData = {};
    const campusSewerData = {};
    const monthlyCostData = {};
    const fiscalData = {};
    const monthFY = {}; // "2024-07" -> "fy25"
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

    // Fiscal years currently checked, oldest first so the order is stable.
    function selectedFYs() {
      return [...document.querySelectorAll('input[name="campus-fy"]:checked')].map(i => i.value).sort();
    }

    // Only the fiscal years that have data for the selected campus, oldest first.
    // A year counts only if it has some non-zero usage or charges, so blank or
    // all-zero placeholder rows in the CSV don't add a checkbox.
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
      const inSelectedFY = record => fys.includes(monthFY[record.month]);

      const waterRecords = (campusWaterData[selectedCampus] || []).filter(inSelectedFY);
      const traces = [
        {
          x: waterRecords.map(record => record.month),
          y: waterRecords.map(record => record.usage),
          type: "scatter",
          mode: "lines+markers",
          name: "Water usage",
          line: { color: "#2563eb", width: 2 },
          marker: { color: "#2563eb", size: 6 },
          hovertemplate: "%{y:,.0f} thousand gallons<extra>Water usage</extra>"
        }
      ];

      if (sewerToggle.getAttribute("aria-pressed") === "true") {
        const sewerRecords = (campusSewerData[selectedCampus] || []).filter(inSelectedFY);
        traces.push({
          x: sewerRecords.map(record => record.month),
          y: sewerRecords.map(record => record.usage),
          type: "scatter",
          mode: "lines+markers",
          name: "Sewer usage",
          line: { color: "#059669", width: 2 },
          marker: { color: "#059669", size: 6 },
          hovertemplate: "%{y:,.0f} thousand gallons<extra>Sewer usage</extra>"
        });
      }

      if (costToggle.getAttribute("aria-pressed") === "true") {
        const costRecords = (monthlyCostData[selectedCampus] || []).filter(inSelectedFY);
        traces.push({
          x: costRecords.map(record => record.month),
          y: costRecords.map(record => record.cost),
          type: "scatter",
          mode: "lines+markers",
          name: "Monthly cost",
          yaxis: "y2",
          line: { color: "#d97706", width: 2 },
          marker: { color: "#d97706", size: 6 },
          hovertemplate: "$%{y:,.2f}<extra>Monthly cost</extra>"
        });
      }

      Plotly.react(chart, traces, {
        title: selectedCampus + " monthly usage",
        xaxis: { title: "Month", type: "date", hoverformat: "%B %Y" },
        yaxis: { title: "Usage (thousand gallons)", rangemode: "tozero" },
        yaxis2: {
          title: "Monthly cost (USD)",
          overlaying: "y",
          side: "right",
          rangemode: "tozero",
          showgrid: false
        },
        margin: { t: 60, r: 20, b: 70, l: 75 },
        hovermode: "x unified"
      }, { responsive: true, displaylogo: false });
    }

    function renderFiscalYearTable() {
      const fys = selectedFYs();
      const records = fiscalData[selectedCampus] || {};
      fiscalHeading.textContent = selectedCampus + " fiscal year summary";
      fiscalTableBody.replaceChildren();
      Object.keys(records).filter(fy => fys.includes(fy)).sort().reverse().forEach(fiscalYear => {
        const record = records[fiscalYear];
        const row = document.createElement("tr");
        [
          fiscalYear.toUpperCase(),
          formatNumber(record.waterUsage),
          formatNumber(record.sewerUsage),
          formatCurrency(record.water),
          formatCurrency(record.sewer)
        ].forEach(value => {
          const cell = document.createElement("td");
          cell.textContent = value;
          row.append(cell);
        });
        fiscalTableBody.append(row);
      });
    }

    function renderAll() {
      renderChart();
      renderFiscalYearTable();
    }

    // One checkbox per fiscal year that has data for the selected campus, newest first.
    // Years that were already checked stay checked when switching campuses; if none of them
    // exist for the new campus, the newest two start checked.
    function buildFYOptions() {
      const available = campusFYs();
      const kept = selectedFYs().filter(fy => available.includes(fy));
      const checked = kept.length ? kept : available.slice(-2);
      fyContainer.replaceChildren();

      // "All" checks or clears every year; it is checked whenever every year is checked.
      // It has its own name so selectedFYs() doesn't count it as a fiscal year.
      const allLabel = document.createElement("label");
      const allInput = document.createElement("input");
      allInput.type = "checkbox"; allInput.name = "campus-fy-all";
      allLabel.append(allInput, " All");
      fyContainer.append(allLabel);
      const yearInputs = () => [...fyContainer.querySelectorAll('input[name="campus-fy"]')];
      const syncAll = () => { allInput.checked = yearInputs().every(i => i.checked); };
      allInput.addEventListener("change", () => {
        yearInputs().forEach(i => { i.checked = allInput.checked; });
        renderAll();
      });

      available.slice().reverse().forEach(fy => {
        const label = document.createElement("label");
        const input = document.createElement("input");
        input.type = "checkbox"; input.name = "campus-fy"; input.value = fy;
        input.checked = checked.includes(fy);
        input.addEventListener("change", () => { syncAll(); renderAll(); });
        label.append(input, " " + fy.toUpperCase());
        fyContainer.append(label);
      });
      syncAll();
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
      costToggle.textContent = isPressed ? "Show total cost" : "Hide total cost";
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
          
          if (!campus || !month || !fiscalYear) return;

          monthFY[month] = fiscalYear;

          const waterUsage = Number.isFinite(rawWaterUsage) ? rawWaterUsage : 0;
          const sewerUsage = Number.isFinite(rawSewerUsage) ? rawSewerUsage : 0;

          campusWaterData[campus] = campusWaterData[campus] || {};
          campusWaterData[campus][month] = (campusWaterData[campus][month] || 0) + waterUsage;

          campusSewerData[campus] = campusSewerData[campus] || {};
          campusSewerData[campus][month] = (campusSewerData[campus][month] || 0) + sewerUsage;

          monthlyCostData[campus] = monthlyCostData[campus] || {};
          monthlyCostData[campus][month] = (monthlyCostData[campus][month] || 0)
            + parseAmount(row[waterChargesIndex]) + parseAmount(row[sewerChargesIndex]);

          fiscalData[campus] = fiscalData[campus] || {};
          fiscalData[campus][fiscalYear] = fiscalData[campus][fiscalYear] || {
            waterUsage: 0, sewerUsage: 0, water: 0, sewer: 0
          };
          fiscalData[campus][fiscalYear].waterUsage += waterUsage;
          fiscalData[campus][fiscalYear].sewerUsage += sewerUsage;
          fiscalData[campus][fiscalYear].water += parseAmount(row[waterChargesIndex]);
          fiscalData[campus][fiscalYear].sewer += parseAmount(row[sewerChargesIndex]);
        });

        const campuses = Object.keys(campusWaterData).sort();
        if (!campuses.length) throw new Error("No usable campus water records were found.");

        Object.keys(campusWaterData).forEach(campus => {
          campusWaterData[campus] = Object.entries(campusWaterData[campus])
            .sort(([monthA], [monthB]) => monthA.localeCompare(monthB))
            .map(([month, usage]) => ({ month, usage }));
          campusSewerData[campus] = Object.entries(campusSewerData[campus] || {})
            .sort(([monthA], [monthB]) => monthA.localeCompare(monthB))
            .map(([month, usage]) => ({ month, usage }));
          monthlyCostData[campus] = Object.entries(monthlyCostData[campus] || {})
            .sort(([monthA], [monthB]) => monthA.localeCompare(monthB))
            .map(([month, cost]) => ({ month, cost }));
        });

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