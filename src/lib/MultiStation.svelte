<script>
  import * as d3 from "d3";
  import { onMount } from "svelte";

  export let data = [];

  let currentPlot = "Main Pollutants by Year";
  const plots = [
    "Main Pollutants by Year",
    "AQI Scatter by Date",
    "Mean AQI per Pollutant",
    "Monthly Mean AQI",
    "COVID Period AQI"
  ];

  const plotDescriptions = {
    "Main Pollutants by Year":
      "This chart shows how the dominant air pollutants have shifted over the years. PM2.5 consistently dominates across stations, while Ozone and SO2 appear sporadically in certain years.",
    "AQI Scatter by Date":
      "Each point represents the AQI reading for a pollutant on a given date. The scatter helps identify spikes, seasonal trends, and years with abnormal air quality variations.",
    "Mean AQI per Pollutant":
      "This line graph tracks the monthly average AQI for each pollutant. It highlights gradual improvements or deteriorations in specific pollutants over time.",
    "Monthly Mean AQI":
      "This bar chart summarizes how AQI typically varies through the months of the year. It reveals recurring seasonal patterns that may correspond to weather or human activity.",
    "COVID Period AQI":
      "This chart focuses on the 2019–2021 COVID period, showing how lockdowns and reduced activity affected air quality across multiple stations."
  };

  const pollutants = ["PM2.5", "PM10", "Ozone", "CO", "NO2", "SO2"];
  const colors = d3.scaleOrdinal().domain(pollutants).range(d3.schemeTableau10);

  const M = { top: 50, right: 50, bottom: 65, left: 75 };
  const W = 900;
  const H = 480;

  function getKey(obj, keys) {
    const found = Object.keys(obj).find(k =>
      keys.some(kk => k.trim().toLowerCase().replace(/[^a-z0-9]/g, "") === kk.trim().toLowerCase().replace(/[^a-z0-9]/g, ""))
    );
    return found ? obj[found] : undefined;
  }

  const rows = data
    .map((d) => {
      const date = new Date(getKey(d, ["timestamp(utc)", "timestamp", "date"]));
      const main = getKey(d, ["mainpollutant", "dominantpollutant"]);
      const aqi = +getKey(d, ["usaqi", "aqi"]) || null;
      const row = { date, main, aqi };
      pollutants.forEach(p => {
        row[p] = +getKey(d, [p, p.toLowerCase(), p.replace(".", "")]) || null;
      });
      return row;
    })
    .filter(d => d.date && !isNaN(d.date.getTime()))
    .map(d => {
      pollutants.forEach(p => { if (d[p] > 300) d[p] = null; });
      if (d.aqi > 300) d.aqi = null;
      return d;
    });

  onMount(drawAll);

  function drawAll() {
    d3.select("#chart").selectAll("*").remove();

    const svg = d3.select("#chart")
      .append("svg")
      .attr("width", W)
      .attr("height", H);

    const g = svg.append("g").attr("transform", `translate(${M.left},${M.top})`);
    const innerW = W - M.left - M.right;
    const innerH = H - M.top - M.bottom;

    function addGrid(y) {
      g.append("g")
        .attr("class", "grid")
        .call(d3.axisLeft(y).tickSize(-innerW).tickFormat(""));
    }

    function addLabels(xText, yText) {
      svg.append("text")
        .attr("x", M.left + innerW / 2)
        .attr("y", H - 15)
        .attr("text-anchor", "middle")
        .attr("class", "axis-label")
        .text(xText);
      svg.append("text")
        .attr("transform", "rotate(-90)")
        .attr("x", -(M.top + innerH / 2))
        .attr("y", 20)
        .attr("text-anchor", "middle")
        .attr("class", "axis-label")
        .text(yText);
    }

    function addLegend() {
  const legend = svg.append("g")
    .attr("transform", `translate(${M.left},${H - 35})`)
    .attr("class", "legend");

  const itemWidth = 95;
  const rowHeight = 20;

  pollutants.forEach((p, i) => {
    const col = i % 3; // 3 items per row for better spacing
    const row = Math.floor(i / 3);
    const xPos = col * itemWidth;
    const yPos = row * rowHeight;

    legend.append("rect")
      .attr("x", xPos)
      .attr("y", yPos)
      .attr("width", 12)
      .attr("height", 12)
      .attr("fill", colors(p));

    legend.append("text")
      .attr("x", xPos + 18)
      .attr("y", yPos + 10)
      .attr("font-size", 12)
      .attr("fill", "#333")
      .text(p);
  });

  // Adjust SVG height if legend overflows
  const legendHeight = Math.ceil(pollutants.length / 3) * rowHeight + 15;
  svg.attr("height", H + legendHeight);
}


    // === PLOT 1: Stacked bar ===
    if (currentPlot === "Main Pollutants by Year") {
      const yearlyShare = d3.rollups(
        rows.filter(d => d.main),
        v => d3.rollup(v, vv => vv.length, d => d.main),
        d => d.date.getUTCFullYear()
      ).map(([year, map]) => ({
        year,
        total: d3.sum(map.values()),
        values: Object.fromEntries(map)
      })).sort((a, b) => a.year - b.year);

      const stacked = yearlyShare.flatMap(y => {
        let y0 = 0;
        return pollutants.map(p => {
          const val = (y.values[p] || 0) / y.total;
          const obj = { year: String(y.year), pollutant: p, y0, y1: y0 + val };
          y0 += val;
          return obj;
        });
      });

      const x = d3.scaleBand().domain(yearlyShare.map(d => String(d.year))).range([0, innerW]).padding(0.1);
      const y = d3.scaleLinear().domain([0, 1]).range([innerH, 0]);
      addGrid(y);

      g.selectAll("rect")
        .data(stacked)
        .enter()
        .append("rect")
        .attr("x", d => x(d.year))
        .attr("y", d => y(d.y1))
        .attr("height", d => y(d.y0) - y(d.y1))
        .attr("width", x.bandwidth())
        .attr("fill", d => colors(d.pollutant));

      g.append("g").attr("transform", `translate(0,${innerH})`).call(d3.axisBottom(x));
      g.append("g").call(d3.axisLeft(y).tickFormat(d3.format(".0%")));
      addLabels("Year", "Proportion of Main Pollutants");
      addLegend();
    }

    // === PLOT 2: AQI scatter ===
    if (currentPlot === "AQI Scatter by Date") {
      const longData = rows.flatMap(r =>
        pollutants.filter(p => r[p]).map(p => ({ date: r.date, pollutant: p, aqi: r[p] }))
      );

      const x = d3.scaleUtc().domain(d3.extent(longData, d => d.date)).range([0, innerW]);
      const y = d3.scaleLinear()
        .domain([0, Math.min(250, d3.max(longData, d => d.aqi))])
        .nice()
        .range([innerH, 0]);
      addGrid(y);

      g.selectAll("circle")
        .data(longData)
        .enter()
        .append("circle")
        .attr("cx", d => x(d.date))
        .attr("cy", d => y(d.aqi))
        .attr("r", 2)
        .attr("fill", d => colors(d.pollutant))
        .attr("opacity", 0.7);

      g.append("g").attr("transform", `translate(0,${innerH})`).call(d3.axisBottom(x));
      g.append("g").call(d3.axisLeft(y));
      addLabels("Date", "AQI (per pollutant)");
      addLegend();
    }

    // === PLOT 3: Line by pollutant ===
    if (currentPlot === "Mean AQI per Pollutant") {
      const grouped = Array.from(
        d3.rollup(
          rows.filter(d => d.aqi),
          v => d3.mean(v, d => d.aqi),
          d => d3.timeMonth(d.date),
          d => d.main
        ),
        ([month, map]) => ({ month, values: Object.fromEntries(map) })
      ).sort((a, b) => +a.month - +b.month);

      const x = d3.scaleUtc().domain(d3.extent(grouped, d => d.month)).range([0, innerW]);
      const y = d3.scaleLinear()
        .domain([0, Math.min(250, d3.max(grouped, d => d3.max(Object.values(d.values))))])
        .nice()
        .range([innerH, 0]);
      addGrid(y);

      const line = d3.line()
        .x(d => x(d.month))
        .y(d => y(d.value))
        .curve(d3.curveMonotoneX);

      pollutants.forEach(p => {
        const arr = grouped.map(m => ({ month: m.month, value: m.values[p] || 0 }));
        g.append("path")
          .datum(arr)
          .attr("fill", "none")
          .attr("stroke", colors(p))
          .attr("stroke-width", 2)
          .attr("d", line);
      });

      g.append("g").attr("transform", `translate(0,${innerH})`).call(d3.axisBottom(x));
      g.append("g").call(d3.axisLeft(y));
      addLabels("Date (monthly)", "Mean AQI per Pollutant");
      addLegend();
    }

    // === PLOT 4: Monthly mean AQI ===
    if (currentPlot === "Monthly Mean AQI") {
      const monthAvg = Array.from(
        d3.rollup(rows, v => d3.mean(v, d => d.aqi), d => d.date.getUTCMonth()),
        ([m, avg]) => ({ m, avg })
      ).sort((a, b) => a.m - b.m);

      const x = d3.scaleBand().domain(monthAvg.map(d => d.m)).range([0, innerW]).padding(0.1);
      const y = d3.scaleLinear().domain([0, d3.max(monthAvg, d => d.avg)]).nice().range([innerH, 0]);
      addGrid(y);

      g.selectAll("rect")
        .data(monthAvg)
        .enter()
        .append("rect")
        .attr("x", d => x(d.m))
        .attr("y", d => y(d.avg))
        .attr("height", d => innerH - y(d.avg))
        .attr("width", x.bandwidth())
        .attr("fill", "#4caf50");

      g.append("g").attr("transform", `translate(0,${innerH})`)
        .call(d3.axisBottom(x).tickFormat(m => d3.timeFormat("%b")(new Date(2020, +m, 1))));
      g.append("g").call(d3.axisLeft(y));
      addLabels("Month", "Mean AQI");
    }

    // === PLOT 5: COVID period ===
    if (currentPlot === "COVID Period AQI") {
      const covidData = rows.filter(d =>
        d.date >= new Date("2019-01-01") && d.date <= new Date("2021-12-31")
      );

      const x = d3.scaleUtc().domain(d3.extent(covidData, d => d.date)).range([0, innerW]);
      const y = d3.scaleLinear()
        .domain([0, Math.min(250, d3.max(covidData, d => d.aqi))])
        .nice()
        .range([innerH, 0]);
      addGrid(y);

      const line = d3.line().x(d => x(d.date)).y(d => y(d.aqi));
      g.append("path")
        .datum(covidData)
        .attr("fill", "none")
        .attr("stroke", "#1f77b4")
        .attr("stroke-width", 1.8)
        .attr("d", line);

      g.selectAll("circle")
        .data(covidData)
        .enter()
        .append("circle")
        .attr("cx", d => x(d.date))
        .attr("cy", d => y(d.aqi))
        .attr("r", 2)
        .attr("fill", "#ff7f0e")
        .attr("opacity", 0.6);

      g.append("g").attr("transform", `translate(0,${innerH})`).call(d3.axisBottom(x));
      g.append("g").call(d3.axisLeft(y));
      addLabels("Date (2019–2021)", "AQI");
    }
  }
