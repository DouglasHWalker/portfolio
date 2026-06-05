<script>
  import { onMount } from 'svelte';
  import maplibregl from 'maplibre-gl';
  import 'maplibre-gl/dist/maplibre-gl.css';
  import * as d3 from 'd3';

  /**
   * @typedef {Object} BusinessRow
   * @property {string} year
   * @property {string} industry
   * @property {string} suburb_id
   * @property {string} suburb
   * @property {number} Non_employing
   * @property {number} Employees1_4
   * @property {number} Employees5_19
   * @property {number} Employees20_199
   * @property {number} Employees200
   */

  /** @type {HTMLDivElement | undefined} */
  let mapContainer = undefined;

  /** @type {maplibregl.Map | null} */
  let map = null;

  /** @type {BusinessRow[]} */
  let businessData = [];
  
  // UI Selection States
  /** @type {string} */
  let selectedYear = '2025';

  /** @type {string} */
  let selectedIndustry = 'Agriculture, Forestry and Fishing';

  /** @type {string} */
  let selectedBucket = '1-4 Employees'; // Matches column keys dynamically

  const MAPBOX_USERNAME = 'atlasgigernzer';
  const TILESET_ID = 'atlasgigerenzer.6qx896'; 
  const SOURCE_LAYER = 'SA2_2021_AUST_SHP_GDA2020';

  onMount(async () => {
    const basePath = import.meta.env.BASE_URL;
    const rawData = await d3.csv(`${basePath}data/business_counts.csv`);
    
    
    businessData = rawData.map((/** @type {Record<string, string | undefined>} */ d) => ({
      year: String(d.year || ''),
      industry: String(d.industry || ''),
      suburb_id: String(d.suburb_id || ''),
      suburb: String(d.suburb || ''),
      Non_employing: Number((d['Non employing'] || '0').replace(/\s/g, '')),
      Employees1_4: Number((d['1-4 Employees'] || '0').replace(/\s/g, '')),
      Employees5_19: Number((d['5-19 Employees'] || '0').replace(/\s/g, '')),
      Employees20_199: Number((d['20-199 Employees'] || '0').replace(/\s/g, '')),
      Employees200: Number((d['200+ Employees'] || '0').replace(/\s/g, ''))
    }));

    // CRITICAL FIX: Ensure the container is ready before mounting MapLibre
    if (!mapContainer) return;

    map = new maplibregl.Map({
      container: mapContainer,
      style: 'https://cartocdn.com',
      center: [133.7751, -25.2744], 
      zoom: 4
    });

    map.on('load', () => {
      if (!map) return;

      map.addSource('suburbs-source', {
        type: 'vector',
        url: `mapbox://${MAPBOX_USERNAME}.${TILESET_ID}`
      });

      map.addLayer({
        id: 'suburbs-layer',
        type: 'fill',
        source: 'suburbs-source',
        'source-layer': SOURCE_LAYER,
        paint: {
          'fill-color': [
            'case',
            ['!=', ['feature-state', 'count'], null],
            [
              'interpolate',
              ['linear'],
              ['feature-state', 'count'],
              0, '#1a2a3a',      
              10, '#e65c00',    
              100, '#ff3333'    
            ],
            'rgba(40, 40, 40, 0.4)' 
          ],
          'fill-opacity': 0.8
        }
      });

      map.addLayer({
        id: 'suburbs-borders',
        type: 'line',
        source: 'suburbs-source',
        'source-layer': SOURCE_LAYER,
        paint: {
          'line-color': '#2a2a2a', 
          'line-width': 0.3
        }
      });

      updateMapData();
    });
  });

  // Watch all three dropdown variables reactively
  $: if (businessData.length > 0 && map && map.isStyleLoaded() && selectedYear && selectedIndustry && selectedBucket) {
    updateMapData();
  }

  function updateMapData() {
    if (!map) return;

    // 1. Filter rows by Year AND Industry
    const currentData = businessData.filter(
      d => d.year === selectedYear && d.industry === selectedIndustry
    );

    // 2. Clear out the old map state before painting new data
    // Mapbox saves state, so we must reset everything back to 0 first
    businessData.forEach(row => {
      /** @type {maplibregl.Map} */(map).setFeatureState(
        { source: 'suburbs-source', sourceLayer: SOURCE_LAYER, id: row.suburb_id },
        { count: 0 }
      );
    });

    // 3. Push the active bucket value into Mapbox
    currentData.forEach(row => {
      // Access the bucket column dynamically using the dropdown string key
      const countValue = row[/** @type {keyof BusinessRow} */(selectedBucket)] || 0;

      /** @type {maplibregl.Map} */(map).setFeatureState(
        {
          source: 'suburbs-source',
          sourceLayer: SOURCE_LAYER,
          id: row.suburb_id
        },
        { count: countValue }
      );
    });
  }
</script>
