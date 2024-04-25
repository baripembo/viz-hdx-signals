<script>
  import * as d3 from 'd3';
  import { sliderRight } from 'd3-simple-slider';
  import Papa from 'papaparse'
  import { onMount } from 'svelte';
  import KeyFigure from './lib/KeyFigure.svelte'
  import Map from './lib/Map.svelte'

  let data, dateSlider, dates, regions, indicators;
  let filters = {};
  
  $: signalsData = [];

  Papa.parse('cholera_signals.csv', {
    header: true,
    download: true,
    complete: function(results) {
      data = results.data.filter(d => d.iso3 !== '');
      dates = data.map(d => new Date(d.date));
      
      //get list of regions
      let regionArray = data.map(d => d.region);
      regions = [... new Set(regionArray)];
      
      //get list of indicators
      let indicatorArray = data.map(d => d.indicator_name);
      indicators = [... new Set(indicatorArray)];

      //keep local copy of full dataset
      signalsData = data;

      createDateSlider();
    }
  })

  function createDateSlider() {
    const sliderHeight = 680;
    dates = dates.sort((a, b) => a.getTime() - b.getTime());

    const slider = sliderRight()
      .min(d3.min(dates))
      .max(d3.max(dates))
      .tickFormat(d3.utcFormat('%b %d, %Y'))
      .tickValues(dates)
      .marks(dates)
      .default([dates[dates.length-1], dates[0]])
      .width(150)
      .height(sliderHeight-50)
      .handle(d3.symbol().type(d3.symbolCircle).size(150)())
      .fill('#007CE0')
      .on('end', (val) => {
        onDateSelect(val)
      });

    const g = d3.select(dateSlider)
      .append('svg')
      .attr('width', 150)
      .attr('height', sliderHeight)
      .append('g')
      .attr('transform', 'translate(25,25)');

    g.call(slider);
  }

  function onRegionSelect(e) {
    filters.region = (e.target.value=='All Regions') ? '' : e.target.value;
    filterData();
    // if (e.target.value == 'All Regions') {
    //   signalsData = data;
    // }
    // else {
    //   signalsData = data.filter(d => d.region == e.target.value);
    // }
  }

  function onIndicatorSelect(e) {
    filters.indicator_name = (e.target.value=='All Indicators') ? '' : e.target.value.toLowerCase();
    filterData();
    // if (e.target.value == 'All Indicators') {
    //   signalsData = data;
    // }
    // else {
    //   signalsData = data.filter(d => d.indicator_name.toLowerCase() == e.target.value);
    // }
  }

  function onHRPSelect(e) {
    filters.hrp_country = (e.target.checked) ? 'TRUE' : '';
    filterData();
    // if (e.target.checked) {
    //   signalsData = data.filter(d => d.hrp_country == 'TRUE');
    // }
    // else {
    //   signalsData = data;
    // }
  }

  function onDateSelect(dateSelect) {
    let startTime = new Date(dateSelect[0]).getTime();
    let endTime = new Date(dateSelect[1]).getTime();
    filters.date = [startTime, endTime];
    filterData();
    // signalsData = data.filter((d) => {
    //   let signalTime = new Date(d.date).getTime();
    //   return signalTime >= startTime && signalTime <= endTime;
    // });
  }

  function filterData() {
    signalsData = data.filter(d => {
      let validSignal = true;
      
      for (const [key, value] of Object.entries(filters)) {
        if (Array.isArray(value)) {
          let signalTime = new Date(d[key]).getTime();
          if (signalTime < value[0] || signalTime > value[1])
            validSignal = false;
        }
        else {
          if (value!=='' && value!==d[key])
            validSignal = false;
        }
        // if (Array.isArray(value)) {
        //   if (!value.includes(d[key])) {
        //     validSignal = false;
        //   }
        // }
        // else if (typeof value === 'function') {
        //   if (!value(d[key])) {
        //     validSignal = false;
        //   }
        // }
        // else {
        //   if (value !== d[key]) validSignal = false;
        // }
      }
      return validSignal;
    });
  }
</script>

<main>
  <div class='filters'>
    {#if regions}
      <div class='select-wrapper'>
        <select on:change={onRegionSelect}>
          <option value={'All Regions'}>{'All Regions'}</option>
          {#each regions as region}
            <option value={region}>{region}</option>
          {/each}
        </select>
      </div>
    {/if}

    {#if indicators}
      <div class='select-wrapper'>
        <select on:change={onIndicatorSelect}>
          <option value={'All Indicators'}>{'All Indicators'}</option>
          {#each indicators as indicator}
            <option value={indicator}>{indicator}</option>
          {/each}
        </select>
      </div>
    {/if}

    <input type='checkbox' id='onlyHRP' on:change={onHRPSelect}> Only HRP
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
