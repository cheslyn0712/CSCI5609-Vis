<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = { movies: TMovie[]; width?: number; height?: number };
  let { movies, width = 500, height = 400 }: Props = $props();

  const margin = { top: 20, right: 20, bottom: 80, left: 50 };
  const innerW = $derived(width - margin.left - margin.right);
  const innerH = $derived(height - margin.top - margin.bottom);

  const pairData = $derived.by(() => {
    const co: Record<string, Record<string, number>> = {};
    movies.forEach((m) => {
      const gs = m.genres.filter((g) => g && g !== "NA");
      for (let i = 0; i < gs.length; i++) {
        for (let j = i + 1; j < gs.length; j++) {
          const a = gs[i], b = gs[j];
          const key = a < b ? `${a}|${b}` : `${b}|${a}`;
          if (!co[a]) co[a] = {};
          co[a][b] = (co[a][b] || 0) + 1;
        }
      }
    });
    const pairs: { label: string; value: number }[] = [];
    Object.entries(co).forEach(([a, rest]) => {
      Object.entries(rest).forEach(([b, v]) => {
        pairs.push({ label: `${a} + ${b}`, value: v });
      });
    });
    return pairs.sort((x, y) => y.value - x.value).slice(0, 14);
  });
  const maxV = $derived(Math.max(...pairData.map((d) => d.value), 1));

  const xScale = $derived(
    d3.scaleBand().domain(pairData.map((d) => d.label)).range([0, innerW]).padding(0.3)
  );
  const yScale = $derived(d3.scaleLinear().domain([0, maxV]).range([innerH, 0]));
</script>

<div class="viz-box">
  <h4>Third Figure: Top Co-occurrence Pairs</h4>
  <p class="enc">Vertical axis = count. Horizontal axis = genre pair (e.g., Drama + Romance).</p>
  {#if movies.length > 0 && pairData.length > 0}
    <svg {width} {height}>
      <g transform="translate({margin.left}, {margin.top})">
        {#each pairData as { label, value }}
          <rect
            x={xScale(label) ?? 0}
            y={yScale(value)}
            width={xScale.bandwidth()}
            height={Math.max(2, innerH - yScale(value))}
            fill="#6b8e7a"
          >
            <title>{label}: {value} movies</title>
          </rect>
          <text
            x={(xScale(label) ?? 0) + xScale.bandwidth() / 2}
            y={innerH + 12}
            text-anchor="end"
            font-size="8"
            transform="rotate(-50, {(xScale(label) ?? 0) + xScale.bandwidth() / 2}, {innerH + 12})"
          >
            {label}
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
</style>
