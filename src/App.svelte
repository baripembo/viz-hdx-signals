<script>
  import * as d3 from 'd3';
  import Papa from 'papaparse'
  import { onMount } from 'svelte';
  import KeyFigure from './lib/KeyFigure.svelte'
  import Slider from './lib/Slider.svelte';
  import { sliderRight, sliderHorizontal } from 'd3-simple-slider';
  import Map from './lib/Map.svelte'
  
  let p = 'MyName'
  $: nameUpper = p.toUpperCase()

  let data, map, dateSlider, dates, regions, indicators;
  let filters = {region: '', indicator_name: ''};
  
  $: signalsData = [];
  $: hasHRP = true;

  Papa.parse('signals.csv', {
    header: true,
    download: true,
    complete: function(results) {
      //get latest date
      results.data.sort(function(a,b) {
        return new Date(b.date) - new Date(a.date);
      });
      let latestDate = new Date(results.data[0].date);
      console.log(latestDate)

      //get list of dates -- restrict to 3 months
      let startDate = latestDate;
      startDate.setMonth(startDate.getMonth() - 3);
      console.log(startDate)

      data = results.data.filter(d => d.iso3 !== '' && new Date(d.date).getTime() >= startDate.getTime());

      //keep local copy of full dataset
      signalsData = data;
      console.log(signalsData);

      // dates = data.filter(d => {
      //   if (new Date(d.date).getTime() >= startDate.getTime()) console.log(new Date(d.date))
      //   return (new Date(d.date).getTime() >= startDate.getTime())
      // });
      dates = signalsData.map(d => new Date(d.date));
      
      createFilters();

      createDateSlider();
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
    const sliderHeight = 580;
    dates = dates.sort((a, b) => a.getTime() - b.getTime());

    const slider = sliderHorizontal()
      .min(d3.min(dates))
      .max(d3.max(dates))
      //.tickFormat(d3.utcFormat('%b %d, %Y'))
      .tickFormat(d3.utcFormat('%b %Y'))
      .ticks(5)
      //.tickValues(dates)
      //.marks(dates)
      .default([dates[dates.length-1], dates[0]])
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

  function onDateSelect(dateSelect) {
    let startTime = new Date(dateSelect[0]).getTime();
    let endTime = new Date(dateSelect[1]).getTime();
    filters.date = [startTime, endTime];
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

  function capitalizeFirstLetter(word) {
    let firstLetter = word.charAt(0).toUpperCase();
    let remainingLetters = word.substring(1);
    return firstLetter + remainingLetters;
  }
</script>

<main>
  <div class='filters'>
    {#if regions}
      <div class='select-wrapper'>
        <select on:change={onRegionSelect}>
          {#each regions as region}
            <option value={region}>{region}</option>
          {/each}
        </select>
      </div>
    {/if}

    {#if indicators}
      <div class='select-wrapper'>
        <select on:change={onIndicatorSelect}>
          {#each indicators as indicator}
            <option value={indicator}>{capitalizeFirstLetter(indicator.replace('_', ' '))}</option>
          {/each}
        </select>
      </div>
    {/if}

    <input type='checkbox' id='onlyHRP' on:change={onHRPSelect} disabled={!hasHRP}> Only HRP

<!--     <Slider bind:person={p} />

    <p>Reactive value in the parent component: {nameUpper}</p>
 -->
    <div class='slider-container'>
      <div class='slider' bind:this={dateSlider} />
    </div>
  </div>
  
  <div class='map'>
    <Map bind:this={map} {signalsData} />
  </div>
</main>

<style lang='scss'>
  main {
    position: relative;
  }
  .filters {
    align-items: center;
    display: flex;
  }
  // .slider-container {
  //   background-color: rgba(255,255,255,0.7);
  //   height: 100%;
  //   position: absolute;
  //   width: 150px;
  //   z-index: 3;
  // }
  .slider-container {

  }
</style>
