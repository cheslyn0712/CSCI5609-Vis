<script lang="ts">
  import * as d3 from "d3";
  import { tweened } from "svelte/motion";
  import { Scroll } from "$lib";
  import type { TMovie } from "../../types";

  type Props = {
    movies: TMovie[];
  };
  let { movies }: Props = $props();

  type GenreYearRow = { year: number; total: number } & Record<string, number>;

  const chartWidth = 820;
  const chartHeight = 500;
  const margin = { top: 24, right: 20, bottom: 56, left: 56 };
  const innerWidth = chartWidth - margin.left - margin.right;
  const innerHeight = chartHeight - margin.top - margin.bottom;

  let progress = $state(0);
  const smoothProgress = tweened(0, { duration: 450 });

  $effect(() => {
    smoothProgress.set(progress);
  });

  function buildRows(moviesIn: TMovie[]): { rows: GenreYearRow[]; keys: string[] } {
    const byYear = new Map<number, Map<string, number>>();
    const genreTotals = new Map<string, number>();

    for (const movie of moviesIn) {
      const year = movie.year.getFullYear();
      if (!Number.isFinite(year) || year < 1900) continue;

      if (!byYear.has(year)) byYear.set(year, new Map<string, number>());
      const yearMap = byYear.get(year)!;

      for (const genre of movie.genres) {
        const g = genre.trim();
        if (!g || g === "NA") continue;
        yearMap.set(g, (yearMap.get(g) ?? 0) + 1);
        genreTotals.set(g, (genreTotals.get(g) ?? 0) + 1);
      }
    }

    const topGenres = [...genreTotals.entries()]
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5)
      .map(([k]) => k);

    const keys = [...topGenres, "Other"];
    const years = [...byYear.keys()].sort((a, b) => a - b);
    const rows: GenreYearRow[] = years.map((year) => {
      const yearMap = byYear.get(year)!;
      let total = 0;
      for (const value of yearMap.values()) total += value;

      const row: GenreYearRow = { year, total };
      let topSum = 0;
      for (const key of topGenres) {
        const count = yearMap.get(key) ?? 0;
        row[key] = count;
        topSum += count;
      }
      row.Other = Math.max(0, total - topSum);
      return row;
    });

    return { rows, keys };
  }

  const built = $derived(buildRows(movies));
  const rows = $derived(built.rows);
  const genreKeys = $derived(built.keys);
  const years = $derived(rows.map((d) => d.year));

  const xBand = $derived(
    d3
      .scaleBand<number>()
      .domain(years)
      .range([0, innerWidth])
      .paddingInner(0.08)
      .paddingOuter(0.02),
  );

  const yScale = $derived(d3.scaleLinear().domain([0, 1]).range([innerHeight, 0]).nice());

  const normalizedRows = $derived(
    rows.map((row) => {
      const total = Math.max(1, row.total);
      const result: GenreYearRow = { year: row.year, total: 1 };
      for (const key of genreKeys) result[key] = (row[key] ?? 0) / total;
      return result;
    }),
  );

  const stackSeries = $derived(
    d3.stack<GenreYearRow>().keys(genreKeys).offset(d3.stackOffsetNone)(normalizedRows),
  );

  const colorScale = $derived(
    d3
      .scaleOrdinal<string, string>()
      .domain(genreKeys)
      .range(["#4e79a7", "#f28e2c", "#e15759", "#76b7b2", "#59a14f", "#bab0ab"]),
  );

  const revealIndexFloat = $derived((($smoothProgress / 100) * Math.max(0, years.length - 1)));
  const currentIndex = $derived(Math.round(revealIndexFloat));
  const currentYear = $derived(years[currentIndex] ?? 0);
  const compareYearTarget = $derived(Math.max(0, currentYear - 10));
  const compareYear = $derived(
    years.length === 0
      ? 0
      : years.reduce((best, y) =>
          Math.abs(y - compareYearTarget) < Math.abs(best - compareYearTarget) ? y : best,
        years[0]),
  );
  const revealWidth = $derived(
    (() => {
    if (years.length === 0) return 0;
    const left = xBand(years[0]) ?? 0;
    const rightYear = years[Math.min(years.length - 1, Math.ceil(revealIndexFloat))];
    const right = (xBand(rightYear) ?? 0) + xBand.bandwidth();
    return Math.max(0, right - left);
    })(),
  );

  const currentRow = $derived(
    rows.find((r) => r.year === currentYear) ?? rows.at(-1),
  );
  const compareRow = $derived(
    rows.find((r) => r.year === compareYear) ?? rows[0],
  );

  const currentNormRow = $derived(
    normalizedRows.find((r) => r.year === currentYear) ?? normalizedRows.at(-1),
  );
  const compareNormRow = $derived(
    normalizedRows.find((r) => r.year === compareYear) ?? normalizedRows[0],
  );

  const currentTop = $derived(
    (() => {
    if (!currentRow || !currentNormRow) return [];
    return genreKeys
      .filter((k) => k !== "Other")
      .map((k) => ({
        genre: k,
        count: currentRow[k] ?? 0,
        share: (currentNormRow[k] ?? 0) * 100,
      }))
      .sort((a, b) => b.count - a.count)
      .slice(0, 3);
    })(),
  );

  const compareTop = $derived(
    (() => {
    if (!compareRow || !compareNormRow) return [];
    return genreKeys
      .filter((k) => k !== "Other")
      .map((k) => ({
        genre: k,
        count: compareRow[k] ?? 0,
        share: (compareNormRow[k] ?? 0) * 100,
      }))
      .sort((a, b) => b.count - a.count)
      .slice(0, 3);
    })(),
  );

  const compareSummary = $derived(
    currentTop.map((item) => {
      const prev = (compareNormRow?.[item.genre] ?? 0) * 100;
      const deltaPctPoint = item.share - prev;
      return { ...item, deltaPctPoint };
    }),
  );

  const tickYears = $derived(
    years.length <= 14
      ? years
      : years.filter((y, i) => i % Math.ceil(years.length / 10) === 0 || i === years.length - 1),
  );

  const yearCenter = (year: number) => (xBand(year) ?? 0) + xBand.bandwidth() / 2;
