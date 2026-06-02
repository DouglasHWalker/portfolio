<script>
  import { onMount } from 'svelte';

  // --- PROJECT 1: PRICED-OUT GENERATION (SCROLLYTELLING) ---
  let scrollY = 0;
  /** @type {HTMLElement | null} */
  let scrollContainer = null;
  let containerTop = 0;
  let containerHeight = 0;
  let scrollPercent = 0;

  // Mock data representing 20 years of ABS index data (2006 = 100)
  const housingTrendData = [
    { year: 2006, wage: 100, rent: 100 },
    { year: 2011, wage: 115, rent: 132 },
    { year: 2016, wage: 128, rent: 165 },
    { year: 2021, wage: 142, rent: 210 },
    { year: 2026, wage: 158, rent: 285 }
  ];

  $: if (scrollContainer) {
    const rect = scrollContainer.getBoundingClientRect();
    containerTop = rect.top + window.scrollY;
    containerHeight = rect.height;
    
    // Calculate how far through the scrolly section the user is (0 to 1)
    const relativeScroll = window.scrollY - containerTop;
    const maxScroll = containerHeight - window.innerHeight;
    scrollPercent = Math.min(Math.max(relativeScroll / maxScroll, 0), 1);
  }

  // Animate line chart draws based on scroll depth
  $: visibleDataPoints = Math.max(1, Math.ceil(scrollPercent * housingTrendData.length));
  $: activeData = housingTrendData.slice(0, visibleDataPoints);

  // --- PROJECT 2: MODERN LOVE (CENSUS INTERACTIVE) ---
  let selectedState = 'ALL';
  /** @type {Object.<string, {single: number, married: number, deFacto: number, total: string}>} */
  const censusData = {
    ALL: { single: 34, married: 48, deFacto: 18, total: "10.4M households" },
    NSW: { single: 33, married: 50, deFacto: 17, total: "3.1M households" },
    VIC: { single: 35, married: 48, deFacto: 17, total: "2.6M households" },
    QLD: { single: 32, married: 46, deFacto: 22, total: "2.1M households" },
    WA:  { single: 31, married: 49, deFacto: 20, total: "1.0M households" }
  };

   /**
   * Keyboard navigation handler for accessibility
   * @param {any} event
   * @param {string} stateCode
   */
  function handleStateKey(event, stateCode) {
    if (event.key === 'Enter' || event.key === ' ') {
      event.preventDefault();
      selectedState = stateCode;
    }
  }

  onMount(() => {
    // Recalculate layout metrics safely on mount/resize
    const handleResize = () => {
      if (scrollContainer) {
        const rect = scrollContainer.getBoundingClientRect();
        containerTop = rect.top + window.scrollY;
        containerHeight = rect.height;
      }
    };
    window.addEventListener('scroll', () => { scrollY = window.scrollY; });
    window.addEventListener('resize', handleResize);
    handleResize();

    return () => {
      window.removeEventListener('scroll', () => {});
      window.removeEventListener('resize', handleResize);
    };
  });
</script>

