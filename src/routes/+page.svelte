<script lang="ts">
  import * as d3 from 'd3';
  import AQIChart from '$lib/AQIChart.svelte';
  import MultiStation from '$lib/MultiStation.svelte';

  // --- dataset URLs ---
  const datasets = {
    avalon: 'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BAvalon%5D_daily-avg.csv',
    glassport_high_street:
      'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BGlassport%20High%20Street%5D_daily-avg.csv',
    lawrenceville:
      'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLawrenceville%5D_daily-avg.csv',
    liberty_sahs:
      'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLiberty%20(SAHS)%5D_daily-avg.csv',
    manchester:
      'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BManchester%5D_daily-avg.csv',
    north_braddock:
      'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BNorth%20Braddock%5D_daily-avg.csv',
    parkway_east_near_road:
      'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BParkway%20East%20(Near%20Road)%5D_daily-avg.csv',
    usa_pennsylvania_pittsburgh:
      'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BUSA-Pennsylvania-Pittsburgh%5D_daily-avg.csv'
  };

  // --- load all station CSVs at once ---
  async function loadAllData() {
    const parseRow = (d: any) => ({
      city: d.City,
      country: d.Country,
      mainPollutant: d['Main pollutant'],
      pm25: +d['PM2.5'],
      pm10: +d['PM10'],
      ozone: +d['Ozone'],
      co: +d['CO'],
      no2: +d['NO2'],
      so2: +d['SO2'],
      state: d.State,
      stationName: d['Station name'],
      timestamp: new Date(d['Timestamp(UTC)']),
      usAqi: +d['US AQI']
    });

    const allDataArrays = await Promise.all(
      Object.values(datasets).map(url => d3.csv(url, parseRow))
    );
    return allDataArrays.flat();
  }

  const dataPromise = loadAllData();

  // --- which part is active ---
  let activePart: 1 | 2 = 1;
</script>

{#await dataPromise}
  <p>Loading air-quality datasets…</p>
{:then data}
  <div class="controls">
    <button
      on:click={() => (activePart = 1)}
      class:active={activePart === 1}
    >
      Part 1 – AQI per Station
    </button>
    <button
      on:click={() => (activePart = 2)}
      class:active={activePart === 2}
    >
      Part 2 – Multi-Station Analysis
    </button>
  </div>

  {#if activePart === 1}
    <AQIChart {data} />
  {:else if activePart === 2}
    <MultiStation {data} />
  {/if}
{:catch error}
  <p style="color:red">Error loading data: {error.message}</p>
{/await}

<style>
  * { font-family: system-ui, sans-serif; }
  .controls {
    display: flex;
    gap: 1rem;
    margin: 1rem 0;
  }
  button {
    background: #eee;
    border: 1px solid #ccc;
    border-radius: 6px;
    padding: 0.5rem 1rem;
    cursor: pointer;
    font-size: 0.95rem;
    transition: all 0.2s ease;
  }
  button:hover {
    background: #ddd;
  }
  button.active {
    background: #0074d9;
    color: white;
    border-color: #0074d9;
  }
</style>