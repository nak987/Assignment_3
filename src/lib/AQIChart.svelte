<script lang="ts">
  import * as d3 from "d3";
  interface Item {
    city: string; country: string; mainPollutant: string;
    pm25: number; state: string; stationName: string;
    timestamp: Date | string; usAqi: number;
  }
  export let data: Item[] = [];
  // Assignment's AQI level code:
  const AQI_BANDS = [
    { name: "Good", min: 0, max: 50, color: "#9cd84e" },
    { name: "Moderate", min: 51, max: 100, color: "#facf39" },
    { name: "Unhealthy for Sensitive Groups", min: 101, max: 150, color: "#f99049" },
    { name: "Unhealthy", min: 151, max: 200, color: "#f65e5f" },
    { name: "Very Unhealthy", min: 201, max: 300, color: "#a070b6" },
    { name: "Hazardous", min: 301, color: "#a06a7b" }
  ];
  type Row = { date: Date; station: string; aqi: number };
  type Monthly = { month: Date; avg: number; p10: number; p90: number };
  let rows: Row[] = [];
  let stations: { name: string; count: number }[] = [];
  let monthly: Monthly[] = [];
  let showRaw = false;
  let selected: string = "all"; // "all" is safer than null for select binding
  const M = { top: 28, right: 28, bottom: 36, left: 52 };
  const W = 960, H = 480, iw = W - M.left - M.right, ih = H - M.top - M.bottom;

  // align to 15th of month
  $: rows = data.map(d => {
    const dt = new Date(d.timestamp);
    if (isNaN(dt.getTime())) return null;
    return {
      date: new Date(Date.UTC(dt.getUTCFullYear(), dt.getUTCMonth(), 15)),
      station: d.stationName,
      aqi: +d.usAqi
    };
}).filter((d): d is Row => !!d && isFinite(d.aqi) && Boolean(d.station));
  // sort station names by count
  $: stations = Array.from(d3.rollup(rows, v => v.length, r => r.station), ([name, count]) => ({ name, count }))
    .sort((a, b) => d3.descending(a.count, b.count));

  // monthly average + percentiles
  function aggregateMonthly(src: Row[], st?: string) {
    const f = st && st !== "all" ? src.filter(r => r.station === st) : src;
    const grouped = d3.rollup(f, vals => {
      const a = vals.map(v => v.aqi).sort(d3.ascending);
      return {
        avg: d3.mean(a)!,
        p10: d3.quantileSorted(a, 0.1)!,
        p90: d3.quantileSorted(a, 0.9)!
      };
    }, v => `${v.date.getUTCFullYear()}-${v.date.getUTCMonth()}`);

    return Array.from(grouped, ([k, s]) => {
      const [y, m] = k.split("-").map(Number);
      return { month: new Date(Date.UTC(y, m, 15)), ...s };
    }).sort((a, b) => +a.month - +b.month);
  }

  $: monthly = rows.length ? aggregateMonthly(rows, selected) : [];

  const x = d3.scaleUtc().range([0, iw]);
  const y = d3.scaleLinear().range([ih, 0]);

  $: x.domain(monthly.length ? d3.extent(monthly, d => d.month) as [Date, Date] : [new Date(), new Date()]);
  // y axis to prevent dense raw dots from bunching up
  $: y.domain([0, Math.max(180, d3.max(monthly, d => d.p90) ?? 100)]).nice();
  const area = d3.area<Monthly>().x(d => x(d.month)).y0(d => y(d.p10)).y1(d => y(d.p90));
  const line = d3.line<Monthly>().x(d => x(d.month)).y(d => y(d.avg));
  $: raw = showRaw ? (selected === "all" ? rows : rows.filter(r => r.station === selected)) : [];
</script>
<div class="chart">
  <h2>AQI Chart</h2>
  <div class="controls">
    <label>Station:
      <select bind:value={selected}>
        <option value="all">All stations</option>
        {#each stations as s}
          <option value={s.name}>{s.name} ({s.count})</option>
        {/each}
      </select>
    </label>
    <label><input type="checkbox" bind:checked={showRaw}> Show Raw Data</label>
  </div>
  <div class="legend">
    {#each AQI_BANDS as b}
      <div class="band" style="background:{b.color}">{b.name} {b.min}–{b.max ?? '300+'}</div>
    {/each}
  </div>
  <p class="meta">Records: {rows.length}</p>
  <svg width={W} height={H} viewBox={`0 0 ${W} ${H}`}>
    <g transform={`translate(${M.left},${M.top})`}>
      {#each AQI_BANDS as b}
        <rect
          x="0"
          y={y(b.max || y.domain()[1])}
          width={iw}
          height={y(b.min) - y(b.max || y.domain()[1])}
          fill={b.color}
          opacity="0.25" />
      {/each}
      <path d={area(monthly)} fill="currentColor" opacity="0.3" />
      <path d={line(monthly)} fill="none" stroke="black" stroke-width="2" />
      {#if raw.length}
        {#each raw as p}
          <circle cx={x(p.date)} cy={y(p.aqi)} r="1.5" fill="black" opacity="0.6" />
        {/each}
      {/if}
      <g transform={`translate(0,${ih})`}>
        {#each x.ticks(8) as t}
          <g transform={`translate(${x(t)},0)`}>
            <line y2="6" stroke="black" />
            <text y="18" text-anchor="middle" font-size="11">{d3.utcFormat("%Y")(t)}</text>
          </g>
        {/each}
        <path d={`M0,0.5H${iw}`} stroke="black" />
      </g>
      <g>
        {#each y.ticks(6) as t}
          <g transform={`translate(0,${y(t)})`}>
            <line x2="-6" stroke="black" />
            <text x="-9" dy="0.32em" text-anchor="end" font-size="11">{t}</text>
            <line x1="0" x2={iw} stroke="black" opacity="0.1" />
          </g>
        {/each}
        <text transform="rotate(-90)" x={-ih/2} y={-40} text-anchor="middle" font-size="12">AQI</text>
      </g>
    </g>
  </svg>
</div>
<style>
  .chart { max-width: 1040px; margin: 1rem auto; }
  .controls { display: flex; gap: 1rem; align-items: center; margin-bottom: .5rem; }
  .legend { display: flex; flex-wrap: wrap; gap: .4rem; margin: .4rem 0; }
  .band { padding: .3rem .6rem; border-radius: .4rem; font-size: .85rem; color: #222; }
  .meta { margin: .3rem 0 .7rem; font-size: .9rem; color: #333; }
  svg { max-width: 100%; font-family: system-ui, sans-serif; }
</style>