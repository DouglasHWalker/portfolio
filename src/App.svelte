<script>
  import StoryInteractives from './StoryInteractives.svelte';
</script>

<!-- Editorial Header Section -->
    <header class="editorial-header">
        <span class="tag">Side Hustles are booming</span>
        <h1>Why having two or more jobs could soon be the norm</h1>
        <!-- <p class="lead-text">
            What's happening your street.
        </p> -->
    </header>

<!-- This brings in your housing crisis and census interactive modules -->
<StoryInteractives />

      <p style="text-align: justify; margin: 0px 50px 20px 50px">
            The defining structural transformation of the post-pandemic era is the rise of the micro-business. Non-employing sole traders now make up the absolute bedrock of the domestic economy. 
            Digital professional services, delivery logistics, and independent consulting have decentralized traditional office hubs.
            While solo operators are highly agile, historical tracking shows that they face brutal lifespans, with less than half surviving past their third year of initial trading.
            Instead of crowding into central business districts, these solo operators dominate outer metropolitan rings, turning bedroom suburbs into active economic engines.
        </p>

<footer class="portfolio-global-footer">
  <div class="footer-content">
    <p>©Atlas - Douglas Walker - 2026 • Built with Svelte, D3, and MapLibre - Data sourced from ABS </p>
  </div>
</footer>

<style>

    .editorial-header {
        margin-top: 2rem;
        border-left: 4px solid #fe662f;
        padding-left: 1.5rem;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-content: center;
        width: 100%;
    }
        .tag {
        text-transform: uppercase;
        font-size: 0.75rem;
        letter-spacing: 0.1em;
        color: #fe662f;
        font-weight: 700;
    }

    h1 {
        font-size: 2.5rem;
        margin: 0.5rem 0;
        font-weight: 800;
    }

    .lead-text {
        color: #94a3b8;
        font-size: 1.1rem;
        max-width: 800px;
        line-height: 1.6;
        margin: 0 auto;
    }

  .portfolio-global-footer {
     font-size: 0.65rem;
        color: #64748b;
        margin-top: 0.5rem;
        border-top: 1px solid #334155;
        padding-top: 0.25rem;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
  }

  
</style>
