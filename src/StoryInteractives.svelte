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

    /** @type {BusinessRow[]} */
    let businessData = [];

    // UI Selection States
    /** @type {string} */
    let selectedYear = "2025";
    /** @type {string} */
    let selectedIndustry = "Agriculture, Forestry and Fishing";
    /** @type {string} */
    let selectedBucket = "1-4 Employees";

    // Tooltip Interactive State
    let hoveredSuburbName = "";
    let hoveredSuburbCount = 0;
    let tooltipX = 0;
    let tooltipY = 0;
    let showTooltip = false;
    /** @type {string | number | null} */
    let hoveredFeatureId = null;

    // Configuration (Matching your Mapbox asset layers)
    const MAPBOX_USERNAME = "atlasgigernzer";
    const TILESET_ID = "6qx896";
    const SOURCE_LAYER = "SA2_2021_AUST_SHP_GDA2020";

    onMount(async () => {
        const basePath = import.meta.env.BASE_URL;
        const rawData = await d3.csv(`${basePath}data/business_counts.csv`);

        businessData = rawData.map(
            (/** @type {Record<string, string | undefined>} */ d) => ({
                year: String(d.year || ""),
                industry: String(d.industry || ""),
                suburb_id: String(d.suburb_id || ""),
                suburb: String(d.suburb || ""),
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
            }),
        );

        if (!mapContainer) return;

        map = new maplibregl.Map({
            container: mapContainer,
            style: "https://demotiles.maplibre.org/style.json",
            center: [133.7751, -25.2744],
            zoom: 4,
        });

        map.on("load", () => {
            if (!map) return;

            // Define your Mapbox Token as a standard string variable
            const MAPBOX_TOKEN = import.meta.env.VITE_MAPBOX_TOKEN;
            map.addSource("suburbs-source", {
                type: "vector",
                // Correct Mapbox Web API structure: /v4/username.tileset_id.json
                // url: `https://api.mapbox.com/v4/${MAPBOX_USERNAME}.${TILESET_ID}/tilequery/${133.7751},${-25.2744}.json`,
                url: `https://api.mapbox.com/v4/${MAPBOX_USERNAME}.${TILESET_ID}.json?secure&access_token=${MAPBOX_TOKEN}`,
                // url: `https://mapbox.com/${MAPBOX_USERNAME}.${TILESET_ID}.json?access_token=${MAPBOX_TOKEN}`,
            });

            map.addLayer({
                id: "suburbs-layer",
                type: "fill",
                source: "suburbs-source",
                "source-layer": SOURCE_LAYER,
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
                            "#e65c00",
                            100,
                            "#ff3333",
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

            updateMapData();
        });
    });

    $: if (
        businessData.length > 0 &&
        map &&
        map.isStyleLoaded() &&
        selectedYear &&
        selectedIndustry &&
        selectedBucket
    ) {
        updateMapData();
    }

    function updateMapData() {
        if (!map) return;

        const currentData = businessData.filter(
            (d) => d.year === selectedYear && d.industry === selectedIndustry,
        );

        // Clear previous settings
        businessData.forEach((row) => {
            /** @type {maplibregl.Map} */ (map).setFeatureState(
                {
                    source: "suburbs-source",
                    sourceLayer: SOURCE_LAYER,
                    id: row.suburb_id,
                },
                { count: null },
            );
        });

        // Populate current data parameters
        currentData.forEach((row) => {
            const countValue =
                row[/** @type {keyof BusinessRow} */ (selectedBucket)] || 0;
            /** @type {maplibregl.Map} */ (map).setFeatureState(
                {
                    source: "suburbs-source",
                    sourceLayer: SOURCE_LAYER,
                    id: row.suburb_id,
                },
                { count: countValue },
            );
        });
    }
</script>

<div class="map-wrap">
    <!-- Clean Editorial Typography Controls Overlay -->
    <div class="controls-card">
        <h3>Filter Business Densities</h3>

        <div class="control-group">
            <label for="year-select">Data Year</label>
            <select id="year-select" bind:value={selectedYear}>
                <option value="2024">2024</option>
                <option value="2025">2025</option>
                <option value="2026">2026</option>
            </select>
        </div>

        <div class="control-group">
            <label for="industry-select">Industry Classification</label>
            <select id="industry-select" bind:value={selectedIndustry}>
                <option value="Agriculture, Forestry and Fishing"
                    >Agriculture, Forestry & Fishing</option
                >
                <option value="Retail Trade">Retail Trade</option>
                <option value="Professional, Scientific and Technical Services"
                    >Professional & Technical Services</option
                >
            </select>
        </div>

        <div class="control-group">
            <label for="bucket-select">Employee Bandwidth Size</label>
            <select id="bucket-select" bind:value={selectedBucket}>
                <option value="Non employing">Non Employing</option>
                <option value="1-4 Employees">1–4 Employees</option>
                <option value="5-19 Employees">5–19 Employees</option>
                <option value="20-199 Employees">20–199 Employees</option>
                <option value="200+ Employees">200+ Employees</option>
            </select>
        </div>
    </div>

    <!-- The Map Canvas Box Wrapper Target -->
    <div bind:this={mapContainer} class="map-container"></div>

    <!-- Interactive Client-side Hover Floating Tooltip Portal -->
    {#if showTooltip}
        <div
            class="floating-tooltip"
            style="top: {tooltipY}px; left: {tooltipX}px;"
        >
            <div class="tooltip-title">{hoveredSuburbName}</div>
            <div class="tooltip-data-row">
                <span class="label">{selectedBucket}:</span>
                <span class="value">{hoveredSuburbCount.toLocaleString()}</span>
            </div>
        </div>
    {/if}
</div>

<style>
    /* CRITICAL FIX: The elements must dictate absolute height boundaries */
    .map-wrap {
        position: relative;
        width: 100%;
        height: 75vh; /* Direct vertical height allocation prevents 0px layout compression */
        min-height: 550px;
        background-color: #0f172a;
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
            Helvetica, Arial, sans-serif;
    }

    .map-container {
        width: 100%;
        height: 100%;
    }

    /* Professional Data Journalism Layer Panel Styling Overlay */
    .controls-card {
        position: absolute;
        top: 20px;
        left: 20px;
        z-index: 5;
        background-color: rgba(15, 23, 42, 0.9);
        backdrop-filter: blur(8px);
        border: 1px solid rgba(255, 255, 255, 0.1);
        border-radius: 8px;
        padding: 16px;
        width: 280px;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.4);
        color: #f8fafc;
    }
    .controls-card h3 {
        margin: 0 0 14px 0;
        font-size: 1.1rem;
        font-weight: 600;
        letter-spacing: -0.01em;
        border-bottom: 1px solid rgba(255, 255, 255, 0.1);
        padding-bottom: 8px;
    }
    .control-group {
        display: flex;
        flex-direction: column;
        gap: 4px;
        margin-bottom: 12px;
    }
    .control-group:last-child {
        margin-bottom: 0;
    }
    .control-group label {
        font-size: 0.75rem;
        text-transform: uppercase;
        letter-spacing: 0.05em;
        color: #94a3b8;
        font-weight: 500;
    }
    .control-group select {
        background-color: #1e293b;
        border: 1px solid #334155;
        border-radius: 4px;
        color: #f8fafc;
        padding: 8px;
        font-size: 0.85rem;
        cursor: pointer;
        transition: border-color 0.2s;
    }
    .control-group select:focus {
        outline: none;
        border-color: #38bdf8;
    }
    /* Interactive Data Tooltip Styling Box*/
    .floating-tooltip {
        position: absolute;
        z-index: 10;
        pointer-events: none;
        background-color: #ffffff;
        color: #0f172a;
        padding: 10px 14px;
        border-radius: 6px;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.25);
        min-width: 150px;
        font-size: 0.85rem;
        border: 1px solid #e2e8f0;
    }
    .tooltip-title {
        font-weight: 700;
        font-size: 0.9rem;
        margin-bottom: 4px;
        color: #0f172a;
        border-bottom: 1px solid #e2e8f0;
        padding-bottom: 4px;
    }
    .tooltip-data-row {
        display: flex;
        justify-content: space-between;
        gap: 12px;
    }
    .tooltip-data-row .label {
        color: #64748b;
    }
    .tooltip-data-row .value {
        font-weight: 600;
        color: #0284c7;
    }
</style>
