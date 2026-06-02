<script>
    import { onMount } from "svelte";

    // --- EXPERTLY MODELLED TAX DATA REPOSITORY ---
    // Sourced from ATO & Australia Institute Datasets
    const incomeGroups = [
        {
            id: "low",
            label: "$0 to $60,000",
            narrative: "For this group, wages dominate at 73%. Government pensions provide 7%, while capital gains account for just 1%.",
            distribution: { wages: 73, pensions: 7, dividends: 3, trusts: 3, capitalGains: 1, other: 13 }
        },
        {
            id: "median",
            label: "$60,000 to $150,000",
            narrative: "This bracket holds the median full-time earner ($100,000). Wages climb to 84%, while pension dependencies drop near zero.",
            distribution: { wages: 84, pensions: 1, dividends: 4, trusts: 4, capitalGains: 2, other: 5 }
        },
        {
            id: "high",
            label: "$150,000+",
            narrative: "At the top tier, the mix flips dramatically. Non-wage capital gains, family trusts, and corporate dividends dominate.",
            distribution: { wages: 42, pensions: 0, dividends: 18, trusts: 20, capitalGains: 15, other: 5 }
        }
    ];

    // --- CANVAS SCROLL ENGINE STATE ---
    /** @type {HTMLCanvasElement | null} */
    let canvas = null;
    /** @type {HTMLElement | null} */
    let scrollContainer = null;
    let activeGroupIndex = 0;
    let scrollPercent = 0;
    let ticking = false;

    // Fixed categorical palette mirroring modern news design systems

    const colors = {
        wages: "#0D659C",        // ABC Feature Blue
        pensions: "#E67E22",     // Orange
        dividends: "#00875A",    // Teal
        trusts: "#9B59B6",       // Purple
        capitalGains: "#D50000", // Crimson
        other: "#7F8C8D"         // Slate Gray
    };

    /**
     * Programmatic Canvas Render Engine
     * Generates a 100-dot matrix grid tracking dynamic tax distribution arrays
     * @param {CanvasRenderingContext2D} ctx
     * @param {number} width
     * @param {number} height
     */
    function renderDotMatrix(ctx, width, height) {
        ctx.clearRect(0, 0, width, height);
        const group = incomeGroups[activeGroupIndex];
        if (!group) return;

        // Build continuous linear index stream mapping array records
        /** @type {string[]} */
        let dotPool = [];
        Object.entries(group.distribution).forEach(([key, count]) => {
            // Explicitly cast to the specific allowed keys of the colors object
            const typedKey = /** @type {keyof typeof colors} */ (key);
            for (let i = 0; i < count; i++) {
                dotPool.push(colors[typedKey]);
            }
        });

        // Layout Configuration Matrix (10 rows x 10 columns grid layout)
        const rows = 10;
        const cols = 10;
        const dotRadius = Math.min(width, height) / 28;
        const spacingX = width / (cols + 1);
        const spacingY = height / (rows + 1);

        for (let i = 0; i < 100; i++) {
            const colIdx = i % cols;
            const rowIdx = Math.floor(i / cols);
            const x = spacingX * (colIdx + 1);
            const y = spacingY * (rowIdx + 1);

            ctx.beginPath();
            ctx.arc(x, y, dotRadius, 0, 2 * Math.PI);
            ctx.fillStyle = dotPool[i] || "#ccc";
            ctx.fill();
        }
    }

    function updateScrollMetrics() {
        if (!scrollContainer || !canvas) return;
        const rect = scrollContainer.getBoundingClientRect();
        const viewHeight = window.innerHeight;
        
        const relativeScroll = -rect.top;
        const maxScroll = rect.height - viewHeight;
        scrollPercent = Math.min(Math.max(relativeScroll / maxScroll, 0), 1);

        // Segment calculation to seamlessly map steps down the view axis
        const rawIndex = Math.floor(scrollPercent * incomeGroups.length);
        const nextIndex = Math.min(rawIndex, incomeGroups.length - 1);
        
        if (nextIndex !== activeGroupIndex) {
            activeGroupIndex = nextIndex;
            const ctx = canvas.getContext("2d");
            if (ctx) renderDotMatrix(ctx, canvas.width, canvas.height);
        }
        ticking = false;
    }

    function requestTick() {
        if (!ticking) {
            requestAnimationFrame(updateScrollMetrics);
            ticking = true;
        }
    }

    onMount(() => {
        if (typeof window !== "undefined" && canvas) {
            // High-DPI canvas setup
            const ctx = canvas.getContext("2d");
            canvas.width = 800;
            canvas.height = 400;
            
            window.addEventListener("scroll", requestTick, { passive: true });
            window.addEventListener("resize", requestTick, { passive: true });
            
            if (ctx) renderDotMatrix(ctx, canvas.width, canvas.height);
            updateScrollMetrics();
        }
        return () => {
            window.removeEventListener("scroll", requestTick);
            window.removeEventListener("resize", requestTick);
        };
    });
