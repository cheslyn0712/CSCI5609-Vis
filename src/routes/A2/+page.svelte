<script lang="ts">
  import * as d3 from "d3";
  import { Scatter, Line } from "$lib";
  import { onMount } from "svelte";
  import { base } from "$app/paths";
  import type { TMovie } from "../../types";

  let movies: TMovie[] = $state([]);
  let yearRange: [Date, Date] | undefined = $state();

  function getYearCountArray(movies: TMovie[]) {
    let yearCount: { [year: number]: number } = {};
    const allYears = [...new Set(movies.map((d) => d.year.getFullYear()))];
    for (let year of allYears) {
      yearCount[year] = movies.filter(
        (d) => d.year.getFullYear() == year,
      ).length;
    }
    const yearCountArray = Object.entries(yearCount).map(
      ([year, count]) => ({
        x: new Date(parseInt(year)),
        y: count as number,
      }),
    );
    yearCountArray.sort((a, b) => (a.x < b.x ? -1 : 1));
    return yearCountArray;
  }

  let yearCountArray = $derived(getYearCountArray(movies));

  type TAxisSelection = {
    x: keyof TMovie;
    y: keyof TMovie;
    size: keyof TMovie;
  };

  let axisSelection: TAxisSelection = $state({
    x: "year",
    y: "average_rating",
    size: "num_votes",
  });

  async function loadCsv() {
    try {
      const csvUrl = `${base}/summer_movies.csv`;
      const rows = await d3.csv(csvUrl, (row) => {
        return {
          tconst: row.tconst ?? "",
          title_type: row.title_type ?? "",
          primary_title: row.primary_title ?? "",
          original_title: row.original_title ?? "",
          year: row.year ? new Date(parseInt(row.year), 0, 1) : new Date(0),
          runtime_minutes: row.runtime_minutes ? parseInt(row.runtime_minutes) || 0 : 0,
          genres: row.genres ? row.genres.split(",").map((g) => g.trim()).filter((g) => g && g !== "NA") : [],
          average_rating: row.average_rating ? parseFloat(row.average_rating) || 0 : 0,
          num_votes: row.num_votes ? parseInt(row.num_votes) || 0 : 0,
        } as TMovie;
      });
      movies = rows;
    } catch (error) {
      console.error("Error loading CSV:", error);
    }
  }
  onMount(loadCsv);

  const attrOptionsX = $derived(movies[0] ? Object.keys(movies[0]) : []);
  const attrOptionsY = $derived(
    movies[0] ? Object.keys(movies[0]).filter((d) => d != "genres") : [],
  );
  const attrOptionsS = $derived(
    movies[0] ? Object.keys(movies[0]).filter((d) => d != "genres") : [],
  );
</script>

<div class="container">
  <h1>Summer Movies</h1>

  <p>Here are {movies.length == 0 ? "..." : movies.length + " "} movies</p>
  {#if movies.length > 0}
    <div class="selectors">
      <label>X Axis:
        <select bind:value={axisSelection.x}>
          {#each attrOptionsX as key}
            <option value={key}>{key}</option>
          {/each}
        </select>
      </label>
      <label>Y Axis:
        <select bind:value={axisSelection.y}>
          {#each attrOptionsY as key}
            <option value={key}>{key}</option>
          {/each}
        </select>
      </label>
      <label>Size:
        <select bind:value={axisSelection.size}>
          {#each attrOptionsS as key}
            <option value={key}>{key}</option>
          {/each}
        </select>
      </label>
    </div>

    <Scatter
      movies={yearRange
        ? movies.filter(
              (d) => d.year <= yearRange![1] && d.year >= yearRange![0],
            )
        : movies}
      x={axisSelection.x}
      y={axisSelection.y}
      size={axisSelection.size}
    />
    <br />

    <Line data={yearCountArray} bind:yearRange />
  {/if}
</div>

<style>
  .container {
    width: 60vw;
    margin: 10px auto;
    padding: 10px;
  }
  .selectors {
    display: flex;
    gap: 1.5rem;
    margin-bottom: 1rem;
    flex-wrap: wrap;
  }
  .selectors label {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
</style>
