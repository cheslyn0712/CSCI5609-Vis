<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = { movies: TMovie[]; width?: number; height?: number };
  let { movies, width = 700, height = 400 }: Props = $props();

  const margin = { top: 20, right: 20, bottom: 80, left: 40 };
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
    yearTop3Data.length > 25
      ? yearTop3Data.filter((_, i) => i % 3 === 0)
      : yearTop3Data
  );
  const maxTotal = $derived(
    Math.max(...yearTop3Data.map((d) => d.top3.reduce((s, t) => s + t.count, 0)), 1)
  );

  const allGenres = $derived.by(() => {
    const s = new Set<string>();
    yearTop3Data.forEach((d) => d.top3.forEach((t) => s.add(t.genre)));
    return Array.from(s);
  });
  const colorScale = $derived(d3.scaleOrdinal(d3.schemeCategory10).domain(allGenres));

  const xScale = $derived(
    d3
      .scaleBand()
      .domain(sampledYears.map((d) => d.year.toString()))
      .range([0, innerWidth])
      .padding(0.3)
  );
  const yScale = $derived(
    d3.scaleLinear().domain([0, maxTotal]).range([innerHeight, 0])
  );
</script>

<div class="viz-box">
  <h4>Second Figure: Stacked Bar Chart</h4>
  <p class="enc">Vertical axis = movie count. Horizontal axis = year. Stacked segments = top 3 genres by rank. Color = genre.</p>
  {#if movies.length > 0 && sampledYears.length > 0}
    <svg {width} {height}>
      <g transform="translate({margin.left}, {margin.top})">
        {#each sampledYears as { year, top3 }}
          {@const x0 = xScale(year.toString()) ?? 0}
          {@const bw = xScale.bandwidth()}
          {#each top3 as { genre, count }, i}
            {@const prevSum = top3.slice(0, i).reduce((s, t) => s + t.count, 0)}
            {@const y = yScale(prevSum + count)}
            {@const h = Math.max(2, yScale(prevSum) - yScale(prevSum + count))}
            <rect
              x={x0 + 2}
              y={y}
              width={bw - 4}
              height={h}
              fill={colorScale(genre)}
            >
              <title>{year}: {genre} ({count})</title>
            </rect>
          {/each}
        {/each}
        <g transform="translate(0, {innerHeight})" font-size="10">
          {#each sampledYears as { year }}
            <text
              x={(xScale(year.toString()) ?? 0) + xScale.bandwidth() / 2}
              y={15}
              text-anchor="middle"
              transform="rotate(-45, {(xScale(year.toString()) ?? 0) + xScale.bandwidth() / 2}, 15)"
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
