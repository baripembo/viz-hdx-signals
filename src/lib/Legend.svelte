<script>
  import * as d3 from 'd3';

  export let maxCount;
  export let maxMarkerSize;
  export let minMarkerSize;

  let height = 175;
  let width = 125;
  let xCircle = 25;
  let xLabel = width - 50;
  let yCircle = 45;
  $: data = [1, round(maxCount/2), round(maxCount)]

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
<!--   <svg {width} {height}>
    {#each data as d}
      <circle cx={xCircle} cy={yCircle - size(d)} r={size(d)}></circle>
      <line x1={xCircle + size(d)} x2={xLabel} y1={yCircle - size(d*2)} y2={yCircle - size(d*2)}></line>
      <text x={xLabel} y={yCircle - size(d*2)}>{d}</text>
    {/each}
  </svg> -->
  <svg width="125" height="175" class="s-9IbsL3aTMVcY">
    <circle cx="25" cy="40" r="5" class="s-9IbsL3aTMVcY"></circle>
    <line x1="30" x2="75" y1="40" y2="40" class="s-9IbsL3aTMVcY"></line>
    <text x="75" y="40" class="s-9IbsL3aTMVcY">{data[0]}</text>

    <circle cx="25" cy="32" r="14" class="s-9IbsL3aTMVcY"></circle>
    <line x1="35" x2="75" y1="24" y2="24" class="s-9IbsL3aTMVcY"></line>
    <text x="75" y="24" class="s-9IbsL3aTMVcY">{data[1]}</text>

    <circle cx="25" cy="24" r="21" class="s-9IbsL3aTMVcY"></circle>
    <line x1="40" x2="75" y1="7" y2="7" class="s-9IbsL3aTMVcY"></line>
    <text x="75" y="7" class="s-9IbsL3aTMVcY">{data[2]}</text>
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