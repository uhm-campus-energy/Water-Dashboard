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
  #cost-toggle {
    margin: .5em 0 0;
    padding: .45em .9em;
    border: 1px solid #6b7280;
    border-radius: 6px;
    background: #f9fafb;
    cursor: pointer;
    font: inherit;
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
</style>

<script src="https://cdn.jsdelivr.net/npm/plotly.js-dist-min@2.35.2/plotly.min.js"></script>

<h2>Monthly Water Usage by Campus</h2>
<p>Select a campus to view its monthly water usage.</p>
<div id="campus-picker" role="radiogroup" aria-label="Select campus"></div>
<button id="cost-toggle" type="button" aria-pressed="false">Show total cost</button>
<div id="campus-chart-message" role="status">Loading campus data...</div>
<div id="campus-chart" aria-label="Monthly water usage line graph"></div>

<h2 id="fiscal-year-heading">Fiscal Year Summary</h2>
<table id="fiscal-year-table">
  <thead>
    <tr>
      <th>Fiscal Year</th>
      <th>Usage (thousand gallons)</th>
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
    const costToggle = document.getElementById("cost-toggle");
    const fiscalHeading = document.getElementById("fiscal-year-heading");
    const fiscalTableBody = document.querySelector("#fiscal-year-table tbody");
    const campusData = {};
    const monthlyCostData = {};
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

    function renderChart() {
      const records = campusData[selectedCampus] || [];
      const x = records.map(record => record.month);
      const y = records.map(record => record.usage);
      const traces = [{
        x,
        y,
        type: "scatter",
        mode: "lines+markers",
        line: { color: "#2563eb", width: 2 },
        marker: { color: "#2563eb", size: 6 },
        hovertemplate: "%{x|%B %Y}<br>%{y:,.0f} thousand gallons<extra></extra>"
      }];
      if (costToggle.getAttribute("aria-pressed") === "true") {
        const costRecords = monthlyCostData[selectedCampus] || [];
        traces.push({
          x: costRecords.map(record => record.month),
          y: costRecords.map(record => record.cost),
          type: "scatter",
          mode: "lines+markers",
          name: "Total cost",
          yaxis: "y2",
          line: { color: "#d97706", width: 2 },
          marker: { color: "#d97706", size: 6 },
          hovertemplate: "%{x|%B %Y}<br>$%{y:,.2f}<extra>Total cost</extra>"
        });
      }
      Plotly.react(chart, traces, {
        title: selectedCampus + " monthly water usage",
        xaxis: { title: "Month", type: "date" },
        yaxis: { title: "Usage (thousand gallons)", rangemode: "tozero" },
        yaxis2: {
          title: "Total cost (USD)",
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
      const records = fiscalData[selectedCampus] || {};
      fiscalHeading.textContent = selectedCampus + " fiscal year summary";
      fiscalTableBody.replaceChildren();
      Object.keys(records).sort().reverse().forEach(fiscalYear => {
        const record = records[fiscalYear];
        const row = document.createElement("tr");
        [
          fiscalYear.toUpperCase(),
          formatNumber(record.usage),
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
          renderChart();
          renderFiscalYearTable();
        });
        label.append(input, campus);
        picker.append(label);
      });
      selectedCampus = campuses[0];
    }

    costToggle.addEventListener("click", () => {
      const isPressed = costToggle.getAttribute("aria-pressed") === "true";
      costToggle.setAttribute("aria-pressed", String(!isPressed));
      costToggle.textContent = isPressed ? "Show total cost" : "Hide total cost";
      renderChart();
    });

    fetch("{{ '/campus_water.csv' | relative_url }}")
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
        const usageIndex = header.indexOf("thousands_gal");
        const waterChargesIndex = header.indexOf("water_charges");
        const sewerChargesIndex = header.indexOf("sewer_charges");
        const fiscalYearIndex = header.indexOf("fiscal_year");
        if ([campusIndex, dateIndex, usageIndex, waterChargesIndex, sewerChargesIndex, fiscalYearIndex]
          .some(index => index < 0)) {
          throw new Error("Campus water data is missing a required column.");
        }

        rows.slice(1).forEach(row => {
          const campus = (row[campusIndex] || "").trim();
          const month = monthKey(row[dateIndex]);
          const usage = Number((row[usageIndex] || "").replace(/,/g, ""));
          const fiscalYear = (row[fiscalYearIndex] || "").trim().toLowerCase();
          if (!campus || !month || !fiscalYear || !Number.isFinite(usage)) return;
          campusData[campus] = campusData[campus] || {};
          campusData[campus][month] = (campusData[campus][month] || 0) + usage;
          monthlyCostData[campus] = monthlyCostData[campus] || {};
          monthlyCostData[campus][month] = (monthlyCostData[campus][month] || 0)
            + parseAmount(row[waterChargesIndex]) + parseAmount(row[sewerChargesIndex]);
          fiscalData[campus] = fiscalData[campus] || {};
          fiscalData[campus][fiscalYear] = fiscalData[campus][fiscalYear] || {
            usage: 0, water: 0, sewer: 0
          };
          fiscalData[campus][fiscalYear].usage += usage;
          fiscalData[campus][fiscalYear].water += parseAmount(row[waterChargesIndex]);
          fiscalData[campus][fiscalYear].sewer += parseAmount(row[sewerChargesIndex]);
        });

        const campuses = Object.keys(campusData).sort();
        if (!campuses.length) throw new Error("No usable campus water records were found.");
        Object.keys(campusData).forEach(campus => {
          campusData[campus] = Object.entries(campusData[campus])
            .sort(([monthA], [monthB]) => monthA.localeCompare(monthB))
            .map(([month, usage]) => ({ month, usage }));
          monthlyCostData[campus] = Object.entries(monthlyCostData[campus] || {})
            .sort(([monthA], [monthB]) => monthA.localeCompare(monthB))
            .map(([month, cost]) => ({ month, cost }));
        });
        renderPicker(campuses);
        message.textContent = "";
        renderChart();
        renderFiscalYearTable();
      })
      .catch(error => {
        message.textContent = "Unable to load campus water data: " + error.message;
      });
  })();
</script>
