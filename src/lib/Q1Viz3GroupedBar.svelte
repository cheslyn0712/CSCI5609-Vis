<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = { movies: TMovie[]; width?: number; height?: number };
  let { movies, width = 700, height = 400 }: Props = $props();

  const margin = { top: 20, right: 20, bottom: 90, left: 45 };
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
    yearTop3Data.length > 24
      ? yearTop3Data.filter((_, i) => i % 3 === 0)
      : yearTop3Data
  );
  const maxCount = $derived(
    Math.max(...yearTop3Data.flatMap((d) => d.top3.map((t) => t.count)), 1)
  );

  const allGenres = $derived.by(() => {
    const s = new Set<string>();
    yearTop3Data.forEach((d) => d.top3.forEach((t) => s.add(t.genre)));
    return Array.from(s);
  });
  const colorScale = $derived(d3.scaleOrdinal(d3.schemeCategory10).domain(allGenres));

  const x0 = $derived(
    d3
      .scaleBand()
      .domain(sampledYears.map((d) => d.year.toString()))
      .range([0, innerWidth])
      .padding(0.2)
  );
  const x1 = $derived(d3.scaleBand().domain(["1", "2", "3"]).range([0, x0.bandwidth()]).padding(0.05));
  const yScale = $derived(d3.scaleLinear().domain([0, maxCount]).range([innerHeight, 0]));
</script>

<div class="viz-box">
  <h4>Third Figure: Grouped Bar Chart</h4>
  <p class="enc">Vertical axis = movie count. Horizontal axis = year. Each year has 3 grouped bars (rank 1, 2, 3). Color = genre.</p>
  {#if movies.length > 0 && sampledYears.length > 0}
    <svg {width} {height}>
      <g transform="translate({margin.left}, {margin.top})">
        {#each sampledYears as { year, top3 }}
          {@const gx = x0(year.toString()) ?? 0}
          {#each top3 as { genre, count }, rankIdx}
            <rect
              x={gx + (x1((rankIdx + 1).toString()) ?? 0)}
              y={yScale(count)}
              width={x1.bandwidth()}
              height={Math.max(1, innerHeight - yScale(count))}
              fill={colorScale(genre)}
            >
              <title>{year} #{rankIdx + 1}: {genre} ({count})</title>
            </rect>
          {/each}
        {/each}
        <g transform="translate(0, {innerHeight})" font-size="9">
          {#each sampledYears as { year }}
            <text
              x={(x0(year.toString()) ?? 0) + x0.bandwidth() / 2}
              y={12}
              text-anchor="end"
              transform="rotate(-50, {(x0(year.toString()) ?? 0) + x0.bandwidth() / 2}, 12)"
            >
              {year}
            </text>
          {/each}
        </g>
        <line x1={0} y1={0} x2={0} y2={innerHeight} stroke="#ccc" stroke-width="1" />
        <line x1={0} y1={innerHeight} x2={innerWidth} y2={innerHeight} stroke="#ccc" stroke-width="1" />
      </g>
    </svg>
    <div class="legend">
      {#each allGenres.slice(0, 10) as g}
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
