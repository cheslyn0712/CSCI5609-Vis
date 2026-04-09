<!-- Inspired by Lea Verou's scrolly pattern -->
<script lang="ts">
  import { onMount } from "svelte";

  type Props = {
    progress?: number;
    progressRaw?: number;
    threshold?: number;
    margin?: number;
    debounce?: number | boolean;
    throttle?: number | boolean;
  };

  let {
    progress = $bindable(),
    progressRaw,
    threshold = 0.5,
    margin = 30,
    debounce = false,
    throttle = false,
  }: Props = $props();

  let container: HTMLElement;
  let minTop = 0;
  let maxTop = 100;
  let pageTop = 0;
  let intersectionObserver: IntersectionObserver | undefined;
  let resizeObserver: ResizeObserver | undefined;

  const clamp = (min: number, value: number, max: number) =>
    Math.min(Math.max(min, value), max);
  const getProgress = (value: number, min: number, max: number) =>
    (100 * (value - min)) / (max - min);
  const runImmediately = (fn: () => void) => fn();
  const identity = (fn: () => void) => fn;

  let last = 0;
  const throttled = $derived(
    (throttle as number) > 0
      ? (fn: () => void) => {
          return () => {
            const now = performance.now();
            if (now - last >= (throttle as number)) {
              fn();
              last = now;
            }
          };
        }
      : identity,
  );

  let debouncerId = 0;
  const debounced = $derived(
    debounce
      ? (debounce as number) > 0
        ? (fn: () => void) => {
            clearTimeout(debouncerId);
            debouncerId = window.setTimeout(fn, debounce as number);
          }
        : (fn: () => void) => {
            cancelAnimationFrame(debouncerId);
            debouncerId = requestAnimationFrame(fn);
          }
      : runImmediately,
  );

  onMount(() => {
    function updateProgress() {
      const clampedProgress = clamp(0, progressRaw as number, 100);
      if (clampedProgress === 0 || clampedProgress === 100) {
        progress = clampedProgress;
      } else {
        debounced(throttled(() => (progress = clampedProgress)));
      }
    }

    function calculateProgress(top = container.getBoundingClientRect().top): void {
      progressRaw = getProgress(top, minTop, maxTop);
      updateProgress();
    }

    function calculateBounds() {
      const rect = container.getBoundingClientRect();
      pageTop = window.scrollY + rect.top;
      minTop = Math.min(pageTop, innerHeight * threshold) + margin;
      maxTop = innerHeight - rect.height + margin;
      calculateProgress(rect.top);
    }

    function attachListeners() {
      window.addEventListener("scroll", handleScroll);
      window.addEventListener("resize", calculateBounds);
      resizeObserver?.observe(container);
    }

    function detachListeners() {
      window.removeEventListener("scroll", handleScroll);
      window.removeEventListener("resize", calculateBounds);
      resizeObserver?.unobserve(container);
    }

    function handleScroll() {
      calculateProgress();
    }

    intersectionObserver = new IntersectionObserver((entries) => {
      const lastEntry = entries.at(-1);
      if (!lastEntry) return;
      if (lastEntry.isIntersecting) {
        calculateBounds();
        calculateProgress();
        attachListeners();
      } else {
        detachListeners();
      }
    });

    resizeObserver = new ResizeObserver(calculateBounds);
    intersectionObserver.observe(container);
    calculateBounds();

    return () => {
      detachListeners();
      intersectionObserver?.disconnect();
      resizeObserver?.disconnect();
    };
  });
</script>

<section class="scrolly" bind:this={container} style="--scrolly-margin: {margin}">
  <section class="story">
    <!-- svelte-ignore slot_element_deprecated -->
    <slot />
  </section>
  <section class="viz">
    <!-- svelte-ignore slot_element_deprecated -->
    <slot name="viz" />
  </section>
</section>

<style>
  .scrolly {
    position: relative;
    display: grid;
    grid-template-columns: var(--scrolly-story-width, 1fr) var(--scrolly-viz-width, 1fr);
    grid-auto-flow: row dense;
    gap: var(--scrolly-gap, 4rem);
  }

  .viz,
  .story {
    grid-row: 1;
  }

  .viz {
    position: sticky;
    top: max(var(--scrolly-margin, 0) * 1px, var(--scrolly-viz-top, 2em));
    max-height: 100vh;
  }

  @container style(--scrolly-layout: viz-first) {
    .scrolly {
      grid-template-columns: var(--scrolly-viz-width, 1fr) var(--scrolly-story-width, 1fr);
    }

    .viz {
      grid-column: 1;
    }

    .story {
      grid-column: 2;
    }
  }

  @container style(--scrolly-layout: overlap) {
    .scrolly {
      grid-template-columns: 1fr;
    }

    .viz,
    .story {
      grid-column: 1;
    }
  }
</style>