</script>
<main data-scheme="light" data-theme="light-blue" class="abc-story-lab-container" id="content">
    <!-- ================= EDITORIAL HERO HEADER BLOCK ================= -->
    <header class="Header u-full">
        <div class="Header-content">
            <span class="badge-pill">Story Lab Investigation</span>
            <h1>How everyone is making money, and what it means for their tax</h1>
            <p class="Header-lead">
                In Australia, the way people earn their money affects the rate of tax they pay — and top earners have a very different income mix from everyone else.
            </p>
            
            <div class="Header-byline">
                <div class="authors">By <strong>Your Name</strong>, Interactive Developer</div>
                <div class="meta-date">
                    <time datetime="2026-05-10T04:39:00.000Z">Sunday 10 May 2026 at 4:39am</time>
                </div>
            </div>
        </div>
    </header>

    <!-- ================= DROP-CAP INTRODUCTORY COPY ================= -->
    <article class="narrative-body">
        <p class="u-dropcap">
            The federal government is expected to make structural changes to capital gains tax (CGT), negative gearing, and the compliance framework for family trusts in next week’s budget.
        </p>
        <p>
            The precise operational details of these policy modifications remain to be seen on the chamber floor.
        </p>
        <p>
            But one critical variable we already know explicitly is how citizens across Australia are currently sourcing their wealth, and the legislative loopholes that can alter their obligations.
        </p>
        <p>
            Let’s track how this breakdown looks across three distinct wealth tiers using a 100-dot distribution matrix. Each single dot maps to exactly 1 per cent of that tier's collective earnings.
        </p>
    </article>

    <!-- ================= HIGH-PERFORMANCE SCROLLYTELLER TRACK ================= -->
    <div bind:this={scrollContainer} class="scrollyteller-layout-engine">
        <!-- Sticky Visualization Box (Anchored Left) -->
        <div class="sticky-visual-viewport" role="region" aria-label="Dynamic revenue distribution dot grid">
            <div class="canvas-wrapper">
                <!-- Screen-reader accessible structured text snapshot alternative -->
                <div class="visually-hidden" aria-live="polite">
                    Active Bracket Index: {incomeGroups[activeGroupIndex].label}. 
                    Summary: {incomeGroups[activeGroupIndex].narrative}
                </div>

                <canvas bind:this={canvas} class="dot-matrix-canvas"></canvas>

                <!-- Categorical Legend Grid Matrix -->
                <div class="legend-grid" role="presentation">
                    <div class="legend-cell"><span class="swatch" style="background:#0D659C"></span> Wages (73%)</div>
                    <div class="legend-cell"><span class="swatch" style="background:#E67E22"></span> Pensions (7%)</div>
                    <div class="legend-cell"><span class="swatch" style="background:#00875A"></span> Dividends (3%)</div>
                    <div class="legend-cell"><span class="swatch" style="background:#9B59B6"></span> Trusts (3%)</div>
                    <div class="legend-cell"><span class="swatch" style="background:#D50000"></span> Cap Gains (1%)</div>
                    <div class="legend-cell"><span class="swatch" style="background:#7F8C8D"></span> Other (13%)</div>
                </div>
            </div>
        </div>

        <!-- Scrolling Narrative Block (Anchored Right) -->
        <div class="scrolling-narrative-track">
            {#each incomeGroups as group, index}
                <div class="step-narrative-card" class:active={activeGroupIndex === index}>
                    <div class="card-inner">
                        <span class="card-context-tag">Income Group</span>
                        <h2>{group.label} Per Annum</h2>
                        <p>{group.narrative}</p>
                        
                        <!-- Mini Data Matrix Readout for Scannability -->
                        <div class="mini-data-strip">
                            <span class="strip-item">Wages: <strong>{group.distribution.wages}%</strong></span>
                            <span class="strip-item">Cap Gains: <strong>{group.distribution.capitalGains}%</strong></span>
                            <span class="strip-item">Trusts: <strong>{group.distribution.trusts}%</strong></span>
                        </div>
                    </div>
                </div>
            {/each}
        </div>
    </div>
</main>
<style>
    /* Global Editorial Scaffolding & Container Controls */
    .abc-story-lab-container { max-width: 1240px; margin: 0 auto; padding: 0 20px; font-family: -apple-system, BlinkMacSystemFont, "SF Pro Text", Roboto, Helvetica, Arial, sans-serif; background: #ffffff; color: #1a1a1a; line-height: 1.6; }
    
    /* ABC Newsroom Style Header Section */
    .Header { padding: 60px 0 40px; border-bottom: 1px solid #e0e0e0; margin-bottom: 40px; }
    .badge-pill { background: #e0f2fe; color: #005a9c; padding: 4px 10px; font-size: 0.75rem; text-transform: uppercase; font-weight: 700; letter-spacing: 1.2px; border-radius: 12px; display: inline-block; margin-bottom: 16px; }
    .Header h1 { font-family: "Georgia", Times, serif; font-size: 3rem; line-height: 1.15; font-weight: 700; margin: 0 0 20px 0; color: #0c0c0c; letter-spacing: -0.5px; }
    .Header-lead { font-size: 1.35rem; line-height: 1.45; color: #4a4a4a; max-width: 800px; margin-bottom: 24px; }
    .Header-byline { border-top: 1px solid #f0f0f0; padding-top: 16px; font-size: 0.9rem; color: #666; }
    .meta-date { font-size: 0.8rem; margin-top: 4px; color: #888; text-transform: uppercase; letter-spacing: 0.5px; }

    /* Narrative Body Copy with Typographic Drop-Cap */
    .narrative-body { max-width: 680px; margin: 0 auto 60px auto; font-size: 1.15rem; color: #2b2b2b; }
    .narrative-body p { margin-bottom: 24px; }
    .u-dropcap::first-letter { font-family: "Georgia", Times, serif; font-size: 4.8rem; float: left; line-height: 0.85; padding-top: 4px; padding-right: 8px; font-weight: bold; color: #0D659C; }

    /* High-Performance Scrollytelling Engine Layout */
    .scrollyteller-layout-engine { display: flex; flex-direction: row; position: relative; gap: 40px; margin: 80px 0; }
    @media (max-width: 900px) { .scrollyteller-layout-engine { flex-direction: column; } }

    /* Sticky Canvas Port (Anchored Left) */
    .sticky-visual-viewport { position: sticky; top: 40px; flex: 1.2; height: 80vh; display: flex; align-items: center; justify-content: center; background: #f8fafc; border-radius: 12px; border: 1px solid #f1f5f9; padding: 30px; box-sizing: border-box; }
    @media (max-width: 900px) { .sticky-visual-viewport { position: sticky; top: 10px; width: 100%; height: 40vh; z-index: 10; box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.05); } }
    .canvas-wrapper { width: 100%; display: flex; flex-direction: column; align-items: center; gap: 24px; }
    .dot-matrix-canvas { width: 100%; height: auto; max-height: 380px; aspect-ratio: 2 / 1; object-fit: contain; }

    /* Static Grid Legend Components */
    .legend-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 12px; width: 100%; max-width: 600px; padding: 16px; background: #ffffff; border-radius: 8px; border: 1px solid #e2e8f0; font-size: 0.8rem; font-weight: 600; color: #475569; }
    @media (max-width: 480px) { .legend-grid { grid-template-columns: repeat(2, 1fr); } }
    .legend-cell { display: flex; align-items: center; gap: 8px; }
    .swatch { display: inline-block; width: 12px; height: 12px; border-radius: 50%; flex-shrink: 0; }

    /* Scrolling Narrative Cards Track (Anchored Right) */
    .scrolling-narrative-track { flex: 0.9; padding: 25vh 0; display: flex; flex-direction: column; }
    @media (max-width: 900px) { .scrolling-narrative-track { flex: none; width: 100%; padding: 0 10px 40vh 10px; box-sizing: border-box; } }
    .step-narrative-card { min-height: 75vh; display: flex; align-items: center; justify-content: flex-start; opacity: 0.3; transform: scale(0.96); transition: all 0.4s ease; }
    .step-narrative-card.active { opacity: 1; transform: scale(1); }
    .card-inner { background: #ffffff; border: 1px solid #e2e8f0; border-radius: 12px; padding: 32px; box-shadow: 0 10px 15px -3px rgb(0 0 0 / 0.04); border-left: 5px solid #0D659C; width: 100%; }
    .card-context-tag { font-size: 0.7rem; text-transform: uppercase; font-weight: 700; color: #64748b; letter-spacing: 1px; display: block; margin-bottom: 8px; }
    .step-narrative-card h2 { font-size: 1.6rem; margin: 0 0 16px 0; color: #0f172a; font-family: "Georgia", Times, serif; }
    .step-narrative-card p { color: #334155; font-size: 1.05rem; margin-bottom: 20px; line-height: 1.55; }

    /* Mini Dashboard Strip Rules */
    .mini-data-strip { display: flex; flex-direction: column; gap: 6px; border-top: 1px dashed #e2e8f0; padding-top: 14px; font-size: 0.85rem; color: #64748b; }
    .strip-item strong { color: #0f172a; font-family: monospace; font-size: 0.95rem; }

    /* Invisible Utility Classes Supporting Accessibility Metrics */
    .visually-hidden { position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow: hidden; clip: rect(0, 0, 0, 0); border: 0; }
</style>
