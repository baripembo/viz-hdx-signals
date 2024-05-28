<script>
  import * as d3 from 'd3';
  import Papa from 'papaparse'
  import { onMount } from 'svelte';
  import KeyFigure from './lib/KeyFigure.svelte'
  import Slider from './lib/Slider.svelte';
  import RangeSlider from 'svelte-range-slider-pips'
  import Map from './lib/Map.svelte'

  let data, map, dateSlider, slider, dates, regions, indicators, latestDate, startDate;
  let filters = {region: '', indicator_name: ''};

  let sliderDates = [];
  let sliderDefault = [0,3];
  
  $: signalsData = [];
  $: hasHRP = true;

  Papa.parse('signals.csv', {
    header: true,
    download: true,
    complete: function(results) {
      //get latest date in data
      results.data.sort(function(a,b) {
        return new Date(b.date) - new Date(a.date);
      });
      latestDate = new Date(results.data[0].date);

      //set start date 3 months prior to latest date
      startDate = new Date(latestDate);
      startDate.setMonth(startDate.getMonth() - 3);
      
      //get results within 3 months
      data = results.data.filter(d => d.iso3 !== '' && new Date(d.date).getTime() >= startDate.getTime());

      signalsData = data;
      console.log(signalsData);

      //dates = signalsData.map(d => new Date(d.date));
      
      createFilters();
      createDateSlider();
      console.log('sliderDates',sliderDates)
    }
  })

  function createFilters() {
    console.log(filters)
    if (filters.region=='') {
      //get list of regions
      let regionArray = signalsData.map(d => d.region);
      regionArray.unshift('All Regions');
      regions = [... new Set(regionArray)].sort();
    }
    if (filters.indicator_name=='') {
      //get list of indicators
      let indicatorArray = signalsData.map(d => d.indicator_name);
      indicatorArray.unshift('All Indicators');
      indicators = [... new Set(indicatorArray)].sort();
    }
  }

  function createDateSlider() {
    //const sliderHeight = 580;
    //dates = dates.sort((a, b) => a.getTime() - b.getTime());
    const listLength = 4; // months
    const dates = [];
    for(let i = 0; i < listLength; i++) {
        const itemDate = new Date(startDate); // starting today
        itemDate.setMonth(itemDate.getMonth() + i);
        dates.push(itemDate);
    }
    sliderDates = dates;

    slider = sliderHorizontal()
      .min(startDate)
      .max(latestDate)
      //.tickFormat(d3.utcFormat('%b %d, %Y'))
      .tickFormat(d3.utcFormat('%b %Y'))
      //.ticks(3)
      .step(1)
      .tickValues(dates)
      .marks(dates)
      .default([latestDate, startDate])
      .width(200)
      //.height(sliderHeight-50)
      .handle(d3.symbol().type(d3.symbolCircle).size(200)())
      .fill('#007CE0')
      .on('end', (val) => {
        onDateSelect(val)
      });

    const g = d3.select(dateSlider)
      .append('svg')
      .attr('width', 250)
      .attr('height', 70)
      .append('g')
      .attr('transform', 'translate(25,25)');

    g.call(slider);

    //console.log('---',slider.value([latestDate,startDate]))
  }

  function onRegionSelect(e) {
    filters.region = (e.target.value=='All Regions') ? '' : e.target.value;
    filterData();
  }

  function onIndicatorSelect(e) {
    filters.indicator_name = (e.target.value=='All Indicators') ? '' : e.target.value.toLowerCase();
    filterData();
  }

  function onHRPSelect(e) {
    filters.hrp_country = (e.target.checked) ? 'TRUE' : '';
    filterData();
  }

  function onDateSelect(e) {
    const selected = e.detail.values;
    let startTime = new Date(sliderDates[selected[0]]).getTime();
    let endTime = new Date(sliderDates[selected[1]]).getTime();
    filters.date = [startTime, endTime];
    filterData();
  }

  //formatter for date slider
  function formatter(value) {
    return d3.utcFormat('%b %Y')(sliderDates[value]);
  }


  function reset() {
    d3.select('#regionSelect').node().value = 'All Regions';
    d3.select('#indicatorSelect').node().value = 'All Indicators';
    sliderDefault = [0, 3];
    filters = {region: '', indicator_name: '', date: [startDate, latestDate]};
    d3.select('#onlyHRP').node().checked = false;
    filterData();
  }

  function filterData() {
    let result = data.filter(d => {
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
        // else if (typeof value === 'function') {
        //   if (!value(d[key])) {
        //     validSignal = false;
        //   }
        // }
      }
      return validSignal;
    });

    //set only HRP checkbox status
    hasHRP = result.some(d => d['hrp_country'] === 'TRUE');

    if (result.length>0) {
      signalsData = result;
    }
    else {
      map.showPopup(`There are no ${filters.indicator_name.replace('_', ' ')} signals in the ${filters.region} region for this selected time period`);
    }

    console.log(signalsData)
    //createFilters();
  }
</script>

<main>
  <header class='header'>
    <h1>HDX Signals</h1>
    <h5>Filter by:</h5>
    <div class='filters'>
      {#if regions}
        <div class='select-wrapper'>
          <select on:change={onRegionSelect} id='regionSelect'>
            {#each regions as region}
              <option value={region}>{region}</option>
            {/each}
          </select>
        </div>
      {/if}

      {#if indicators}
        <div class='select-wrapper'>
          <select on:change={onIndicatorSelect} id='indicatorSelect'>
            {#each indicators as indicator}
              <option value={indicator}>{map.capitalizeFirstLetter(indicator.replace('_', ' '))}</option>
            {/each}
          </select>
        </div>
      {/if}

      <div class='slider-container'>
        {#if sliderDates.length>0}
          <RangeSlider
            all='label'
            id='dateSlider'
            pips 
            max={3}
            range
            step={1}
            bind:values={sliderDefault}
            {formatter} 
            on:change={onDateSelect} 
          />
        {/if}
      </div>

      <div class='input-wrapper'>
        <input type='checkbox' id='onlyHRP' on:change={onHRPSelect} disabled={!hasHRP}> <label for='onlyHRP'>Only countries with a Humanitarian Response Plan</label>
      </div>

      <a href='#' class='btn-reset' on:click={reset}>Reset</a>
    </div>

    <!-- <div class='filters-secondary'></div> -->
  </header>
  
  <div class='map'>
    <Map bind:this={map} {signalsData} />
  </div>
</main>

<style lang='scss'>
  main {
    position: relative;
  }
  .filters,
  .filters-secondary {
    align-items: center;
    display: flex;
  }
  .slider-container {
    margin: 0 40px;
    width: 250px;
  }
  .btn-reset {
    margin-left: auto;
    text-decoration: underline;
    &:hover {
      text-decoration: none;
    }
  }
</style>
