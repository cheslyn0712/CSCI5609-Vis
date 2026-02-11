<script>
    import sunsetImage from '$lib/assets/sunset.jpg';
    
    let maxClick = $state(6);
    let cnt = $state(6); // tip: https://svelte.dev/docs/svelte/$state
    
    // Reset counter when maxClick changes
    $effect(() => {
      cnt = maxClick;
    });
  
    function onClick() {
      // tip: Since DOM (i.e., the webpage content) will automatically update based on values, [<p id="info">Remaining Number of Clicks: {cnt}</p>]
      // we only need to change the cnt number here.
      if (cnt > 0) {
        cnt -= 1;
      }
    }
</script>
  
<h1>Chunlin Gong's VIS Site</h1>
<img
  width="200px"
  src={sunsetImage}
  alt="Sunset"
/>
<div>
  You can click up to
  <select 
      bind:value={maxClick}>
    {#each [2, 4, 6] as optionNum}
      <option value={optionNum}>
        {optionNum}
      </option>
    {/each}
  </select>
  times
</div>
<button onclick={onClick}> Click Me </button>

{#if cnt > 0}
  <p id="info">Remaining Number of Clicks: {cnt}</p>
{:else}
  <p>No more clicks allowed</p>
{/if}

<style>
  :global(body) {
    font-family: Arial, Helvetica, sans-serif;
  }
  button {
    background-color: #44aa66;
    color: white;
    font-size: xx-large;
    padding: 10px 20px;
    border: none;
    cursor: pointer;
    border-radius: 5px;
  }
</style>
