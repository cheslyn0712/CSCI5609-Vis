<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = {
    movies: TMovie[];
    progress?: number;
    width?: number;
    height?: number;
  };
  let { movies, progress = 100, width = 500, height = 400 }: Props = $props();

  let selectedGenre: string = $state("");

  const yearRange = $derived(d3.extent(movies.map((d) => d.year)) as [Date, Date]);

  function getUpYear(yearRange: [undefined, undefined] | [Date, Date]) {
    if (!yearRange || !yearRange[0]) return new Date();
    const timeScale = d3.scaleTime().domain(yearRange).range([0, 100]);
    return timeScale.invert(progress);
  }
  const upYear: Date = $derived(getUpYear(yearRange ?? [undefined, undefined]));

  function getGenreNums(movies: TMovie[], upYear: Date) {
    let res: { [genre: string]: number } = {};
    movies
      .filter((movie) => movie.year <= upYear)
      .forEach((movie) => {
        movie.genres.filter((g) => g && g !== "NA").forEach((genre: string) => {
          res[genre] = (res[genre] || 0) + 1;
        });
      });
    return res;
  }

  const genreNums = $derived(getGenreNums(movies, upYear));

  const sortedGenres = $derived(
    Object.keys(genreNums).sort((a, b) => genreNums[b] - genreNums[a])
  );

  const margin = {
    top: 15,
    bottom: 50,
    left: 30,
    right: 10,
  };

  const usableArea = $derived({
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left,
  });

  const xScale = $derived(
    d3
      .scaleBand()
      .range([usableArea.left, usableArea.right])
      .domain(sortedGenres)
      .padding(0.1)
  );

  const yMax = $derived(Math.max(...Object.values(genreNums), 1));
  const yScale = $derived(
    d3
      .scaleLinear()
      .range([usableArea.bottom, usableArea.top])
      .domain([0, yMax])
  );

  const xBarwidth: number = $derived(xScale.bandwidth());

  let xAxis: SVGGElement | undefined = $state(),
    yAxis: SVGGElement | undefined = $state();

  function updateAxis() {
    if (xAxis) {
      d3.select(xAxis)
        .call(d3.axisBottom(xScale))
        .selectAll("text")
        .attr("transform", "rotate(45)")
        .style("text-anchor", "start");
    }
    if (yAxis) {
      d3.select(yAxis).call(d3.axisLeft(yScale));
    }
  }
  $effect(() => {
    updateAxis();
  });
</script>

<h3>
  The Distribution of Genres {yearRange?.[0]?.getFullYear()} - {yearRange?.[1]?.getFullYear()}
</h3>

{#if movies.length > 0 && sortedGenres.length > 0}
  <svg {width} {height}>
    <g class="bars">
      {#each sortedGenres as genre}
        {@const cnt = genreNums[genre]}
        {@const xPos = xScale(genre)}
        <g class={genre}>
          <rect
            role="img"
            tabindex="0"
            aria-label="Genre {genre}: {cnt} movies"
            width={xBarwidth}
            height={yScale(0) - yScale(cnt)}
            x={xPos ?? 0}
            y={yScale(cnt)}
            fill="#449900"
            class="bar"
            opacity={selectedGenre === "" || selectedGenre === genre ? 1 : 0.3}
            onmouseover={() => {
              selectedGenre = genre;
            }}
            onmouseout={() => {
              selectedGenre = "";
            }}
            onfocus={() => {
              selectedGenre = genre;
            }}
            onblur={() => {
              selectedGenre = "";
            }}
          />
          <text
            x={(xPos ?? 0) + xBarwidth / 2}
            y={yScale(cnt) - 5}
            font-size="12"
            text-anchor="middle"
          >
            {cnt}
          </text>
        </g>
      {/each}
    </g>

    <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
    <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />
  </svg>
{/if}

<style>
  .bar {
    transition:
      opacity 0.2s ease;
    cursor: pointer;
  }
  .bar:hover {
    opacity: 0.9 !important;
  }
</style>
