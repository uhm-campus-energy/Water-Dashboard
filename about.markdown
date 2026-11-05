---
layout: default
title: About
permalink: /about/
---

<style>
  .about-intro { font-size: 1.1em; max-width: 46em; }
  .about-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(230px, 1fr));
    gap: 1rem;
    margin: 1rem 0 2rem;
  }
  .about-card {
    display: flex; flex-direction: column;
    border: 1px solid #ddd; border-left: 5px solid #2a6fb0; border-radius: 8px;
    padding: 1rem 1.1rem; background: #f9fbfd;
  }
  .about-card h3 { margin: 0 0 .4rem; }
  .about-card p { margin: 0 0 .75rem; flex: 1; }
  .about-card a { font-weight: 600; }
  .about-defs dt { font-weight: 600; margin-top: 1rem; }
  .about-defs dd { margin: .2rem 0 0 0; }
  .about-links a {
    display: inline-block; margin: 0 .75rem .5rem 0; padding: .45em .9em;
    border: 1px solid #2a6fb0; border-radius: 6px; text-decoration: none;
  }
  .about-links a:hover { background: #eef4fb; }
  .about-note { color: #666; font-size: .9em; }
</style>

{% assign campus_page = site.pages | where: "title", "All Campuses" | first %}
{% assign fy_page = site.pages | where: "title", "FY Comparison" | first %}
{% assign acct_page = site.pages | where: "title", "Account Comparison" | first %}

<h1>About This Site</h1>

<p class="about-intro">
  This site makes University of Hawaiʻi water and sewer data easy to explore. It turns monthly billing
  records into charts and tables so you can see how much water each campus uses, what it costs,
  and how that changes from one fiscal year to the next.
</p>

<h2>What you can explore</h2>

<div class="about-grid">
  <div class="about-card">
    <h3>Summary</h3>
    <p>Total water and sewer usage for a fiscal year, what it cost, and the top 10 UH Manoa accounts by charges.</p>
    <a href="{{ '/' | relative_url }}">Open the summary →</a>
  </div>
  <div class="about-card">
    <h3>All Campuses</h3>
    <p>Monthly water usage, sewer usage and charges for any campus, with a fiscal year summary table.</p>
    {% if campus_page %}<a href="{{ campus_page.url | relative_url }}">Open all campuses →</a>{% endif %}
  </div>
  <div class="about-card">
    <h3>FY Comparison</h3>
    <p>Line up fiscal years for a campus month by month, for water usage, sewer usage or total cost.</p>
    {% if fy_page %}<a href="{{ fy_page.url | relative_url }}">Open FY comparison →</a>{% endif %}
  </div>
  <div class="about-card">
    <h3>Account Comparison</h3>
    <p>Pick one UH Manoa account and compare its fiscal years, to spot changes in usage or cost.</p>
    {% if acct_page %}<a href="{{ acct_page.url | relative_url }}">Open account comparison →</a>{% endif %}
  </div>
</div>

<h2>Where the data comes from</h2>

<p>
  The numbers come from monthly water and sewer billing records for UH Manoa and the UH Community Colleges
  (Honolulu, Kapiolani, Leeward, Windward, Hawaiʻi, Maui and Kauaʻi). Each month's usage and charges are
  collected from the billing spreadsheets, checked, and loaded into a database. The charts on this site
  read files exported from that database, and new months are added as billing data comes in.
</p>

<h2>How to read the numbers</h2>

<dl class="about-defs">
  <dt>Fiscal year</dt>
  <dd>Runs July through June and is named for the year it ends. FY26 is July 2025 through June 2026.</dd>

  <dt>Usage</dt>
  <dd>Water and sewer usage are shown in thousands of gallons (kgal).</dd>

  <dt>STARS filter</dt>
  <dd>Limits the summary to accounts flagged for STARS (Sustainability Tracking, Assessment &amp; Rating System) reporting. "Every Account" shows all accounts.</dd>

  <dt>Partial years</dt>
  <dd>The current fiscal year only has the months billed so far, so its totals will be lower than a full year until it is complete.</dd>
</dl>

<h2>Tips</h2>

<ul>
  <li>Use the <strong>Download Excel</strong> button above any table to save it as a spreadsheet. The table's title is in cell A1.</li>
  <li>On the charts, the camera button saves a PNG with the values labeled.</li>
  <li>Click a fiscal year, or "All", in the fiscal year box beside the chart to change what is shown.</li>
</ul>

<h2>Contact</h2>

<p>Questions, corrections or ideas for the site? We would like to hear from you.</p>
<p>Email us at <strong>uhenergy@hawaii.edu</strong>.</p>

<h2>Curious About Sea Grant?</h2>
<p>Look at our webstie as we deal with many other things in addition to water and sewer</P>
<div class="about-links">
  <a href="https://seagrant.soest.hawaii.edu">Sea Grant Website</a>
</div>

<hr>

<p class="about-note">
  Built with <a href="https://jekyllrb.com/">Jekyll</a>, hosted on <a href="https://pages.github.com/">GitHub Pages</a>,
  and charted with <a href="https://plotly.com/javascript/">Plotly</a>.
</p>