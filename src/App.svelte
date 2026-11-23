<script>
  const distances = [
    { value: 5000, label: '5K (5000m)' },
    { value: 3200, label: '3200m' },
    { value: 1600, label: '1600m / Mile' },
    { value: 800, label: '800m' },
    { value: 400, label: '400m' },
    { value: 200, label: '200m' },
    { value: 100, label: '100m' },
    { value: 55, label: '55m' },
  ];
  const percentages = [0, 50, 70, 80, 85, 90];

  let raceDistance = '1600';
  let raceTime = '5:00';
  let percentage = '0';
  let activeMode = 'distance';
  let rows = buildRows(raceDistance, raceTime, percentage, activeMode);

  function parseTimeToSeconds(time) {
    const parts = time.trim().split(':');
    if (parts.length === 2) {
      return parseInt(parts[0], 10) * 60 + parseFloat(parts[1]);
    }
    if (parts.length === 1) return parseFloat(parts[0]);
    return null;
  }

  function formatTime(totalSeconds) {
    const tenths = Math.round(totalSeconds * 10);
    const minutes = Math.floor(tenths / 600);
    const seconds = (tenths % 600) / 10;
    return `${minutes}:${seconds.toFixed(1).padStart(4, '0')}`;
  }

  function buildRows(distance, time, pacePercentage, mode) {
    const raceDistanceMeters = Number.parseInt(distance, 10);
    const totalSeconds = parseTimeToSeconds(time);
    if (!Number.isFinite(totalSeconds) || totalSeconds <= 0) return null;

    const isSprint = mode === 'sprint';
    const intervalDistances = isSprint
      ? [55, 100, 150, 200, 250, 300, 350]
      : [200, 400, 500, 600, 800, 1000, 1200, 1600];
    const effortModifiers = isSprint
      ? [1, 1.06, 1.15]
      : [1, 1.06, 1.15, 1.25, 1.4];
    const basePace = totalSeconds / raceDistanceMeters;
    const selectedModifier = 1 + Number(pacePercentage) / 100;

    return intervalDistances.map((distanceMeters) => ({
      distance: `${distanceMeters}m`,
      times: effortModifiers.map((modifier) =>
        formatTime(basePace * selectedModifier * distanceMeters * modifier),
      ),
      selectedPace: isSprint
        ? null
        : formatTime(basePace * selectedModifier * 1600),
    }));
  }

  function calculateSplits() {
    const nextRows = buildRows(raceDistance, raceTime, percentage, activeMode);
    if (!nextRows) {
      window.alert('Please enter a valid time (e.g., 5:00).');
      return false;
    }
    rows = nextRows;
    return true;
  }

  function changePercentage() {
    calculateSplits();
  }

  function selectMode(mode) {
    const previousMode = activeMode;
    activeMode = mode;
    if (!calculateSplits()) activeMode = previousMode;
  }

  function handleTabKeydown(event) {
    if (!['ArrowLeft', 'ArrowRight', 'Home', 'End'].includes(event.key)) return;
    event.preventDefault();
    const mode =
      event.key === 'Home'
        ? 'distance'
        : event.key === 'End'
          ? 'sprint'
          : activeMode === 'distance'
            ? 'sprint'
            : 'distance';
    if (selectMode(mode)) {
      document.getElementById(mode === 'sprint' ? 'sprintTab' : 'distanceTab')?.focus();
    }
  }
</script>

<svelte:head>
  <meta name="description" content="Calculate running workout splits from a race time." />
</svelte:head>