</script>

<style>
  h2 {
    text-align: center;
    padding: 0.6rem 1.2rem;
    border-radius: 8px;
    font-size: 1.2rem;
    width: fit-content;
    margin: 0 auto;
  }

  .tabs-container {
    display: flex;
    justify-content: center;
    margin: 1.5rem 0;
  }

  .tabs {
    display: inline-flex;
    background: #ffffff;
    border: 1px solid #dcdcdc;
    border-radius: 10px;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
    overflow: hidden;
  }

  .tabs button {
    border: none;
    background: transparent;
    padding: 0.75rem 1.25rem;
    font-size: 0.95rem;
    font-weight: 600;
    color: #444;
    cursor: pointer;
    transition: all 0.25s ease;
    border-right: 1px solid #eee;
    min-width: 170px;
  }

  .tabs button:last-child {
    border-right: none;
  }

  .tabs button:hover {
    background: #f3f6fc;
    color: #0b6efd;
  }

  .tabs button.active {
    background: #0b6efd;
    color: white;
    box-shadow: inset 0 -2px 0 #004bbd;
  }

  #chart {
    display: flex;
    justify-content: center;
    align-items: center;
    background: #fafafa;
    border-radius: 8px;
    box-shadow: 0 0 5px rgba(0, 0, 0, 0.08);
    padding: 1rem 0;
    margin: 0 auto;
    width: 100%;
    max-width: 950px;
  }

  .axis-label {
    font-size: 13px;
    font-weight: 600;
    fill: #333;
  }

  .grid line {
    stroke: #e0e0e0;
    stroke-opacity: 0.7;
  }

  .grid path {
    display: none;
  }

  .legend text {
    font-size: 12px;
    font-family: 'Helvetica Neue', Arial, sans-serif;
  }

  .plot-description {
    max-width: 850px;
    margin: 0.8rem auto 2rem;
    color: #444;
    font-size: 0.95rem;
    font-family: 'Georgia', serif;
    line-height: 1.45;
    text-align: justify;
  }
</style>

<h2>AQI Dashboard – Key Pollutant Drivers Over Time</h2>

<div class="tabs-container">
  <div class="tabs">
    {#each plots as p}
      <button
        class:active={currentPlot === p}
        on:click={() => { currentPlot = p; drawAll(); }}
      >
        {p}
      </button>
    {/each}
  </div>
</div>

<div id="chart"></div>
<p class="plot-description">{plotDescriptions[currentPlot]}</p>