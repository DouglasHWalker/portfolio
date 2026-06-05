<script>
    import { onMount } from "svelte";
    import maplibregl from "maplibre-gl";
    import "maplibre-gl/dist/maplibre-gl.css";

    import * as d3 from "d3";
    import { slide } from "svelte/transition";

    /**
     * @typedef {Object} BusinessRow
     * @property {string} year
     * @property {string} industry
     * @property {number} suburb_id
     * @property {string} suburb
     * @property {Record<string, number>} sizes
     */

    /**
     * @typedef {Object} LookupMetrics
     * @property {string} suburbName
     * @property {number} count
     * @property {number} growth
     */

    /** @type {HTMLDivElement | undefined} */
    let mapContainer = undefined;
    /** @type {maplibregl.Map | null} */
    let map = null;

    // Core Reactive Application States
    /** @type {BusinessRow[]} */
    let businessData = $state([]);
    /** @type {Map<number, number>} */
    let precomputedGrowthMap = $state(new Map());

    /** @type {string[]} */
    let continuousYears = $state([]);
    /** @type {string[]} */
    let continuousIndustries = $state([]);

    // Reactive UI Selection Bindings
    let selectedYear = $state("2025");
    let selectedIndustry = $state("Accommodation and Food Services");
    let selectedBucket = $state("All Sizes");
    let mapMetricMode = $state("count"); // "count" or "growth"

    const bucketOptions = [
        "All Sizes",
        "Non employing",
        "1-4 Employees",
        "5-19 Employees",
        "20-199 Employees",
        "200+ Employees",
    ];

    // Tooltip and Hover Interaction State Hooks
    let hoveredSuburbName = $state("");
    let hoveredSuburbCount = $state(0);
    let hoveredSuburbGrowth = $state(0);
    let tooltipX = $state(0);
    let tooltipY = $state(0);
    let showTooltip = $state(false);
    /** @type {string | number | null} */
    let hoveredFeatureId = null;

    // Fixed Vector Asset Target Constants
    const TILESET_ID = "atlasgigerenzer.6qx896";
    const SOURCE_LAYER = "SA2_2021_AUST_SHP_GDA2020.zip-mg2r8x";

    // 1. Reactive Data Filter Slice (Filters main industry data frame instantly)
    let filteredRecords = $derived(
        businessData.filter(
            (d) => d.year === selectedYear && d.industry === selectedIndustry,
        ),
    );

    // 2. High-Performance Dictionary Lookups (O(1) lookups for the GPU engine)
    /** @type {Map<number, LookupMetrics>} */
    let activeDataLookup = $derived.by(() => {
        const lookup = new Map();

        filteredRecords.forEach((d) => {
            // Pair the filtered volume with the suburb's total baseline growth
            const baselineGrowth = precomputedGrowthMap.get(d.suburb_id) || 0;

            lookup.set(d.suburb_id, {
                suburbName: d.suburb,
                count: d.sizes[selectedBucket] || 0,
                growth: baselineGrowth,
            });
        });
        return lookup;
    });


    // 3. Automated Sidebar Editorial Insight Engine (With dynamic mode routing)
    let statsSummary = $derived.by(() => {
        if (mapMetricMode === "growth") {
            // Calculate growth specific insights using the precomputed map
            let totalGrowthValue = 0;
            let countWithGrowthData = 0;
            /** @type {[number, number]} */
            let topGrower = [0, -Infinity]; // [suburb_id, percentage]

            precomputedGrowthMap.forEach((growthVal, id) => {
                totalGrowthValue += growthVal;
                countWithGrowthData++;
                
                if (growthVal > topGrower[1]) {
                    topGrower = [id, growthVal];
                }
            });

            // Find the suburb name matching our top growth ID from the main array
            const topGrowerRecord = businessData.find(d => d.suburb_id === topGrower[0]);
            const topGrowerName = topGrowerRecord ? topGrowerRecord.suburb : `ID: ${topGrower[0]}`;
            const averageGrowth = countWithGrowthData > 0 ? (totalGrowthValue / countWithGrowthData) : 0;

            return {
                label1: "Average Suburb Growth",
                value1: `${averageGrowth > 0 ? '+' : ''}${Math.round(averageGrowth * 10) / 10}%`,
                subtext1: "Mean trend across all regions",
                label2: "Fastest Growing Market",
                value2: topGrowerName,
                subtext2: `Surged by <strong>+${topGrower[1]}%</strong> overall`
            };
        } else {
            // Default fallback: Your original volumetric industry absolute count layout
            const total = filteredRecords.reduce((acc, curr) => acc + (curr.sizes[selectedBucket] || 0), 0);
            const top = filteredRecords.reduce(
                (max, curr) => (curr.sizes[selectedBucket] || 0) > (max.sizes[selectedBucket] || 0) ? curr : max,
                /** @type {BusinessRow} */ ({ suburb: "None", sizes: {}, year: "", industry: "", suburb_id: 0 }),
            );

            return {
                label1: "Total Sector Footprint",
                value1: total.toLocaleString(),
                subtext1: "Active entities Nationwide",
                label2: "Top Commercial Hotspot",
                value2: top.suburb,
                subtext2: `Hosts <strong>${(top.sizes?.[selectedBucket] || 0).toLocaleString()}</strong> active units`
            };
        }
    });


    // 4. Reactive GPU Paint Sync Trigger
    $effect(() => {
        // Explicitly read primitive values inline to register them with the tracker engine
        mapMetricMode;
        selectedBucket;
        selectedYear;
        selectedIndustry;
        filteredRecords.length; // Accessing a sub-primitive forces dependency tracking

        // Ensure both the dataset and the MapLibre engine style matrices are fully mounted
        // if (map && map.isStyleLoaded() && businessData.length > 0) {
            updateMapData();
        // }
    });

    onMount(async () => {
        const basePath = import.meta.env.BASE_URL;

        // 1. Parallel loading of both the main dataset and the secondary precomputed growth dataset
        const [rawBusinessData, rawGrowthData] = await Promise.all([
            d3.csv(`${basePath}data/business_counts.csv`),
            d3.csv(`${basePath}data/suburb_growth.csv`), // Assumed filename for your new growth table
        ]);

        // 2. Parse and index the new secondary growth map instantly by its suburb identifier
        const growthIndex = new Map();

        rawGrowthData.forEach(
            (/** @type {{ suburb_id: any; Growth: any; }} */ row) => {
                const id = Number(String(row.suburb_id || "").trim());
                // Translate fractional growth rates into clean layout percentages (e.g. 0.0125 -> 1.3%)
                const rawGrowthDecimal = Number(
                    String(row.Growth || "0").trim(),
                );
                const roundedPercentage =
                    Math.round(rawGrowthDecimal * 100 * 10) / 10;

                if (!isNaN(id)) {
                    growthIndex.set(id, roundedPercentage);
                }
            },
        );
        precomputedGrowthMap = growthIndex;

        // 3. Scrub and structure the core business records array

        businessData = rawBusinessData.map(
            (
                /** @type {{ [x: string]: any; suburb_id: any; year: any; industry: any; suburb: any; }} */ d,
            ) => {
                const id = Number(String(d.suburb_id || "").trim());
                return {
                    year: String(d.year || ""),
                    industry: String(d.industry || "").replaceAll('"', ""),
                    suburb_id: isNaN(id) ? 0 : id,
                    suburb: String(d.suburb || ""),
                    sizes: {
                        "All Sizes": Number(
                            (d["Total"] || "0").replace(/\s/g, ""),
                        ),
                        "Non employing": Number(
                            (d["Non employing"] || "0").replace(/\s/g, ""),
                        ),
                        "1-4 Employees": Number(
                            (d["1-4 Employees"] || "0").replace(/\s/g, ""),
                        ),
                        "5-19 Employees": Number(
                            (d["5-19 Employees"] || "0").replace(/\s/g, ""),
                        ),
                        "20-199 Employees": Number(
                            (d["20-199 Employees"] || "0").replace(/\s/g, ""),
                        ),
                        "200+ Employees": Number(
                            (d["200+ Employees"] || "0").replace(/\s/g, ""),
                        ),
                    },
                };
            },
        );

        continuousYears = [...new Set(businessData.map((d) => d.year))].sort();
        continuousIndustries = [
            ...new Set(businessData.map((d) => d.industry)),
        ].sort();

        if (!mapContainer) return;

        map = new maplibregl.Map({
            container: mapContainer,
            style: "https://demotiles.maplibre.org/style.json",
            center: [146, -36],
            zoom: 6,
        });

        map.on("load", () => {
            if (!map) return;

            const MAPBOX_TOKEN = import.meta.env.VITE_MAPBOX_TOKEN;
            map.addSource("suburbs-source", {
                type: "vector",
                // Correct Mapbox Web API structure: /v4/username.tileset_id.json
                url: `https://api.mapbox.com/v4/${TILESET_ID}.json?secure&access_token=${MAPBOX_TOKEN}`,
                // url: `https://api.mapbox.com/v4/${MAPBOX_USERNAME}.${TILESET_ID}/tilequery/${133.7751},${-25.2744}.json`,
                // url: `https://api.mapbox.com/v4/${MAPBOX_USERNAME}.${TILESET_ID}.json?secure&access_token=${MAPBOX_TOKEN}`,
                // url: `https://mapbox.com/${MAPBOX_USERNAME}.${TILESET_ID}.json?access_token=${MAPBOX_TOKEN}`,
                promoteId: "SA2_CODE21",
                minzoom: 0,
                maxzoom: 22,
            });

            map.addLayer({
                id: "suburbs-layer",
                type: "fill",
                source: "suburbs-source",
                "source-layer": SOURCE_LAYER,
                minzoom: 4,
                maxzoom: 18,
                paint: {
                    "fill-color": [
                        "case",
                        // 1. If we are looking at growth metrics, apply the growth diverging palette
                        [
                            "boolean",
                            ["literal", mapMetricMode === "growth"],
                            false,
                        ],
                        [
                            "case",
                            ["!=", ["feature-state", "growth"], null],
                            [
                                "interpolate",
                                ["linear"],
                                ["feature-state", "growth"],
                                -5,
                                "#d7191c", // Decline (Red)
                                -1,
                                "#fdae61",
                                0,
                                "#f7f7f7", // Stagnant (Gray/White)
                                1,
                                "#a6d96a",
                                5,
                                "#1a9641", // Growth (Green)
                            ],
                            "rgba(40, 40, 40, 0.4)", // Fallback for suburbs with missing growth records
                        ],
                        // 2. Default fallback: Apply your original absolute volumetric count scale
                        [
                            "case",
                            ["!=", ["feature-state", "count"], null],
                            [
                                "interpolate",
                                ["linear"],
                                ["feature-state", "count"],
                                0,
                                "#1a2a3a",
                                10,
                                "#ffba7a",
                                25,
                                "#ff9257",
                                50,
                                "#fe662f",
                                100,
                                "#ef350b",
                                200,
                                "#cc0e00",
                                500,
                                "#a30000",
                                1000,
                                "#7a0000",
                            ],
                            "rgba(40, 40, 40, 0.4)", // Fallback for suburbs with zero business records
                        ],
                    ],
                    "fill-opacity": [
                        "case",
                        ["boolean", ["feature-state", "hover"], false],
                        1.0,
                        0.75,
                    ],
                },
            });

            map.addLayer({
                id: "suburbs-borders",
                type: "line",
                source: "suburbs-source",
                "source-layer": SOURCE_LAYER,
                paint: {
                    "line-color": [
                        "case",
                        ["boolean", ["feature-state", "hover"], false],
                        "#ffffff", // Highlight border white on hover
                        "#2a2a2a", // Dark borders normally
                    ],
                    "line-width": [
                        "case",
                        ["boolean", ["feature-state", "hover"], false],
                        1.5,
                        0.3,
                    ],
                },
            });

            // Fast, ultra-responsive mouse moving tooltip tracking at 60fps
            map.on("mousemove", "suburbs-layer", (e) => {
                if (!map || !e.features || e.features.length === 0) return;

                map.getCanvas().style.cursor = "pointer";
                const feature = e.features[0];

                const propId = feature.properties
                    ? Number(feature.properties.SA2_CODE21)
                    : null;
                const propName = feature.properties
                    ? feature.properties.SA2_NAME21
                    : "Unknown Suburb";

                // Instant O(1) matching block bypasses array searching loops
                const metrics =
                    propId !== null ? activeDataLookup.get(propId) : null;

                hoveredSuburbCount = metrics ? metrics.count : 0;
                hoveredSuburbGrowth = metrics ? metrics.growth : 0;
                hoveredSuburbName = metrics
                    ? metrics.suburbName || propName
                    : propName;

                tooltipX = e.point.x + 15;
                tooltipY = e.point.y + 15;
                showTooltip = true;

                if (feature.id !== undefined && feature.id !== null) {
                    if (hoveredFeatureId !== null) {
                        map.setFeatureState(
                            {
                                source: "suburbs-source",
                                sourceLayer: SOURCE_LAYER,
                                id: hoveredFeatureId,
                            },
                            { hover: false },
                        );
                    }
                    hoveredFeatureId = feature.id;
                    map.setFeatureState(
                        {
                            source: "suburbs-source",
                            sourceLayer: SOURCE_LAYER,
                            id: hoveredFeatureId,
                        },
                        { hover: true },
                    );
                }
            });

            map.on("mouseleave", "suburbs-layer", () => {
                if (!map) return;
                map.getCanvas().style.cursor = "";
                showTooltip = false;

                if (hoveredFeatureId !== null) {
                    map.setFeatureState(
                        {
                            source: "suburbs-source",
                            sourceLayer: SOURCE_LAYER,
                            id: hoveredFeatureId,
                        },
                        { hover: false },
                    );
                    hoveredFeatureId = null;
                }
            });

            map.on("sourcedata", (e) => {
                if (e.sourceId === "suburbs-source" && e.isSourceLoaded) {
                    updateMapData();
                }
            });
        });
    });

    function updateMapData() {
        if (!map || !map.isStyleLoaded() || businessData.length === 0) return;
        
        // 1. Wipe out old data states cleanly to clear previous configurations
        map.removeFeatureState({
            source: "suburbs-source",
            sourceLayer: SOURCE_LAYER,
        });
        
        // 2. Inject pre-computed records into feature-state keys
        activeDataLookup.forEach((metrics, numericId) => {
            const strictNumericId = Number(numericId);
            
            if (!isNaN(strictNumericId)) {
                // @ts-ignore
                map.setFeatureState(
                    {
                        source: "suburbs-source",
                        sourceLayer: SOURCE_LAYER,
                        id: strictNumericId,
                    },
                    {
                        count: metrics.count,
                        growth: metrics.growth,
                    },
                );
            }
        });

        // 3. FIX: Direct style injection updates the layer based on the active mode
        if (mapMetricMode === "growth") {
            map.setPaintProperty("suburbs-layer", "fill-color", [
                "case",
                ["!=", ["feature-state", "growth"], null],
                [
                    "interpolate",
                    ["linear"],
                    ["feature-state", "growth"],
                    -100,"#2A0000", // Deep (Dark Red)
                    -5,  "#d7191c", // Decline (Red)
                    -1,  "#fdae61", 
                    0,   "#f7f7f7", // Stagnant (Gray/White)
                    1,   "#a6d96a", 
                    5,   "#1a9641",  // Growth (Green)
                    100, "#003300",
                    400, "#001100"
                ],
                "rgba(40, 40, 40, 0.4)"
            ]);
        } else {
            map.setPaintProperty("suburbs-layer", "fill-color", [
                "case",
                ["!=", ["feature-state", "count"], null],
                [
                    "interpolate",
                    ["linear"],
                    ["feature-state", "count"],
                    0,    "#1a2a3a",
                    10,   "#ffba7a",
                    25,   "#ff9257",
                    50,   "#fe662f",
                    100,  "#ef350b",
                    200,  "#cc0e00",
                    500,  "#a30000",
                    1000, "#7a0000"
                ],
                "rgba(40, 40, 40, 0.4)"
            ]);
        }

        // 4. Keep the operational refresh trick active to smoothly render layout shifts
        map.setPaintProperty("suburbs-layer", "fill-opacity", [
            "case",
            ["boolean", ["feature-state", "hover"], false],
            1.0,
            0.75,
        ]);
    }

