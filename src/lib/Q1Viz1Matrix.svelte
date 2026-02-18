<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = { movies: TMovie[]; width?: number; height?: number };
  let { movies, width = 700, height = 550 }: Props = $props();

  const margin = { top: 20, right: 180, bottom: 30, left: 50 };
  const innerWidth = $derived(width - margin.left - margin.right);
  const innerHeight = $derived(height - margin.top - margin.bottom);

  const yearTop3Data = $derived.by(() => {
    const yearGenreCounts: Record<number, Record<string, number>> = {};
    movies.forEach((m) => {
      const y = m.year.getFullYear();
      const validGenres = m.genres.filter((g) => g && g !== "NA");
      if (!validGenres.length) return;
      if (!yearGenreCounts[y]) yearGenreCounts[y] = {};
      validGenres.forEach((g) => {
        yearGenreCounts[y][g] = (yearGenreCounts[y][g] || 0) + 1;
      });
    });
    return Object.entries(yearGenreCounts)
      .map(([yearStr, counts]) => {
        const sorted = Object.entries(counts)
          .sort((a, b) => b[1] - a[1])
          .slice(0, 3);
        return { year: parseInt(yearStr), top3: sorted.map(([g, c]) => ({ genre: g, count: c })) };
      })
      .sort((a, b) => a.year - b.year);
  });

  const sampledYears = $derived(
    yearTop3Data.length > 30
      ? yearTop3Data.filter((_, i) => i % 2 === 0)
      : yearTop3Data
  );
  const rowHeight = $derived(Math.max(10, innerHeight / Math.max(sampledYears.length, 1)));
  const blockWidth = $derived(innerWidth / 3);

  const allGenres = $derived.by(() => {
    const s = new Set<string>();
    yearTop3Data.forEach((d) => d.top3.forEach((t) => s.add(t.genre)));
    return Array.from(s);
  });
  const colorScale = $derived(d3.scaleOrdinal(d3.schemeCategory10).domain(allGenres));
</script>

<div class="viz-box">
  <h4>First Figure: Matrix (Year × Rank)</h4>
  <p class="enc">Vertical axis = year. Horizontal axis = rank (#1, #2, #3). Color = genre.</p>
  {#if movies.length > 0 && sampledYears.length > 0}
    <svg {width} {height}>
      <g transform="translate({margin.left}, {margin.top})">
        {#each sampledYears as { year, top3 }, rowIndex}
          {#each top3 as { genre, count }, colIndex}
            <rect
              x={colIndex * blockWidth + 1}
              y={rowIndex * rowHeight + 1}
              width={blockWidth - 2}
              height={rowHeight - 2}
              fill={colorScale(genre)}
              style="cursor:pointer"
            >
              <title>Year {year} #{colIndex + 1}: {genre} ({count})</title>
            </rect>
          {/each}
          <text
            x={-5}
            y={rowIndex * rowHeight + rowHeight / 2}
            text-anchor="end"
            dominant-baseline="middle"
            font-size="10"
          >
            {year}
          </text>
        {/each}
        <text x={blockWidth / 2} y={-5} text-anchor="middle" font-size="9">#1</text>
        <text x={blockWidth * 1.5} y={-5} text-anchor="middle" font-size="9">#2</text>
        <text x={blockWidth * 2.5} y={-5} text-anchor="middle" font-size="9">#3</text>
      </g>
    </svg>
    <div class="legend">
      {#each allGenres.slice(0, 12) as g}
        <span><span class="swatch" style="background:{colorScale(g)}"></span>{g}</span>
      {/each}
    </div>
  {/if}
</div>
<style>
  .viz-box { margin: 1rem 0; }
  .viz-box h4 { margin: 0 0 0.25rem 0; font-size: 1rem; }
  .enc { font-size: 0.75rem; color: #666; margin: 0 0 0.5rem 0; }
  .legend { display: flex; flex-wrap: wrap; gap: 0.5rem; font-size: 10px; margin-top: 0.5rem; }
  .swatch { display: inline-block; width: 8px; height: 8px; margin-right: 2px; vertical-align: middle; }
</style>
