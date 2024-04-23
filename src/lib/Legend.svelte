<script>
  import * as d3 from 'd3';
  import { onMount, onDestroy } from 'svelte';

  export let maxCount;
  export let maxMarkerSize;
  export let minMarkerSize;

  let height = 175;
  let width = 125;
  let xCircle = 20;
  let xLabel = width - 50;
  let yCircle = 40;
  let data = [1, maxCount]

  //marker scale
  let size = d3.scaleLinear()
    .domain([1, maxCount])
    .range([minMarkerSize, maxMarkerSize])
</script>

<div class='legend-container'>
  <h4>Number of Signals</h4>
  <svg {width} {height}>
    {#each data as d}
      <circle cx={xCircle} cy={yCircle - size(d)} r={size(d)}></circle>
      <line x1={xCircle + size(d)} x2={xLabel} y1={yCircle - size(d*2)} y2={yCircle - size(d*2)}></line>
      <text x={xLabel} y={yCircle - size(d*2)}>{d}</text>
    {/each}
  </svg>
</div>


<style lang='scss'>
  .legend-container {
    background-color: rgba(255,255,255,0.7);
    bottom: 20px;
    font-size: 14px;
    height: 100px;
    position: absolute;
    padding: 15px;
    right: 20px;
    width: 175px;
    z-index: 2;
    circle {
      fill: none;
      stroke: #F2645A;
    }
    line {
      stroke: #F2645A;
      stroke-dasharray: 2,2;
    }
    text {
      alignment-baseline: middle;
      font-size: 12px;
    }
  }
</style>