</script>
<!-- Import the slide animator at the very top of your file right below other imports -->
<!-- import { slide } from "svelte/transition"; -->

<main class="article-container">
    <header></header>

    <!-- Operational Dashboard Grid -->
    <section class="dashboard-grid">
        <div class="control-panel">
            
            <!-- Metric Mode Toggle Selector -->
            <div class="input-group">

                <div class="toggle-container">
                    <button 
                        class="toggle-btn" 
                        class:active={mapMetricMode === 'count'} 
                        onclick={() => mapMetricMode = 'count'}
                    >
                        Number of Businesses
                    </button>
                    <button 
                        class="toggle-btn" 
                        class:active={mapMetricMode === 'growth'} 
                        onclick={() => mapMetricMode = 'growth'}
                    >
                        Total Growth %
                    </button>
                </div>
            </div>

            <!-- Wrap drop-downs inside a reactive conditional visibility block -->
            {#if mapMetricMode === 'count'}
                <div class="animated-selectors-wrapper" transition:slide={{ duration: 300 }}>
                    <div class="input-group spacer-bottom">
                        <label for="year-select">Reporting Year</label>
                        <select id="year-select" bind:value={selectedYear}>
                            {#each continuousYears as yr}
                                <option value={yr}>{yr}</option>
                            {/each}
                        </select>
                    </div>

                    <div class="input-group spacer-bottom">
                        <label for="industry-select">Industrial Sector</label>
                        <select id="industry-select" bind:value={selectedIndustry}>
                            {#each continuousIndustries as ind}
                                <option value={ind}>{ind}</option>
                            {/each}
                        </select>
                    </div>

                    <div class="input-group spacer-bottom">
                        <label for="bucket-select">Operation Capacity</label>
                        <select id="bucket-select" bind:value={selectedBucket}>
                            {#each bucketOptions as bkt}
                                <option value={bkt}>{bkt}</option>
                            {/each}
                        </select>
                    </div>
                </div>
            {/if}

            <!-- Dynamically Updating Scannable Insights Component -->
            <div class="insights-box">
                
                
                <div class="kpi-card">
                    <span class="kpi-label">{statsSummary.label1}</span>
                    <span class="kpi-value">{statsSummary.value1}</span>
                    <span class="kpi-subtext">{statsSummary.subtext1}</span>
                </div>

                <div class="kpi-card">
                    <span class="kpi-label">{statsSummary.label2}</span>
                    <span class="kpi-value targeted">{statsSummary.value2}</span>
                    <span class="kpi-subtext">{@html statsSummary.subtext2}</span>
                </div>
            </div>
        </div>

        <!-- Main Map Canvas Frame Container -->
        <div class="map-wrapper">
            <div bind:this={mapContainer} class="map-canvas"></div>

            <!-- Dynamic Floating Legend Map Element -->
            <div class="map-legend">
                {#if mapMetricMode === 'growth'}
                    <h4>Overall Suburb Growth %</h4>
                    <div class="legend-gradient growth-scale"></div>
                    <div class="legend-labels">
                        <span>-100%</span>
                        <span>0%</span>
                        <span>+5%</span>
                        <span>+400%</span>
                    </div>
                {:else}
                    <h4>Active Business Units</h4>
                    <div class="legend-gradient count-scale"></div>
                    <div class="legend-labels">
                        <span>0</span>
                        <span>15</span>
                        <span>75</span>
                        <span>400</span>
                        <span>1k+</span>
                    </div>
                {/if}
            </div>

            <!-- Vector Mouse Anchor Tooltip Layer -->
            {#if showTooltip}
                <div class="custom-tooltip" style="left: {tooltipX}px; top: {tooltipY}px;">
                    <div class="tooltip-title">{hoveredSuburbName}</div>
                    <div class="tooltip-metric">
                        {#if mapMetricMode === 'growth'}
                            <span class="metric-number" class:negative={hoveredSuburbGrowth < 0}>
                                {hoveredSuburbGrowth > 0 ? '+' : ''}{hoveredSuburbGrowth}%
                            </span>
                            <span class="metric-unit">Overall Growth</span>
                        {:else}
                            <span class="metric-number">{hoveredSuburbCount.toLocaleString()}</span>
                            <span class="metric-unit">Registered Units</span>
                        {/if}
                    </div>
                    <div class="tooltip-footer">
                        {#if mapMetricMode === 'growth'}
                            Precomputed Baseline Trend
                        {:else}
                            {selectedBucket} • {selectedIndustry}
                        {/if}
                    </div>
                </div>
            {/if}
        </div>
    </section>
</main>


<style>
    :global(body) {
        margin: 0;
        background-color: #0f172a;
        color: #f8fafc;
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
            Helvetica, Arial, sans-serif;
    }

    .article-container {
        max-width: 1400px;
        margin: 0 auto;
        padding: 2rem;
    }

    .dashboard-grid {
        display: grid;
        grid-template-columns: 400px 1fr;
        gap: 2rem;

    }

    .control-panel {
        background-color: #1e293b;
        padding: 1.5rem;
        border-radius: 12px;
        display: flex;
        flex-direction: column;
        gap: 1.25rem;
        box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
        overflow-y: auto;
    }

    .input-group {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
    }

    label {
        font-size: 0.85rem;
        font-weight: 600;
        color: #94a3b8;
    }

    .toggle-container {
        display: grid;
        grid-template-columns: 1fr 1fr;
        background-color: #0f172a;
        padding: 0.25rem;
        border-radius: 8px;
        border: 1px solid #334155;
    }

    .toggle-btn {
        background: transparent;
        color: #94a3b8;
        border: none;
        padding: 0.5rem;
        font-size: 0.85rem;
        font-weight: 500;
        border-radius: 6px;
        cursor: pointer;
        transition: all 0.2s ease;
    }

    .toggle-btn.active {
        background-color: #334155;
        color: #f8fafc;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
    }

    select {
        background-color: #0f172a;
        color: #f8fafc;
        border: 1px solid #334155;
        padding: 0.75rem;
        border-radius: 6px;
        font-size: 0.95rem;
        cursor: pointer;
        transition: border-color 0.2s;
    }

    select:focus {
        outline: none;
        border-color: #fe662f;
    }

    .insights-box {
        margin-top: auto;
        border-top: 1px solid #334155;
        padding-top: 1.25rem;
    }

    .insights-box h3 {
        font-size: 1rem;
        margin: 0 0 1rem 0;
        color: #cbd5e1;
    }

    .kpi-card {
        background-color: #0f172a;
        padding: 1rem;
        border-radius: 8px;
        margin-bottom: 0.75rem;
        display: flex;
        flex-direction: column;
    }

    .kpi-label {
        font-size: 0.75rem;
        color: #64748b;
        text-transform: uppercase;
    }

    .kpi-value {
        font-size: 1.5rem;
        font-weight: 700;
        color: #f8fafc;
        margin: 0.25rem 0;
    }

    .kpi-value.targeted {
        font-size: 1.15rem;
        color: #ffba7a;
    }

    .kpi-subtext {
        font-size: 0.8rem;
        color: #94a3b8;
    }

    .map-wrapper {
        position: relative;
        min-width: 600px;
        border-radius: 12px;
        overflow: hidden;
        box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.3);
    }

    .map-canvas {
        width: 100%;
        height: 100%;
    }

    .map-legend {
        position: absolute;
        bottom: 1.5rem;
        right: 1.5rem;
        background-color: rgba(15, 23, 42, 0.9);
        backdrop-filter: blur(8px);
        padding: 1rem;
        border-radius: 8px;
        border: 1px solid #334155;
        width: 240px;
        pointer-events: none;
        z-index: 10;
    }

    .map-legend h4 {
        margin: 0 0 0.5rem 0;
        font-size: 0.8rem;
        color: #cbd5e1;
    }

    .legend-gradient {
        height: 12px;
        border-radius: 4px;
        margin-bottom: 0.25rem;
    }

    .legend-gradient.count-scale {
        background: linear-gradient(
            to right,
            #1a2a3a,
            #ffba7a,
            #ff9257,
            #fe662f,
            #ef350b,
            #cc0e00,
            #a30000,
            #7a0000
        );
    }

    .legend-gradient.growth-scale {
        background: linear-gradient(
            to right,
            #d7191c,
            #fdae61,
            #f7f7f7,
            #a6d96a,
            #1a9641
        );
    }

    .legend-labels {
        display: flex;
        justify-content: space-between;
        font-size: 0.75rem;
        color: #94a3b8;
    }

    .custom-tooltip {
        position: absolute;
        background-color: rgba(15, 23, 42, 0.95);
        backdrop-filter: blur(6px);
        border: 1px solid #334155;
        padding: 0.75rem;
        border-radius: 8px;
        color: #f8fafc;
        pointer-events: none;
        z-index: 100;
        box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.5);
        min-width: 200px;
    }

    .tooltip-title {
        font-weight: 600;
        font-size: 0.9rem;
        margin-bottom: 0.25rem;
        color: #cbd5e1;
    }
    .tooltip-metric {
        display: flex;
        align-items: baseline;
        gap: 0.5rem;
        margin-bottom: 0.25rem;
    }
    .metric-number {
        font-size: 1.4rem;
        font-weight: 700;
        color: #fe662f;
    }
    .metric-number.negative {
        color: #f87171;
    }
    .metric-unit {
        font-size: 0.8rem;
        color: #94a3b8;
    }
    .tooltip-footer {
        font-size: 0.7rem;
        color: #64748b;
        border-top: 1px solid #334155;
        padding-top: 0.25rem;
        margin-top: 0.25rem;
    }

        .animated-selectors-wrapper {
        display: flex;
        flex-direction: column;
        gap: 0.25rem;
        overflow: hidden; /* Guards animation layout containment boundaries */
    }

    .spacer-bottom {
        margin-bottom: 0.25rem;
    }

</style>
