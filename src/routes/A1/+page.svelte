<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import { base } from "$app/paths";
  import type { TMovie } from "../../types";
  import Bar from "$lib/Bar.svelte";
  import Q1Viz1Matrix from "$lib/Q1Viz1Matrix.svelte";
  import Q1Viz2StackedBar from "$lib/Q1Viz2StackedBar.svelte";
  import Q1Viz3GroupedBar from "$lib/Q1Viz3GroupedBar.svelte";
  import Q2Viz1Heatmap from "$lib/Q2Viz1Heatmap.svelte";
  import Q2Viz2BarByGenre from "$lib/Q2Viz2BarByGenre.svelte";
  import Q2Viz3Chord from "$lib/Q2Viz3Chord.svelte";

  let movies: TMovie[] = [];

  async function loadCsv() {
    try {
      const csvUrl = `${base}/summer_movies.csv`;
      movies = await d3.csv(csvUrl, (row) => {
        return {
          tconst: row.tconst ?? "",
          title_type: row.title_type ?? "",
          primary_title: row.primary_title ?? "",
          original_title: row.original_title ?? "",
          year: row.year ? new Date(parseInt(row.year), 0, 1) : new Date(0),
          runtime_minutes: row.runtime_minutes ? parseInt(row.runtime_minutes) || 0 : 0,
          genres: row.genres ? row.genres.split(",").map((g) => g.trim()) : [],
          average_rating: row.average_rating ? parseFloat(row.average_rating) || 0 : 0,
          num_votes: row.num_votes ? parseInt(row.num_votes) || 0 : 0,
        } as TMovie;
      });
    } catch (error) {
      console.error("Error loading CSV:", error);
    }
  }
  onMount(loadCsv);
</script>

<h1>Summer Movies</h1>

<p>Here are {movies.length == 0 ? "..." : movies.length + " "} movies</p>

<h2>Genre Distribution</h2>
<Bar {movies} />

<h2>Q1: Annual Top 3 Genres Over Time</h2>
<div class="viz-grid">
  <Q1Viz1Matrix {movies} />
  <Q1Viz2StackedBar {movies} />
  <Q1Viz3GroupedBar {movies} />
</div>

<h2>Q2: Genre Co-occurrence</h2>
<div class="viz-grid">
  <Q2Viz1Heatmap {movies} />
  <Q2Viz2BarByGenre {movies} />
  <Q2Viz3Chord {movies} />
</div>

<style>
  .viz-grid { display: flex; flex-wrap: wrap; gap: 2rem; }
</style>
