<script>
  import * as d3 from 'd3';

  export let maxCount;
  export let maxMarkerSize;
  export let minMarkerSize;

  let height = 175;
  let width = 125;
  let xCircle = 25;
  let xLabel = 75;
  let yCircle = 35;
  let xLine = 31;
  let yOffset = 13;
  let data;


  $: if (maxCount==1) {
      data = [1];
    }
    else if (maxCount > 1 && maxCount < 3) {
      data = [1, maxCount];
    }
    else {
      data = [1, round(maxCount/2), round(maxCount)];
    }


  //marker scale
  $: size = d3.scaleSqrt()
    .domain([1, maxCount])
    .range([minMarkerSize, maxMarkerSize]);

  //round to nearest 10
  function round(num) {
    let roundedVal = (num>10) ? Math.round(num/10)*10 : Math.round(num);
    return roundedVal;
  }
</script>

<div class='legend-container'>
  <h4>Number of Signals</h4>
  <svg {width} {height}>
    {#if data.length==1}
      <circle cx="25" cy="35" r="8"></circle>
      <line x1="31" x2="75" y1="35" y2="35"></line>
      <text x="75" y="35">1</text>
    {:else if data.length==2}
      <circle cx="25" cy="35" r="8"></circle>
      <line x1="32" x2="75" y1="35" y2="35"></line>
      <text x="75" y="35">1</text>
      <circle cx="25" cy="25" r="20"></circle>
      <line x1="40" x2="75" y1="9" y2="9"></line>
      <text x="75" y="9">2</text>
    {:else}
      {#each data as d, i}
        <circle cx={xCircle} cy={yCircle - (5*i)} r={size(d)}></circle>
        <line x1={xLine + (5*i)} x2={xLabel} y1={yCircle - (yOffset*i)} y2={yCircle - (yOffset*i)}></line>
        <text x={xLabel} y={yCircle - (yOffset*i)}>{d}</text>
      {/each}
    {/if}
  </svg>
</div>


<style lang='scss'>
</style>