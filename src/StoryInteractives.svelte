<script>
    import { onMount } from "svelte";
    import maplibregl from "maplibre-gl";
    import "maplibre-gl/dist/maplibre-gl.css";
    import * as d3 from "d3";

    /**
     * @typedef {Object} BusinessRow
     * @property {string} year
     * @property {string} industry
     * @property {string} suburb_id
     * @property {string} suburb
     * @property {number} [Total]
     * @property {number} [Non employing]
     * @property {number} [1-4 Employees]
     * @property {number} [5-19 Employees]
     * @property {number} [20-199 Employees]
     * @property {number} [200+ Employees]
     */

    /** @type {HTMLDivElement | undefined} */
    let mapContainer = undefined;
    /** @type {maplibregl.Map | null} */
    let map = null;

    // Core Reactive Business State Arrays
    /**
     * @type {any[]}
     */
    let businessData = $state([]);
    /**
     * @type {any[] | null | undefined}
     */
    let continuousYears = $state([]);
    /**
     * @type {any[] | null | undefined}
     */
    let continuousIndustries = $state([]);

    // Reactive UI Selection Bindings
    let selectedYear = $state("2025");
    let selectedIndustry = $state("Accommodation and Food Services");
    let selectedBucket = $state("All Sizes");

    const bucketOptions = [
        "All Sizes",
        "Non employing",
        "1-4 Employees",
        "5-19 Employees",
        "20-199 Employees",
        "200+ Employees",
    ];

    // Tooltip State Hooks
    let hoveredSuburbName = $state("");
    let hoveredSuburbCount = $state(0);
    let tooltipX = $state(0);
    let tooltipY = $state(0);
    let showTooltip = $state(false);
    /** @type {string | number | null} */
    let hoveredFeatureId = null;

    // Fixed Vector Asset Target Constants
    const TILESET_ID = "atlasgigerenzer.6qx896";
    const SOURCE_LAYER = "SA2_2021_AUST_SHP_GDA2020.zip-mg2r8x";

    // 1. Reactive Data Filter Node
    let filteredRecords = $derived(
        businessData.filter(
            (d) => d.year === selectedYear && d.industry === selectedIndustry,
        ),
    );

    // 2. Automated Editorial Insight Engine
    let statsSummary = $derived({
        totalBusinesses: filteredRecords.reduce(
            (acc, curr) => acc + (curr[selectedBucket] || 0),
            0,
        ),
        topSuburb: filteredRecords.reduce(
            (max, curr) =>
                (curr[selectedBucket] || 0) > (max[selectedBucket] || 0)
                    ? curr
                    : max,
            { suburb: "None", [selectedBucket]: 0 },
        ),
    });

    // 3. Native Map Pipeline Trigger
    $effect(() => {
        if (map && map.isStyleLoaded() && filteredRecords.length > 0) {
            updateMapFeatures();
            updateMapData();
        }
    });
    onMount(async () => {
        const basePath = import.meta.env.BASE_URL;
        const rawData = await d3.csv(`${basePath}data/business_counts.csv`);

        // Scrub layout strings and sanitize spaces
        businessData = rawData.map((/** @type {Record<string, string | undefined>} */d) => ({
            year: String(d.year || ""),
            industry: String(d.industry || "").replaceAll('"', ""),
            suburb_id: String(d.suburb_id || ""),
            suburb: String(d.suburb || ""),
            "All Sizes": Number((d["Total"] || "0").replace(/\s/g, "")),
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
        }));

        continuousYears = [...new Set(businessData.map((d) => d.year))].sort();
        continuousIndustries = [
            ...new Set(businessData.map((d) => d.industry)),
        ].sort();

        // Map init phase stubbed out here for clarity (Fully written in Part 2)

        if (!mapContainer) return;

        map = new maplibregl.Map({
            container: mapContainer,
            style: "https://demotiles.maplibre.org/style.json",
            center: [146, -36],
            zoom: 6,
        });

        map.on("load", () => {
            if (!map) return;

            // Define your Mapbox Token as a standard string variable
            const MAPBOX_TOKEN = import.meta.env.VITE_MAPBOX_TOKEN;
            map.addSource("suburbs-source", {
                type: "vector",
                // Correct Mapbox Web API structure: /v4/username.tileset_id.json
                url: `https://api.mapbox.com/v4/${TILESET_ID}.json?secure&access_token=${MAPBOX_TOKEN}`,
                // url: `https://api.mapbox.com/v4/${MAPBOX_USERNAME}.${TILESET_ID}/tilequery/${133.7751},${-25.2744}.json`,
                // url: `https://api.mapbox.com/v4/${MAPBOX_USERNAME}.${TILESET_ID}.json?secure&access_token=${MAPBOX_TOKEN}`,
                // url: `https://mapbox.com/${MAPBOX_USERNAME}.${TILESET_ID}.json?access_token=${MAPBOX_TOKEN}`,
                promoteId: "SA2_CODE21",
                minzoom: 0, // Don't waste network requests if zoomed out to a global worldview
                maxzoom: 22,
            });

            map.addLayer({
                id: "suburbs-layer",
                type: "fill",
                source: "suburbs-source",
                "source-layer": SOURCE_LAYER,

                // --- ADD THESE LINES ---
                minzoom: 4, // Suburbs only fade into view once the user zooms into country-scale [2]
                maxzoom: 18, // Completely hide the tiles if they zoom past extreme street levels

                paint: {
                    "fill-color": [
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
                        "rgba(40, 40, 40, 0.4)",
                    ],
                    "fill-opacity": [
                        "case",
                        ["boolean", ["feature-state", "hover"], false],
                        1.0, // Fully opaque on hover
                        0.75, // Semi-transparent normally
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

            // --- Interactive Mouse Handlers for Tooltip ---
            map.on("mousemove", "suburbs-layer", (e) => {
                if (!map || !e.features || e.features.length === 0) return;

                map.getCanvas().style.cursor = "pointer";
                const feature = e.features[0];

                // Match the feature ID with your CSV layout structure data map
                // Mapbox vector tile attributes live inside feature.properties
                const propId = feature.properties
                    ? feature.properties.SA2_CODE21
                    : null;
                const propName = feature.properties
                    ? feature.properties.SA2_NAME21
                    : "Unknown Suburb";

                // Find matched business count data record
                const matchedRecord = businessData.find(
                    (d) =>
                        d.suburb_id === String(propId) &&
                        d.year === selectedYear &&
                        d.industry === selectedIndustry,
                );

                // FIX: Cast the dynamic lookup inside Number() to guarantee it compiles as a number primitive
                const count = matchedRecord
                    ? Number(
                          matchedRecord[
                              /** @type {keyof BusinessRow} */ (selectedBucket)
                          ] || 0,
                      )
                    : 0;

                hoveredSuburbName = matchedRecord
                    ? matchedRecord.suburb || propName
                    : propName;
                hoveredSuburbCount = count;

                // Pin tooltip coordinates to mouse position offsets
                tooltipX = e.point.x + 15;
                tooltipY = e.point.y + 15;
                showTooltip = true;

                // Visual hover boundary stroke updates
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
                // Only run when our specific source finishes loading its vector data frames
                if (e.sourceId === "suburbs-source" && e.isSourceLoaded) {
                    updateMapData();
                }
            });
        });
    });

    // Re-run the data join whenever the user changes any dropdown selection
    //  $effect(() => {
    //     // Safe check to ensure the data is loaded and the MapLibre canvas style is ready
    //     if (businessData.length > 0 && map && map.isStyleLoaded()) {
    //         // Any reactive states read inside here (like selectedYear, selectedBucket) 
    //         // will automatically trigger this effect when their values change.
    //         updateMapData();
    //     }
    // });

    function updateMapFeatures() {
        if (!map || !map.isStyleLoaded()) return;

        filteredRecords.forEach((record) => {
            // @ts-ignore
            map.setFeatureState(
                {
                    source: "suburbs-source",
                    sourceLayer: SOURCE_LAYER,
                    id: record.suburb_id,
                },
                { count: record[selectedBucket] || 0 },
            );
        });
    }

    function updateMapData() {
        if (!map || !map.isStyleLoaded()) return;

        // 1. Wipe out old data states cleanly to clear previous shapes
        map.removeFeatureState({
            source: "suburbs-source",
            sourceLayer: SOURCE_LAYER,
        });

        // 2. Filter records down to selected categories
        const currentData = businessData.filter(
            (d) => d.year === selectedYear && d.industry === selectedIndustry,
        );

        // 3. Inject matching metrics down to promoted Mapbox feature keys
        currentData.forEach((row) => {
            const countValue = Number(
                row[/** @type {keyof BusinessRow} */ (selectedBucket)] || 0,
            );

            // Cast ID to number if it is a numeric string to match Mapbox promotedId types
            const targetId = isNaN(Number(row.suburb_id))
                ? row.suburb_id
                : Number(row.suburb_id);

            /** @type {maplibregl.Map} */ (map).setFeatureState(
                {
                    source: "suburbs-source",
                    sourceLayer: SOURCE_LAYER,
                    id: targetId,
                },
                { count: countValue },
            );
        });
    }
</script>
<main class="article-container">
    <header></header>

    <!-- Operational Dashboard Grid -->
    <section class="dashboard-grid">
        <div class="control-panel">
            <div class="input-group">
                <label for="year-select">Reporting Year</label>
                <select id="year-select" bind:value={selectedYear}>
                    {#each continuousYears as yr}
                        <option value={yr}>{yr}</option>
                    {/each}
                </select>
            </div>

            <div class="input-group">
                <label for="industry-select">Industrial Sector</label>
                <select id="industry-select" bind:value={selectedIndustry}>
                    {#each continuousIndustries as ind}
                        <option value={ind}>{ind}</option>
                    {/each}
                </select>
            </div>

            <div class="input-group">
                <label for="bucket-select">Operation Capacity</label>
                <select id="bucket-select" bind:value={selectedBucket}>
                    {#each bucketOptions as bkt}
                        <option value={bkt}>{bkt}</option>
                    {/each}
                </select>
            </div>

            <!-- Scannable Insights Component -->
            <div class="insights-box">
                <h3>Quick Insights ({selectedYear})</h3>
                
                <div class="kpi-card">
                    <span class="kpi-label">Total Sector Footprint</span>
                    <span class="kpi-value">{statsSummary.totalBusinesses.toLocaleString()}</span>
                    <span class="kpi-subtext">Active entities Nationwide</span>
                </div>

                <div class="kpi-card">
                    <span class="kpi-label">Top Commercial Hotspot</span>
                    <span class="kpi-value targeted">{statsSummary.topSuburb.suburb}</span>
                    <span class="kpi-subtext">
                        Hosts <strong>{(statsSummary.topSuburb[selectedBucket] || 0).toLocaleString()}</strong> active units
                    </span>
                </div>
            </div>
        </div>

        <!-- Main Map Interactive Canvas Wrapper -->
        <div class="map-wrapper">
            <div bind:this={mapContainer} class="map-canvas"></div>

            <!-- Dynamic Floating Legend Map Element -->
            <div class="map-legend">
                <h4>Active Business Units</h4>
                <div class="legend-gradient"></div>
                <div class="legend-labels">
                    <span>0</span>
                    <span>15</span>
                    <span>75</span>
                    <span>400</span>
                    <span>1k+</span>
                </div>
            </div>

            <!-- Vector Mouse Anchor Tooltip Layer -->
            {#if showTooltip}
                <div class="custom-tooltip" style="left: {tooltipX}px; top: {tooltipY}px;">
                    <div class="tooltip-title">{hoveredSuburbName}</div>
                    <div class="tooltip-metric">
                        <span class="metric-number">{hoveredSuburbCount.toLocaleString()}</span>
                        <span class="metric-unit">Registered Units</span>
                    </div>
                    <div class="tooltip-footer">{selectedBucket} • {selectedIndustry}</div>
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
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
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
        height: 650px;
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
        width: 220px;
        pointer-events: none;
    }

    .map-legend h4 {
        margin: 0 0 0.5rem 0;
        font-size: 0.8rem;
        color: #cbd5e1;
    }

    .legend-gradient {
        height: 12px;
        border-radius: 4px;
        background: linear-gradient(to right, #1a2a3a, #ffba7a, #ff9257, #fe662f, #ef350b, #cc0e00, #a30000, #7a0000);
    }

    .legend-labels {
        display: flex;
        justify-content: space-between;
        font-size: 0.7rem;
        color: #94a3b8;
        margin-top: 0.35rem;
    }

    .custom-tooltip {
        position: absolute;
        z-index: 1000;
        pointer-events: none;
        background-color: rgba(15, 23, 42, 0.95);
        border: 1px solid #fe662f;
        border-radius: 6px;
        padding: 0.75rem 1rem;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.5);
        min-width: 180px;
        transform: translate(5px, 5px);
        backdrop-filter: blur(4px);
    }

    .tooltip-title {
        font-weight: 700;
        font-size: 0.9rem;
        color: #f8fafc;
        margin-bottom: 0.25rem;
    }

    .tooltip-metric {
        display: flex;
        align-items: baseline;
        gap: 0.35rem;
    }

    .metric-number {
        font-size: 1.4rem;
        font-weight: 800;
        color: #fe662f;
    }

    .metric-unit {
        font-size: 0.75rem;
        color: #94a3b8;
    }

    .tooltip-footer {
        font-size: 0.65rem;
        color: #64748b;
        margin-top: 0.5rem;
        border-top: 1px solid #334155;
        padding-top: 0.25rem;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }

    @media (max-width: 1024px) {
        .dashboard-grid {
            grid-template-columns: 1fr;
            height: auto;
        }
        .map-wrapper {
            height: 450px;
        }
        .control-panel {
            height: auto;
        }
    }
</style>

