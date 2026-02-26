<script lang="ts">
  import * as d3 from "d3";

  type TProps = {
    data: Array<{ x: Date; y: number }>;
    yearRange: [Date, Date] | undefined;
    width?: number;
    height?: number;
  };
  let {
    data = [],
    yearRange = $bindable(),
    height = 150,
    width = 600,
  }: TProps = $props();

  const margin = { top: 15, bottom: 50, left: 30, right: 10 };
  const innerWidth = $derived(width - margin.left - margin.right);
  const innerHeight = $derived(height - margin.top - margin.bottom);

  const xMin = $derived(data.length ? d3.min(data, (d) => d.x) : null);
  const xMax = $derived(data.length ? d3.max(data, (d) => d.x) : null);
  const yMin = $derived(data.length ? d3.min(data, (d) => d.y) : null);
  const yMax = $derived(data.length ? d3.max(data, (d) => d.y) : null);

  const xScale = $derived(
    d3
      .scaleTime()
      .domain(
        xMin && xMax ? [xMin, xMax] : [new Date(1970, 0, 1), new Date()]
      )
      .range([0, innerWidth])
  );
  const yScale = $derived(
    d3
      .scaleLinear()
      .domain(yMin != null && yMax != null ? [yMin, yMax] : [0, 1])
      .range([innerHeight, 0])
  );

  const line = $derived(
    d3
      .line<{ x: Date; y: number }>()
      .x((d) => xScale(d.x))
      .y((d) => yScale(d.y))
      .curve(d3.curveBasis)(data)
  );

  let brushG: SVGGElement | undefined = $state();
  let xAxisG: SVGGElement | undefined = $state();
  let yAxisG: SVGGElement | undefined = $state();

  function applyBrush(brushNode: SVGGElement) {
    if (data.length === 0) return;
    const domain = xMin && xMax ? [xMin, xMax] : [new Date(1970, 0, 1), new Date()];
    const xScaleSvg = d3
      .scaleTime()
      .domain(domain as [Date, Date])
      .range([margin.left, margin.left + innerWidth]);

    const yScaleSvg = d3
      .scaleLinear()
      .domain(
        (yMin != null && yMax != null ? [yMin, yMax] : [0, 1]) as [number, number]
      )
      .range([margin.top + innerHeight, margin.top]);

    d3.select(xAxisG).call(d3.axisBottom(xScaleSvg));
    d3.select(yAxisG).call(d3.axisLeft(yScaleSvg));

    const brush = d3
      .brushX()
      .extent([
        [margin.left, margin.top],
        [margin.left + innerWidth, margin.top + innerHeight],
      ])
      .on("brush end", (event: d3.D3BrushEvent<unknown>) => {
        const sel = event.selection;
        if (!sel) {
          yearRange = undefined;
          return;
        }
        const px0 = Math.min(sel[0] as number, sel[1] as number);
        const px1 = Math.max(sel[0] as number, sel[1] as number);
        const d0 = xScaleSvg.invert(px0);
        const d1 = xScaleSvg.invert(px1);
        yearRange = [d0, d1];
      });

    d3.select(brushNode).call(brush);
  }

  $effect(() => {
    if (!brushG || !xAxisG || !yAxisG || data.length === 0) return;
    applyBrush(brushG);
    return () => {
      d3.select(brushG).selectAll(".overlay, .selection, .handle").remove();
    };
  });
</script>

<svg {width} {height} class="line-chart">
  <g class="chart-area" transform="translate({margin.left}, {margin.top})">
    {#if line}
      <path d={line} fill="none" stroke="steelblue" stroke-width="2" />
    {/if}
    <g class="points">
      {#each data as point (point.x.getTime())}
        <circle
          cx={xScale(point.x)}
          cy={yScale(point.y)}
          r="4"
          fill="steelblue"
          opacity="0.6"
        />
      {/each}
    </g>
  </g>
  <g
    transform="translate(0, {margin.top + innerHeight})"
    bind:this={xAxisG}
  ></g>
  <g transform="translate({margin.left}, 0)" bind:this={yAxisG}></g>
  <g class="brush-layer" bind:this={brushG}></g>

  <text x={width / 2} y={height - 5} text-anchor="middle">
    Number of Movies by Year:
  </text>
  {#if yearRange && yearRange[0] instanceof Date && yearRange[1] instanceof Date}
    <text x={width / 2} y={height - 20} text-anchor="middle">
      {yearRange[0].getFullYear()} - {yearRange[1].getFullYear()}
    </text>
  {:else}
    <text x={width / 2} y={height - 20} text-anchor="middle">
      Brush to select a range
    </text>
  {/if}
</svg>
