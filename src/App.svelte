<script>
  import * as d3 from 'd3';
  import { sliderRight } from 'd3-simple-slider';
  import Papa from 'papaparse'
  import { onMount } from 'svelte';
  import KeyFigure from './lib/KeyFigure.svelte'
  import Map from './lib/Map.svelte'

  let data, dateSlider, dates, regions, regionSelect, indicators;
  
  $: signalsData = [];

  Papa.parse('cholera_signals.csv', {
    header: true,
    download: true,
    complete: function(results) {
      data = results.data.filter(d => d.iso3 !== '');
      dates = data.map(d => new Date(d.date));
      
      let regionArray = data.map(d => d.region);
      regions = [... new Set(regionArray)];
      
      let indicatorArray = data.map(d => d.indicator_name);
      indicators = [... new Set(indicatorArray)];

      signalsData = data;
      //console.log(signalsData)

      createDateSlider();
    }
  })

  function createDateSlider() {
    const sliderHeight = 680;
    dates = dates.sort((a, b) => a.getTime() - b.getTime());

    const slider = sliderRight()
      .min(d3.min(dates))
      .max(d3.max(dates))
      .tickFormat(d3.timeFormat('%b %d, %Y'))
      .step(1)
      .tickValues(dates)
      .marks(dates)
      .default([dates[dates.length-1],dates[0]])
      .width(150)
      .height(sliderHeight-50)
      .handle(d3.symbol().type(d3.symbolCircle).size(150)())
      .fill('#007CE0');

    const g = d3.select(dateSlider)
      .append('svg')
      .attr('width', 150)
      .attr('height', sliderHeight)
      .append('g')
      .attr('transform', 'translate(25,25)');

    g.call(slider);
  }

  function onHRP(e) {
    if (e.target.checked) {
      signalsData = data.filter(d => d.hrp_country == 'TRUE');
    }
    else {
      signalsData = data;
    }
  }
</script>

<main>
  <div class='filters'>
    {#if regions}
      <div class='select-wrapper'>
        <select bind:value={regionSelect}><!--on:change={() => getCountryData(countrySelect)}-->
          <option value={'All Regions'}>{'All Regions'}</option>
          {#each regions as region}
            <option value={region}>{region}</option>
          {/each}
        </select>
      </div>
    {/if}

    {#if indicators}
      <div class='select-wrapper'>
        <select>
          <option value={'All Indicators'}>{'All Indicators'}</option>
          {#each indicators as indicator}
            <option value={indicator}>{indicator}</option>
          {/each}
        </select>
      </div>
    {/if}

    <input type='checkbox' id="onlyHRP" on:change={onHRP}> Only HRP
  </div>
  <div class='slider-container'>
    <div class='slider' bind:this={dateSlider} />
  </div>
  <div class='map'>
    <Map {signalsData} />
  </div>
</main>

<style lang='scss'>
  main {
    position: relative;
  }
  .filters {
    align-items: baseline;
    display: flex;
  }
  .slider-container {
    background-color: rgba(255,255,255,0.7);
    height: 100%;
    position: absolute;
    width: 150px;
    z-index: 3;
  }
</style>
