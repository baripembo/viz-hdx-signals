<script>
  import * as d3 from 'd3';

  export let maxCount;
  export let maxMarkerSize;
  export let minMarkerSize;

  let height = 175;
  let width = 125;
  let xCircle = 25;
  let xLabel = 75;
  let yCircle = 40;
  let xLine = 30;
  $: data = (maxCount<=2) ? [1] : [1, round(maxCount/2), round(maxCount)]

  //marker scale
  $: size = d3.scaleLinear()
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
    {#each data as d, i}
      <circle cx={xCircle} cy={yCircle - (7*i)} r={size(d)}></circle>
      <line x1={xLine + (5*i)} x2={xLabel} y1={yCircle - (15*i)} y2={yCircle - (15*i)}></line>
      <text x={xLabel} y={yCircle - (15*i)}>{d}</text>
    {/each}
  </svg>
</div>


<style lang='scss'>
  .legend-container {
    background-color: rgba(255,255,255,0.8);
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