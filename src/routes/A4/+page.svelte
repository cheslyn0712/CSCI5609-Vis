<script lang="ts">
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import { base } from "$app/paths";
  import type { TMovie } from "../../types";
  import StoryOpen from "./StoryOpen.svelte";
  import Scrolly2D from "./Scrolly2D.svelte";
  import Scrolly3D from "./Scrolly3D.svelte";

  let movies: TMovie[] = $state([]);
  let loadError = $state("");

  const yearExtent = $derived(
    d3.extent(movies, (d) => d.year.getFullYear()) as [number | undefined, number | undefined],
  );

  async function loadCsv() {
    try {
      const csvUrl = `${base}/summer_movies.csv`;
      const rows = await d3.csv(csvUrl, (row) => {
        return {
          tconst: row.tconst ?? "",
          title_type: row.title_type ?? "",
          primary_title: row.primary_title ?? "",
          original_title: row.original_title ?? "",
          year: row.year ? new Date(parseInt(row.year, 10), 0, 1) : new Date(0),
          runtime_minutes: row.runtime_minutes ? parseInt(row.runtime_minutes, 10) || 0 : 0,
          genres: row.genres
            ? row.genres
                .split(",")
                .map((g) => g.trim())
                .filter((g) => g && g !== "NA")
            : [],
          average_rating: row.average_rating ? parseFloat(row.average_rating) || 0 : 0,
          num_votes: row.num_votes ? parseInt(row.num_votes, 10) || 0 : 0,
        } as TMovie;
      });
      movies = rows;
    } catch (err) {
      console.error(err);
      loadError = "Failed to load summer_movies.csv";
    }
  }

  onMount(loadCsv);
</script>

<main class="a4">
  <StoryOpen
    movieNum={movies.length}
    startYear={yearExtent[0] ?? 0}
    endYear={yearExtent[1] ?? 0}
  />

  {#if loadError}
    <p class="error">{loadError}</p>
  {:else if movies.length > 0}
    <Scrolly2D {movies} />
    <Scrolly3D {movies} />
  {/if}
</main>

<style>
  .a4 {
    width: min(1200px, 92vw);
    margin: 0 auto;
    padding-bottom: 6rem;
  }

  .error {
    margin: 1rem 0;
    color: #9f1f1f;
    font-weight: 600;
  }
</style>
