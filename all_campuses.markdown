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
</style>

<script src="https://cdn.jsdelivr.net/npm/plotly.js-dist-min@2.35.2/plotly.min.js"></script>

{% include quick-links.html %}

<h2>Monthly Water Usage by Campus</h2>
<p>Select a campus to view its monthly water usage.</p>
<div id="campus-picker" role="radiogroup" aria-label="Select campus"></div>
<div id="campus-chart-message" role="status">Loading campus data...</div>
<div id="campus-chart" aria-label="Monthly water usage line graph"></div>

<script>
  (() => {
    const picker = document.getElementById("campus-picker");
    const chart = document.getElementById("campus-chart");
    const message = document.getElementById("campus-chart-message");
    const campusData = {};
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

    function renderChart() {
      const records = campusData[selectedCampus] || [];
      const x = records.map(record => record.month);
      const y = records.map(record => record.usage);
      Plotly.react(chart, [{
        x,
        y,
        type: "scatter",
        mode: "lines+markers",
        line: { color: "#2563eb", width: 2 },
        marker: { color: "#2563eb", size: 6 },
        hovertemplate: "%{x|%B %Y}<br>%{y:,.0f} thousand gallons<extra></extra>"
      }], {
        title: selectedCampus + " monthly water usage",
        xaxis: { title: "Month", type: "date" },
        yaxis: { title: "Usage (thousand gallons)", rangemode: "tozero" },
        margin: { t: 60, r: 20, b: 70, l: 75 },
        hovermode: "x unified"
      }, { responsive: true, displaylogo: false });
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
        });
        label.append(input, campus);
        picker.append(label);
      });
      selectedCampus = campuses[0];
    }

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
        if ([campusIndex, dateIndex, usageIndex].some(index => index < 0)) {
          throw new Error("Campus water data is missing a required column.");
        }

        rows.slice(1).forEach(row => {
          const campus = (row[campusIndex] || "").trim();
          const month = monthKey(row[dateIndex]);
          const usage = Number((row[usageIndex] || "").replace(/,/g, ""));
          if (!campus || !month || !Number.isFinite(usage)) return;
          campusData[campus] = campusData[campus] || {};
          campusData[campus][month] = (campusData[campus][month] || 0) + usage;
        });

        const campuses = Object.keys(campusData).sort();
        if (!campuses.length) throw new Error("No usable campus water records were found.");
        Object.keys(campusData).forEach(campus => {
          campusData[campus] = Object.entries(campusData[campus])
            .sort(([monthA], [monthB]) => monthA.localeCompare(monthB))
            .map(([month, usage]) => ({ month, usage }));
        });
        renderPicker(campuses);
        message.textContent = "";
        renderChart();
      })
      .catch(error => {
        message.textContent = "Unable to load campus water data: " + error.message;
      });
  })();
</script>
