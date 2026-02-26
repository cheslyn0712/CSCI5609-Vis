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

  const xExtent = $derived(d3.extent(data.map((d) => d.x)));
  const yExtent = $derived(d3.extent(data.map((d) => d.y)));
  const xScale = $derived(
    d3
      .scaleTime()
      .domain((xExtent[0] && xExtent[1] ? xExtent : [new Date(0), new Date()]) as [Date, Date])
      .range([usableArea.left, usableArea.right])
  );
  const yScale = $derived(
    d3
      .scaleLinear()
      .domain((yExtent[0] != null && yExtent[1] != null ? yExtent : [0, 1]) as [number, number])
      .range([usableArea.bottom, usableArea.top])
  );

  const lineGenerator = $derived(
    d3
      .line<{ x: Date; y: number }>()
      .x((d) => xScale(d.x))
      .y((d) => yScale(d.y))
      .curve(d3.curveBasis)
  );

  const path = $derived(lineGenerator(data));

  let xAxis: SVGGElement | undefined = $state(),
    yAxis: SVGGElement | undefined = $state(),
    brushElement: SVGGElement | undefined = $state();

  function updateAxis() {
    if (!xScale || !yScale) {
      return;
    }
    if (xAxis) d3.select(xAxis).call(d3.axisBottom(xScale));
    if (yAxis) d3.select(yAxis).call(d3.axisLeft(yScale));
  }

  function handleBrush(event: d3.D3BrushEvent<unknown>) {
    const selection = event.selection;
    if (selection) {
      const [start, end] = selection.map((v) => xScale.invert(v));
      yearRange = [start, end];
    } else {
      yearRange = undefined;
    }
  }

  function setupBrush() {
    if (!brushElement) return;
    const brush = d3
      .brushX<{ x: Date; y: number }>()
      .extent([
        [usableArea.left, usableArea.top],
        [usableArea.right, usableArea.bottom],
      ])
      .on("end", handleBrush);

    d3.select(brushElement).call(brush);
  }

  $effect(() => {
    setupBrush();
    updateAxis();
  });
</script>

<svg {width} {height} class="line">
  {#if path}
    <path d={path} fill="none" stroke="steelblue" stroke-width="2" />
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
  <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
  <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />
  <g class="brush" bind:this={brushElement} />

  <text x={width / 2} y={height - 5} text-anchor="middle">
    Number of Movies by Year:
  </text>
  {#if yearRange}
    <text x={width / 2} y={height - 20} text-anchor="middle">
      {yearRange[0].getFullYear()} - {yearRange[1].getFullYear()}
    </text>
  {:else}
    <text x={width / 2} y={height - 20} text-anchor="middle">
      Brush to select a range
    </text>
  {/if}
</svg>