<main class="abc-editorial-container">
  
  <!-- ================= PROJECT 1 SECTION ================= -->
  <section class="story-section">
    <header class="section-header">
      <span class="badge">Data Investigation</span>
      <h2>The Priced-Out Generation: Mapping Youth Housing Stress</h2>
      <p class="intro-text">Scroll down to observe how property and rental indexes have aggressively separated from wage growth since 2006.</p>
    </header>

    <!-- Scrollytelling Container -->
    <div bind:this={scrollContainer} class="scrolly-container">
      
      <!-- Sticky Graphic Visualization -->
      <div class="sticky-visual" aria-hidden="true">
        <div class="chart-wrapper">
          <svg viewBox="0 0 500 300" class="svg-chart">
            <!-- Grid Lines -->
            <line x1="50" y1="50" x2="450" y2="50" stroke="#e0e0e0" stroke-dasharray="4"/>
            <line x1="50" y1="150" x2="450" y2="150" stroke="#e0e0e0" stroke-dasharray="4"/>
            <line x1="50" y1="250" x2="450" y2="250" stroke="#ccc" />

            <!-- X Axis Labels -->
            {#each housingTrendData as d, i}
              <text x={50 + (i * 100)} y="270" class="axis-label" text-anchor="middle">{d.year}</text>
            {/each}

            <!-- Y Axis Labels -->
            <text x="40" y="55" class="axis-label" text-anchor="end">300%</text>
            <text x="40" y="155" class="axis-label" text-anchor="end">200%</text>
            <text x="40" y="255" class="axis-label" text-anchor="end">100%</text>

            <!-- Line: Youth Wages (Map 100-300 index to Y 250-50) -->
            {#if activeData.length > 0}
              <path
                d={`M ${activeData.map((d, i) => `${50 + (i * 100)},${250 - ((d.wage - 100) * 1)}`).join(' L ')}`}
                fill="none"
                stroke="#005A9C"
                stroke-width="4"
                stroke-linecap="round"
              />
              <!-- Line: Median Capital Rents -->
              <path
                d={`M ${activeData.map((d, i) => `${50 + (i * 100)},${250 - ((d.rent - 100) * 1)}`).join(' L ')}`}
                fill="none"
                stroke="#D50000"
                stroke-width="4"
                stroke-linecap="round"
              />
            {/if}
          </svg>

          <!-- Dynamic Overlay Legend -->
          <div class="legend-panel">
            <div class="legend-item"><span class="dot wage"></span> Youth Wages Index</div>
            <div class="legend-item"><span class="dot rent"></span> Capital City Rent Index</div>
          </div>
        </div>
      </div>

      <!-- Scroll Text Steps -->
      <div class="scroll-narrative">
        <div class="step-card">
          <h3>2006: The Baseline</h3>
          <p>At the turn of the census cycle, average youth wages and capital city rents tracked evenly relative to baseline cost-of-living indicators.</p>
        </div>
        <div class="step-card">
          <h3>2016: The Decoupling</h3>
          <p>A decade later, a widening chasm formed. Structural supply bottlenecks and tax incentives pulled housing prices cleanly out of reach for median earners.</p>
        </div>
        <div class="step-card">
          <h3>Present Day: Structural Stress</h3>
          <p>The gap cements into a crisis profile. Rent burdens exceed 30% of gross young-adult income levels in almost every major metropolitan sector.</p>
        </div>
      </div>
    </div>
  </section>

  <!-- ================= PROJECT 2 SECTION ================= -->
  <section class="story-section interactive-dashboard">
    <header class="section-header">
      <span class="badge">Census Data Interactive</span>
      <h2>Mapping Modern Love: The Changing Australian Household</h2>
      <p class="intro-text">Select an option or click a region below to explore how relationship structures have shifted dynamically in your state.</p>
    </header>

    <div class="dashboard-grid">
      <!-- Accessible Controls / Map Simulation -->
      <nav class="map-controls" aria-label="Filter data by State or Territory">
        {#each Object.keys(censusData) as stateCode}
          <button 
            class="state-btn" 
            class:active={selectedState === stateCode}
            aria-pressed={selectedState === stateCode}
            on:click={() => selectedState = stateCode}
            on:keydown={(e) => handleStateKey(e, stateCode)}
          >
            {stateCode === 'ALL' ? 'Australia-wide' : stateCode}
          </button>
        {/each}
      </nav>

      <!-- Data Visual Output Display Panel -->
      <article class="data-display" aria-live="polite">
        <header class="display-header">
          <h3>Region Summary: <span class="highlight">{selectedState === 'ALL' ? 'National Data' : selectedState}</span></h3>
          <p class="sample-size">Sample size: {censusData[selectedState].total}</p>
        </header>

        <!-- Dynamic Bar Charts -->
        <div class="bar-chart-container">
          <div class="chart-row">
            <div class="row-label">Married</div>
            <div class="bar-wrapper">
              <div class="bar bar-married" style="width: {censusData[selectedState].married}%"></div>
              <span class="bar-value">{censusData[selectedState].married}%</span>
            </div>
          </div>

          <div class="chart-row">
            <div class="row-label">Single / Lone</div>
            <div class="bar-wrapper">
              <div class="bar bar-single" style="width: {censusData[selectedState].single}%"></div>
              <span class="bar-value">{censusData[selectedState].single}%</span>
            </div>
          </div>

          <div class="chart-row">
            <div class="row-label">De Facto</div>
            <div class="bar-wrapper">
              <div class="bar bar-defacto" style="width: {censusData[selectedState].deFacto}%"></div>
              <span class="bar-value">{censusData[selectedState].deFacto}%</span>
            </div>
          </div>
        </div>
      </article>
    </div>
  </section>
</main>

<style>
  /* Base typography and resetting rules aligned with editorial tech specs */
  .abc-editorial-container {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    color: #1a1a1a;
    max-width: 1100px;
    margin: 0 auto;
    padding: 20px;
    box-sizing: border-box;
  }

  .story-section {
    margin-bottom: 120px;
  }

  .section-header {
    margin-bottom: 40px;
    border-bottom: 1px solid #e0e0e0;
    padding-bottom: 20px;
  }

  .badge {
    background-color: #ff6f00;
    color: white;
    font-size: 0.75rem;
    font-weight: 700;
    text-transform: uppercase;
    padding: 4px 8px;
    letter-spacing: 1px;
    border-radius: 2px;
  }

  h2 {
    font-size: 2.2rem;
    margin: 10px 0;
    color: #06254B; /* ABC Corporate Blue hue matching */
  }

  .intro-text {
    font-size: 1.15rem;
    color: #555;
    line-height: 1.6;
    max-width: 750px;
  }

  /* --- PRICED-OUT GENERATION (SCROLLYTELLING CSS) --- */
  .scrolly-container {
    position: relative;
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
  }

  .sticky-visual {
    position: sticky;
    top: 10%;
    flex: 1 1 500px;
    height: 60vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: #f9f9f9;
    border-radius: 6px;
    padding: 20px;
    box-shadow: inset 0 0 10px rgba(0,0,0,0.03);
}
.chart-wrapper {width: 100%;max-width: 500px;}
.svg-chart {width: 100%;height: auto;overflow: visible;}
.axis-label {font-size: 0.75rem;fill: #666;}
.legend-panel {display: flex;justify-content: center;gap: 20px;margin-top: 15px;font-size: 0.85rem;font-weight: 600;}
.legend-item {display: flex;align-items: center;gap: 6px;}
.dot {width: 12px;height: 12px;border-radius: 50%;display: inline-block;}
.dot.wage { background-color: #005A9C; }
.dot.rent { background-color: #D50000; }
.scroll-narrative {flex: 1 1 350px;padding-bottom: 20vh;}
.step-card {background: white;padding: 30px;margin: 40vh 0;border-left: 4px solid #06254B;box-shadow: 0 4px 12px rgba(0,0,0,0.08);border-radius: 0 4px 4px 0;}
.step-card h3 {margin-top: 0;color: #06254B;}/* --- MODERN LOVE (INTERACTIVE CSS) --- */
.dashboard-grid {display: grid;grid-template-columns: 1fr 2fr;gap: 30px;background: #ffffff;border: 1px solid #e2e8f0;border-radius: 8px;padding: 24px;box-shadow: 0 2px 4px rgba(0,0,0,0.02);}@media (max-width: 768px) {
.dashboard-grid {grid-template-columns: 1fr;}
.sticky-visual {position: relative;height: 40vh;top: 0;}}
.map-controls {display: flex;flex-direction: column;gap: 10px;}
.state-btn {background: #f1f5f9;border: 1px solid #cbd5e1;color: #334155;padding: 14px;font-weight: 600;text-align: left;border-radius: 6px;cursor: pointer;transition: all 0.2s ease;}
.state-btn:hover {background: #e2e8f0;}
.state-btn.active {background: #06254B;color: #ffffff;border-color: #06254B;}
.data-display {background: #f8fafc;padding: 24px;border-radius: 6px;}
.display-header h3 {margin: 0;font-size: 1.4rem;}
.sample-size {font-size: 0.85rem;color: #64748b;margin: 4px 0 20px 0;}
.bar-chart-container {display: flex;flex-direction: column;gap: 20px;}
.chart-row {display: flex;flex-direction: column;gap: 6px;}
.row-label {font-weight: 600;font-size: 0.95rem;}
.bar-wrapper {display: flex;align-items: center;gap: 12px;background: #e2e8f0;border-radius: 4px;overflow: hidden;position: relative;height: 32px;}
.bar {height: 100%;transition: width 0.4s cubic-bezier(0.16, 1, 0.3, 1);}
.bar-married { background-color: #0284c7; }
.bar-single  { background-color: #0d9488; }
.bar-defacto { background-color: #4f46e5; }
.bar-value {position: absolute;right: 12px;font-weight: 700;color: #1e293b;font-size: 0.9rem;}
</style>