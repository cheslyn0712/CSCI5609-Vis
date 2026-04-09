<script lang="ts">
  import { Scroll } from "$lib";
  import { fly, slide } from "svelte/transition";

  type Props = {
    movieNum: number;
    startYear: number;
    endYear: number;
  };

  let { movieNum, startYear, endYear }: Props = $props();
  let progress = $state(0);
</script>

<Scroll bind:progress --scrolly-story-width="0" --scrolly-viz-width="1fr">
  <div id="virtual"></div>
  <section slot="viz" class="hero">
    <h1>A4 Scrollytelling</h1>
    {#if progress > 20}
      <p in:slide={{ duration: 650, axis: "x" }}>
        Summer movies from {startYear} to {endYear}
      </p>
    {/if}
    {#if progress > 55}
      <p in:fly={{ duration: 600, y: 80 }}>
        Explore changes of genre distribution with 2D and 3D narratives.
      </p>
    {/if}
    {#if progress > 82}
      <p in:fly={{ duration: 550, y: 50 }}>
        Dataset size: {movieNum} movies
      </p>
    {/if}
  </section>
</Scroll>

<style>
  #virtual {
    height: 130vh;
  }

  .hero {
    min-height: 58vh;
    background: linear-gradient(145deg, #fff2e1, #f9dfc7);
    border-radius: 14px;
    padding: 2.6rem 2.2rem;
    box-shadow: 0 8px 28px rgba(51, 39, 15, 0.13);
    border: 1px solid #f0cfa6;
  }

  .hero h1 {
    margin: 0 0 1rem;
    font-size: clamp(2rem, 5vw, 3.3rem);
    color: #483310;
  }

  .hero p {
    margin: 0.8rem 0;
    font-size: clamp(1rem, 2vw, 1.35rem);
    color: #5c4a2b;
    max-width: 42rem;
  }
</style>
