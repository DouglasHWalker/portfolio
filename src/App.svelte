<script>
  import StoryInteractives from "./StoryInteractives.svelte";
</script>

<!-- Editorial Header Section -->
<header class="editorial-header">
  <span class="tag">Side gigs are booming</span>
  <h1>Most Businesses Don't Employ Anyone.</h1>
  <p class="lead-text">
    The vast majority of those non-employing entities go bust or dissolve within
    three years.
  </p>
</header>

<!-- This brings in your housing crisis and census interactive modules -->
<StoryInteractives />

<!-- Append this block directly below your </header> tag inside the .article-container -->

<section class="editorial-narrative">
  <div class="narrative-col">
    <p>
      Small business may no longer be "the engine room of the Australian
      economy". The vast majority of new business registrations today are
      non-employing . In what economists call, "lifestyle businesses",
      individuals are running legal vehicles for independent contracting,
      freelance gigs, and solo side hustles; Anything to make a ends meet right.
      A second job is not new, but the scale of entrants is. And unfortunatley
      because these one-person operations lack a team or financial backup, they
      are incredibly fragile. Nearly 40% of them collapse or close their doors
      within their first few years.
    </p>
    <p>
      True economic dynamism, job creation, and productivity growth are driven
      almost entirely by a vanishingly small subset of young companies known as
      "Gazelles". These are high-growth firms that enter the market small but
      scale exponentially within their first five years. A Marshallian
      agglomeration economy dictates that firms in the same industry do not
      distribute themselves randomly across a landscape. Instead, they sort into
      specialized geographic clusters to exploit three distinct advantages: a
      localized pool of specialized labor, the rapid diffusion of
      industry-specific knowledge, and proximity to specialized suppliers.
    </p>
    <p>
      This explains why a booming digital or hospitality sector doesn't lift all
      suburbs equally. Instead, growth feeds on growth. Suburbs that already
      possess strong infrastructural foundations or existing industry density
      attract further entrants, driving up property values and creating economic
      hotspots. Conversely, outer regions may show high raw counts of total
      entities, often driven by residential home-based solo operators, but
      remain entirely stagnant in the long-term.
    </p>
    <p>
      This spatial sorting is further accelerated by a structural shift in how
      modern enterprises operate. Driven by the suburbanization of knowledge
      workers and prohibitive commercial rents, businesses are increasingly
      migrating away from major metropolitan central business districts. Enabled
      by remote work, artificial intelligence, cloud infrastructure, and
      automation, these hyper-efficient solo ventures are highly lucrative for
      the individual, but they break the historical link between commercial
      growth and macroeconomic expansion; they generate substantial wealth
      without generating a matching footprint of physical employment or capital
      infrastructure in the surrounding community. While AI-enabled side hustles
      and gig work can generate individual wealth, societies are not measurably
      better off as the geographic advantage of a big city slowly dies.
    </p>
    <p></p>
  </div>
</section>

<footer class="portfolio-global-footer">
  <div class="footer-content">
    <p>
      ©Atlas - Douglas Walker - 2026 • Built with Svelte, D3, and MapLibre -
      Data sourced from ABS
    </p>
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

  .editorial-narrative {
    display: grid;
    grid-template-columns: 1fr;
    gap: 3rem;
    text-align: justify;
    margin-top: 2 rem;
    margin-bottom: 4rem;
    border-bottom: 1px solid #334155;
    padding-bottom: 3rem;
    margin-left: 10rem;
    margin-right: 10rem;
  }

  .narrative-col h3 {
    font-size: 1.3rem;
    font-weight: 700;
    color: #cbd5e1;
    margin: 0 0 1rem 0;
  }

  .narrative-col p {
    font-size: 1rem;
    line-height: 1.7;
    color: #94a3b8;
    margin-bottom: 1.25rem;
  }

  .narrative-col strong {
    color: #fe662f;
  }

  /* Responsive scaling wrapper override for standard viewport frames */
  @media (max-width: 1024px) {
    .editorial-narrative {
      grid-template-columns: 1fr;
      gap: 2rem;
    }
  }
</style>
