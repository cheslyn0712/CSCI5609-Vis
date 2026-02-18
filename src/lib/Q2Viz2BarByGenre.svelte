<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = { movies: TMovie[]; width?: number; height?: number };
  let { movies, width = 500, height = 380 }: Props = $props();
  let selectedGenre = $state("Comedy");

  const margin = { top: 30, right: 20, bottom: 100, left: 50 };
  const innerW = $derived(width - margin.left - margin.right);
  const innerH = $derived(height - margin.top - margin.bottom);

  const { coOccurrence, allGenres } = $derived.by(() => {
    const co: Record<string, Record<string, number>> = {};
    movies.forEach((m) => {
      const gs = m.genres.filter((g) => g && g !== "NA");
      for (let i = 0; i < gs.length; i++) {
        for (let j = i; j < gs.length; j++) {
          const a = gs[i], b = gs[j];
          if (!co[a]) co[a] = {};
          if (!co[b]) co[b] = {};
          co[a][b] = (co[a][b] || 0) + 1;
          if (a !== b) co[b][a] = (co[b][a] || 0) + 1;
        }
      }
    });
    return { coOccurrence: co, allGenres: Object.keys(co).sort() };
  });

  const barData = $derived.by(() => {
    if (!selectedGenre || !coOccurrence[selectedGenre]) return [];
    return Object.entries(coOccurrence[selectedGenre])
      .filter(([g]) => g !== selectedGenre)
      .sort((a, b) => b[1] - a[1])
      .slice(0, 15);
  });
  const maxCnt = $derived(Math.max(...barData.map((d) => d[1]), 1));

  const xScale = $derived(
    d3.scaleBand().domain(barData.map((d) => d[0])).range([0, innerW]).padding(0.25)
  );
  const yScale = $derived(d3.scaleLinear().domain([0, maxCnt]).range([innerH, 0]));
</script>

<div class="viz-box">
  <h4>Second Figure: Co-occurrence with Selected Genre</h4>
  <p class="enc">Vertical axis = count. Horizontal axis = other genres that co-occur with selected genre. Select: <select bind:value={selectedGenre}>
    {#each allGenres as g}
      <option value={g}>{g}</option>
    {/each}
  </select></p>
  {#if movies.length > 0 && barData.length > 0}
    <svg {width} {height}>
      <g transform="translate({margin.left}, {margin.top})">
        {#each barData as [genre, cnt]}
          <rect
            x={xScale(genre) ?? 0}
            y={yScale(cnt)}
            width={xScale.bandwidth()}
            height={Math.max(2, innerH - yScale(cnt))}
            fill="#4a7c59"
          />
          <text
            x={(xScale(genre) ?? 0) + xScale.bandwidth() / 2}
            y={innerH + 14}
            text-anchor="end"
            font-size="9"
            transform="rotate(-40, {(xScale(genre) ?? 0) + xScale.bandwidth() / 2}, {innerH + 14})"
          >
            {genre}
          </text>
        {/each}
        <line x1={0} y1={0} x2={0} y2={innerH} stroke="#999" stroke-width="1" />
        <line x1={0} y1={innerH} x2={innerW} y2={innerH} stroke="#999" stroke-width="1" />
      </g>
    </svg>
  {/if}
</div>
<style>
  .viz-box { margin: 1rem 0; }
  .viz-box h4 { margin: 0 0 0.25rem 0; font-size: 1rem; }
  .enc { font-size: 0.75rem; color: #666; margin: 0 0 0.5rem 0; }
  select { font-size: 0.85rem; margin-left: 0.25rem; }
</style>
