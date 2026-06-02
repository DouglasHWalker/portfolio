<script>
  import StoryInteractives from './StoryInteractives.svelte';
</script>

<header class="portfolio-global-header">
  <div class="header-content">
    <h1>Editorial Development Portfolio</h1>
    <p class="subtitle">Interactive Data Journalism & Visual Storytelling</p>
  </div>
</header>

<!-- This brings in your housing crisis and census interactive modules -->
<StoryInteractives />

<footer class="portfolio-global-footer">
  <div class="footer-content">
    <p>© 2026 • Built with Svelte and Native Web APIs</p>
  </div>
</footer>

<style>
  /* Quick baseline wrapper styles to frame out your projects */
  .portfolio-global-header {
    background-color: #06254B; /* Matches ABC Corporate Blue */
    color: #ffffff;
    padding: 40px 20px;
    text-align: center;
    margin-bottom: 40px;
  }
  .portfolio-global-header h1 {
    margin: 0;
    font-size: 2.2rem;
    font-weight: 700;
  }
  .subtitle {
    margin: 5px 0 0 0;
    font-size: 1.1rem;
    color: #cbd5e1;
  }
  .portfolio-global-footer {
    background-color: #f1f5f9;
    color: #475569;
    text-align: center;
    padding: 30px 20px;
    margin-top: 80px;
    border-top: 1px solid #e2e8f0;
    font-size: 0.9rem;
  }
  .footer-content p {
    margin: 0;
  }
</style>