<main class="container">
  <h1>Training Pace Calculator</h1>
  <p class="instructions">
    Select your race distance and enter your race time to calculate your workout splits. The
    selected percentage adds time to race pace for all intervals: 50% means a split takes 50%
    longer. The VO2 Max, Threshold, Tempo, and Easy percentages are applied to that adjusted pace.
  </p>

  <div class="input-group">
    <label for="distanceSelect">Race Distance</label>
    <select id="distanceSelect" bind:value={raceDistance}>
      {#each distances as distance (distance.value)}
        <option value={String(distance.value)}>{distance.label}</option>
      {/each}
    </select>
  </div>

  <div class="input-group">
    <label for="timeInput">Race Time (MM:SS)</label>
    <input
      type="text"
      id="timeInput"
      bind:value={raceTime}
      placeholder="e.g. 5:00"
      aria-describedby="timeHint"
    />
    <span id="timeHint" class="visually-hidden">Enter a time in minutes and seconds.</span>
  </div>

  <div class="input-group">
    <label for="percentageSelect">Percentage of Race Pace</label>
    <select id="percentageSelect" bind:value={percentage} onchange={changePercentage}>
      {#each percentages as value (value)}
        <option value={String(value)}>
          {value === 0 ? '100% (race pace)' : `${value}%`}
        </option>
      {/each}
    </select>
  </div>

  <button class="calculate-button" onclick={calculateSplits}>Calculate Splits</button>

  <div class="tabs" role="tablist" aria-label="Interval type">
    <button
      type="button"
      id="distanceTab"
      class="tab"
      role="tab"
      aria-selected={activeMode === 'distance'}
      aria-controls="intervalPanel"
      tabindex={activeMode === 'distance' ? 0 : -1}
      onclick={() => selectMode('distance')}
      onkeydown={handleTabKeydown}
    >
      Distance
    </button>
    <button
      type="button"
      id="sprintTab"
      class="tab"
      role="tab"
      aria-selected={activeMode === 'sprint'}
      aria-controls="intervalPanel"
      tabindex={activeMode === 'sprint' ? 0 : -1}
      onclick={() => selectMode('sprint')}
      onkeydown={handleTabKeydown}
    >
      Sprinters
    </button>
  </div>

  <div
    class="table-container"
    id="intervalPanel"
    role="tabpanel"
    aria-labelledby={activeMode === 'sprint' ? 'sprintTab' : 'distanceTab'}
    tabindex="0"
  >
    <table>
      <thead>
        <tr>
          <th scope="col">Rep<br />Dist</th>
          <th scope="col">Adjusted Race Pace<br />(100%)</th>
          <th scope="col">VO2 Max<br />(106%)</th>
          <th scope="col">Threshold<br />(115%)</th>
          {#if activeMode === 'distance'}
            <th scope="col">Tempo<br />(125%)</th>
            <th scope="col">Easy<br />(140%)</th>
            <th scope="col">Selected Pace<br />per 1600m</th>
          {/if}
        </tr>
      </thead>
      <tbody>
        {#each rows as row (row.distance)}
          <tr>
            <td>{row.distance}</td>
            {#each row.times as time, index (`${row.distance}-${index}`)}
              <td>{time}</td>
            {/each}
            {#if row.selectedPace !== null}<td>{row.selectedPace}</td>{/if}
          </tr>
        {/each}
      </tbody>
    </table>
  </div>
</main>

<style>
  :global(*) {
    box-sizing: border-box;
  }

  :global(body) {
    margin: 0;
    padding: 40px 20px;
    background: #f3f4f6;
    color: #374151;
    font-family:
      -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  }

  .container {
    max-width: 800px;
    margin: 0 auto;
    padding: 30px;
    border-radius: 8px;
    background: #fff;
    box-shadow: 0 4px 6px rgb(0 0 0 / 5%);
  }

  h1 {
    margin: 0 0 20px;
    padding-bottom: 15px;
    border-bottom: 2px solid #3b82f6;
    color: #1f2937;
    font-size: 24px;
  }

  .instructions {
    margin-bottom: 25px;
    color: #4b5563;
  }

  .input-group {
    margin-bottom: 20px;
  }

  label {
    display: block;
    margin-bottom: 8px;
    font-size: 14px;
    font-weight: 700;
  }

  select,
  input[type='text'] {
    width: 100%;
    padding: 10px 12px;
    border: 1px solid #d1d5db;
    border-radius: 4px;
    background: #fff;
    color: inherit;
    font-size: 16px;
  }

  input[type='text']:focus,
  select:focus,
  button:focus-visible,
  .table-container:focus-visible {
    outline: 2px solid #2563eb;
    outline-offset: 2px;
  }

  button {
    border: 0;
    border-radius: 6px;
    cursor: pointer;
    font-size: 16px;
    font-weight: 700;
    transition: background-color 0.2s;
  }

  .calculate-button {
    width: 100%;
    margin-bottom: 30px;
    padding: 14px;
    background: #3fa9f5;
    color: white;
  }

  .calculate-button:hover,
  .tab[aria-selected='true'] {
    background: #2b8ccd;
    color: white;
  }

  .tabs {
    display: flex;
    gap: 8px;
    margin-bottom: 20px;
  }

  .tab {
    width: auto;
    padding: 12px 18px;
    background: #e5e7eb;
    color: #374151;
  }

  .table-container {
    overflow-x: auto;
  }

  table {
    width: 100%;
    border-collapse: collapse;
    text-align: center;
  }

  th,
  td {
    padding: 14px;
    border: 1px solid #f3f4f6;
  }

  th {
    background: #fafafa;
    color: #6b7280;
    font-size: 14px;
    font-weight: 600;
  }

  td {
    color: #4b5563;
  }

  td:first-child {
    color: #1f2937;
    font-weight: 700;
  }

  .visually-hidden {
    position: absolute;
    width: 1px;
    height: 1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    clip-path: inset(50%);
  }

  @media (max-width: 560px) {
    :global(body) {
      padding: 16px 10px;
    }

    .container {
      padding: 20px 14px;
    }

    th,
    td {
      padding: 10px 8px;
    }
  }
</style>