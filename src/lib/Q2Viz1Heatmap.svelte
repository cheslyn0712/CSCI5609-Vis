<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  type Props = { movies: TMovie[]; width?: number; height?: number };
  let { movies, width = 520, height = 520 }: Props = $props();

  const margin = { top: 8, right: 90, bottom: 8, left: 90 };
  const cellSize = 18;

  const { matrix, genres } = $derived.by(() => {
    const coOccurrence: Record<string, Record<string, number>> = {};
    movies.forEach((m) => {
      const gs = m.genres.filter((g) => g && g !== "NA");
      for (let i = 0; i < gs.length; i++) {
        for (let j = i; j < gs.length; j++) {
          const a = gs[i], b = gs[j];
          if (!coOccurrence[a]) coOccurrence[a] = {};
          if (!coOccurrence[b]) coOccurrence[b] = {};
          coOccurrence[a][b] = (coOccurrence[a][b] || 0) + 1;
          if (a !== b) coOccurrence[b][a] = (coOccurrence[b][a] || 0) + 1;
        }
      }
    });
    const allGenres = Object.keys(coOccurrence).sort();
    const matrixData = allGenres.map((g1) =>
      allGenres.map((g2) => coOccurrence[g1]?.[g2] ?? 0)
    );
    return { matrix: matrixData, genres: allGenres };
  });

  const size = $derived(genres.length * cellSize);
  const maxVal = $derived(matrix.length > 0 ? Math.max(...matrix.flat()) : 1);
  const colorScale = $derived(d3.scaleSequential(d3.interpolateYlOrRd).domain([0, maxVal]));
  let hovered = $state<{ i: number; j: number } | null>(null);
</script>

<div class="viz-box">
  <h4>First Figure: Co-occurrence Heatmap</h4>
  <p class="enc">Vertical axis = genre (row). Horizontal axis = genre (column). Each cell = number of movies with both genres. Color intensity = co-occurrence count.</p>
  {#if movies.length > 0 && genres.length > 0}
    <div class="heatmap-wrap">
      <svg width={size + margin.left + margin.right} height={size + margin.top + margin.bottom}>
        <g transform="translate({margin.left}, {margin.top})">
          {#each genres as g1, i}
            {#each genres as g2, j}
              {@const val = matrix[i]?.[j] ?? 0}
              <rect
                x={j * cellSize}
                y={i * cellSize}
                width={cellSize - 1}
                height={cellSize - 1}
                fill={colorScale(val)}
                stroke={hovered?.i === i && hovered?.j === j ? "#000" : "none"}
                stroke-width={2}
                style="cursor:pointer"
                onmouseover={() => hovered = { i, j }}
                onmouseout={() => hovered = null}
              >
                <title>{g1} &amp; {g2}: {val}</title>
              </rect>
            {/each}
          {/each}
          {#each genres as g, i}
            <text
              x={-4}
              y={i * cellSize + cellSize / 2}
              text-anchor="end"
              dominant-baseline="middle"
              font-size="9"
            >
              {g}
            </text>
          {/each}
          {#each genres as g, j}
            <text
              x={size + 4}
              y={j * cellSize + cellSize / 2}
              text-anchor="start"
              dominant-baseline="middle"
              font-size="9"
            >
              {g}
            </text>
          {/each}
        </g>
      </svg>
    </div>
    {#if hovered}
      <p class="tooltip">{genres[hovered.i]} &amp; {genres[hovered.j]}: {matrix[hovered.i]?.[hovered.j]} movies</p>
    {/if}
    <div class="legend">
      <span>0</span>
      <div class="grad" style="background:linear-gradient(to right,{colorScale(0)},{colorScale(maxVal)})"></div>
      <span>{maxVal}</span>
    </div>
  {/if}
</div>
<style>
  .viz-box { margin: 1rem 0; }
  .viz-box h4 { margin: 0 0 0.25rem 0; font-size: 1rem; }
  .enc { font-size: 0.75rem; color: #666; margin: 0 0 0.5rem 0; }
  .heatmap-wrap { overflow-x: auto; overflow-y: auto; max-height: 420px; }
  .tooltip { font-size: 12px; margin: 0.25rem 0; }
  .legend { display: flex; align-items: center; gap: 0.5rem; font-size: 10px; }
  .grad { width: 60px; height: 8px; border-radius: 2px; }
</style>