</script>

<h2>2D Scrolly: Genre Distribution Over Time</h2>
<p class="intro">
  Stacked bars keep every year visible. Scrolling moves a focus year and a 10-year-ago comparison anchor.
</p>

<Scroll bind:progress --scrolly-story-width="0.95fr" --scrolly-viz-width="1.25fr">
  <section class="story-step">
    <h3>Start with all years on screen</h3>
    <p>
      This keeps context and supports direct comparisons across distant years, following the "eyes beat memory" rule.
    </p>
  </section>
  <section class="story-step">
    <h3>Track a focus year while scrolling</h3>
    <p>
      The solid guide marks the focused year, while the dashed guide marks about 10 years earlier.
    </p>
  </section>
  <section class="story-step">
    <h3>Read decade-level change quickly</h3>
    <p>
      The panel reports top genres and +/- change vs 10 years ago as the story advances.
    </p>
  </section>

  <section slot="viz" class="viz">
    <div class="meta">
      <p><b>Focused year:</b> {currentYear || "-"} | <b>Compare year:</b> {compareYear || "-"}</p>
      <div class="chips">
        {#each currentTop as item}
          <span class="chip">{item.genre}: {item.share.toFixed(1)}%</span>
        {/each}
      </div>
    </div>

    <svg width={chartWidth} height={chartHeight} role="img" aria-label="Genre distribution timeline">
      <g transform="translate({margin.left},{margin.top})">
        {#each [0, 0.25, 0.5, 0.75, 1] as t}
          <line x1="0" x2={innerWidth} y1={yScale(t)} y2={yScale(t)} class="grid" />
          <text x="-10" y={yScale(t) + 4} text-anchor="end" class="axis-text">{Math.round(t * 100)}%</text>
        {/each}

        <defs>
          <clipPath id="reveal2d">
            <rect x="0" y="0" width={Math.max(0, revealWidth)} height={innerHeight} />
          </clipPath>
        </defs>

        <g clip-path="url(#reveal2d)">
          {#each stackSeries as layer}
            {#each layer as segment, idx}
              {@const y = years[idx]}
              {@const x = xBand(y) ?? 0}
              {@const h = yScale(segment[0]) - yScale(segment[1])}
              <rect
                x={x}
                y={yScale(segment[1])}
                width={xBand.bandwidth()}
                height={Math.max(0, h)}
                fill={colorScale(layer.key)}
                opacity="0.95"
              />
            {/each}
          {/each}
        </g>

        <line
          x1={yearCenter(compareYear)}
          x2={yearCenter(compareYear)}
          y1="0"
          y2={innerHeight}
          class="cursor compare"
        />
        <line
          x1={yearCenter(currentYear)}
          x2={yearCenter(currentYear)}
          y1="0"
          y2={innerHeight}
          class="cursor current"
        />

        <line x1="0" x2={innerWidth} y1={innerHeight} y2={innerHeight} class="axis" />
        {#each tickYears as t}
          {@const tx = yearCenter(t)}
          <line x1={tx} x2={tx} y1={innerHeight} y2={innerHeight + 6} class="axis" />
          <text x={tx} y={innerHeight + 20} text-anchor="middle" class="axis-text">{Math.round(t)}</text>
        {/each}
      </g>
    </svg>

    <div class="compare-box">
      <p><b>Top genres in {currentYear}</b> (vs {compareYear})</p>
      {#each compareSummary as row}
        <p>
          {row.genre}: {row.share.toFixed(1)}%
          <span class:up={row.deltaPctPoint > 0} class:down={row.deltaPctPoint < 0}>
            ({row.deltaPctPoint >= 0 ? "+" : ""}{row.deltaPctPoint.toFixed(1)} pp)
          </span>
        </p>
      {/each}
      <p class="small">Reference top genres in {compareYear}: {compareTop.map((d) => d.genre).join(", ")}</p>
    </div>

    <div class="legend">
      {#each genreKeys as key}
        <span><i style:background={colorScale(key)}></i>{key}</span>
      {/each}
    </div>
  </section>
</Scroll>

<style>
  .intro {
    margin-top: 0;
    color: #4a4f5b;
  }

  .story-step {
    min-height: 95vh;
    padding: 1rem 0 2rem;
    border-left: 3px solid #e8edf7;
    padding-left: 1rem;
  }

  .story-step h3 {
    margin: 0 0 0.5rem;
    color: #1f2a44;
  }

  .story-step p {
    margin: 0;
    line-height: 1.5;
    color: #38425d;
  }

  .viz {
    background: #f8fbff;
    border: 1px solid #d9e5f5;
    border-radius: 10px;
    padding: 0.9rem;
  }

  .meta {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 1rem;
    flex-wrap: wrap;
    margin-bottom: 0.35rem;
  }

  .meta p {
    margin: 0;
    color: #1d2740;
  }

  .chips {
    display: flex;
    gap: 0.4rem;
    flex-wrap: wrap;
  }

  .chip {
    background: #ecf2fb;
    color: #244266;
    border-radius: 999px;
    padding: 0.22rem 0.6rem;
    font-size: 0.78rem;
  }

  .grid {
    stroke: #d8e2f0;
    stroke-width: 1;
  }

  .axis {
    stroke: #46536d;
    stroke-width: 1;
  }

  .axis-text {
    fill: #3d4a65;
    font-size: 11px;
  }

  .cursor {
    stroke-width: 2;
  }

  .cursor.current {
    stroke: #1f2024;
  }

  .cursor.compare {
    stroke: #4a556f;
    stroke-width: 2;
    stroke-dasharray: 5 3;
  }

  .compare-box {
    margin-top: 0.45rem;
    padding: 0.5rem 0.6rem;
    background: #f0f4fb;
    border: 1px solid #d8e3f4;
    border-radius: 8px;
    color: #2a3550;
    font-size: 0.83rem;
  }

  .compare-box p {
    margin: 0.2rem 0;
  }

  .compare-box .small {
    color: #52617c;
  }

  .up {
    color: #0d7a3a;
    font-weight: 600;
  }

  .down {
    color: #b13232;
    font-weight: 600;
  }

  .legend {
    display: flex;
    gap: 0.8rem;
    flex-wrap: wrap;
    margin-top: 0.45rem;
    font-size: 0.83rem;
    color: #2f3c57;
  }

  .legend span {
    display: inline-flex;
    align-items: center;
    gap: 0.35rem;
  }

  .legend i {
    width: 12px;
    height: 12px;
    border-radius: 2px;
    display: inline-block;
  }
</style>
