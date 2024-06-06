<script>
  import * as d3 from 'd3';
  import Papa from 'papaparse'
  import { onMount } from 'svelte';
  import RangeSlider from 'svelte-range-slider-pips'
  import Map from './lib/Map.svelte'

  let data, map, dateSlider, regions, indicators, latestDate, startDate, headerHeight;
  let filters = {region: '', indicator_name: ''};
  let sliderDates = [];
  let sliderDefault = [0,3];
  let coordsData = [];
  
  $: signalsData = [];
  $: hasHRP = true;

  const numMonths = 3;

  const coordsURL = 'https://raw.githubusercontent.com/OCHA-DAP/hdx-signals-alerts/DSCI-21-HDX-signals-alerts-pipeline/metadata/location_metadata.csv';
  const signalsURL = 'hdx-signals.csv';//'https://stage.data-humdata-org.ahconu.org/dataset/30a97df6-ac93-4ff9-b08b-9c24038fd667/resource/ce6bfedf-8326-40f6-95e4-8213466b27a4/download/hdx-signals.csv';



  Promise.all([loadCSV(coordsURL), loadCSV(signalsURL)])
    .then(([coords, signals]) => {
      console.log('coords:', coords);
      console.log('signals:', signals);

      dataLoaded(coords, signals);
    })
    .catch(error => {
      console.error('Error loading files:', error);
    });


  function loadCSV(filePath) {
    return new Promise((resolve, reject) => {
      Papa.parse(filePath, {
        download: true,
        header: true,
        complete: function(results) {
          resolve(results.data);
        },
        error: function(error) {
          reject(error);
        }
      });
    });
  }

  function dataLoaded(coords, signals) {
    coordsData = coords;

    //get latest date in data
    signals.sort(function(a,b) {
      return new Date(b.date) - new Date(a.date);
    });
    latestDate = new Date(signals[0].date);

    //add coords to matched signals
    signals.forEach((signal) => {
      let coords = findLatLong(signal.iso3);
      signal.lat = (coords!==null) ? coords.lat : null;
      signal.lon = (coords!==null) ? coords.lon : null;
    });

    //set start date 3 months prior to latest date
    startDate = new Date(latestDate);
    startDate.setMonth(startDate.getMonth() - numMonths);
    
    //get results within 3 months
    data = signals.filter(d => d.iso3 !== '' && new Date(d.date).getTime() >= startDate.getTime());

    signalsData = data;
    console.log('signalsData', signalsData);
    
    createFilters();
    createDateSlider();
  }

  function findLatLong(iso3) {
    const match = coordsData.find(row => {
      return row['Alpha-3 code'] === iso3
    });
    if (match) {
      return {
        lat: +match.Latitude,
        lon: +match.Longitude
      };
    } 
    else {
      return null;
    }
  }

  function createFilters() {
    if (filters.region=='') {
      //get list of regions
      let regionArray = signalsData.map(d => d.region);
      regionArray.unshift('All regions');
      regions = [... new Set(regionArray)].sort();
    }
    if (filters.indicator_name=='') {
      //get list of indicators
      let indicatorArray = signalsData.map(d => d.indicator_name);
      indicatorArray.unshift('All datasets');
      indicators = [... new Set(indicatorArray)].sort();
    }
  }

  function createDateSlider() {
    const listLength = numMonths + 1; // months
    const dates = [];
    for(let i = 0; i < listLength; i++) {
      const itemDate = new Date(startDate);
      itemDate.setMonth(itemDate.getMonth() + i);
      dates.push(itemDate);
    }
    sliderDates = dates;
  }

  function onRegionSelect(e) {
    filters.region = (e.target.value=='All regions') ? '' : e.target.value;
    filterData();
  }

  function onIndicatorSelect(e) {
    filters.indicator_name = (e.target.value=='All datasets') ? '' : e.target.value.toLowerCase();
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
    d3.select('#regionSelect').node().value = 'All regions';
    d3.select('#indicatorSelect').node().value = 'All datasets';
    sliderDefault = [0, 3];
    filters = {region: '', indicator_name: '', date: [startDate, latestDate]};
    d3.select('#onlyHRP').node().checked = false;
    filterData();
  }

  function filterData() {
    map.closePopup();

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
      map.showPopup(`There are no ${filters.indicator_name.replace('_', ' ')} signals in the ${filters.region} region for this selected time period`, true);
    }
  }

  onMount(() => {
    if (map.isMobile()) d3.select('header').node().style.height = (window.innerHeight/2 - 20) + 'px';
  });

</script>

<main>
  <header bind:clientHeight={headerHeight}>
    <div class='intro'>
      <div class='logo'><a href='https://data.humdata.org/signals' target='_blank'><img src='HDXSignalsLogo_V2.png' alt='HDX Signals' /></a></div>
      <div class='description'>
        <p><a href='https://data.humdata.org/signals' target='_blank'>HDX Signals</a> monitors key datasets and generates automated emails when significant, negative changes are detected.</p>
        <p>Explore recent and historical signals in the map below, and click on any country bubble to see signals content and links to the original email. Read more about HDX Signals on <a href='https://data.humdata.org/signals' target='_blank'>our website</a>.</p>
      </div>
    </div>
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
        <input type='checkbox' id='onlyHRP' on:change={onHRPSelect} disabled={!hasHRP}> <label for='onlyHRP'>Only priority humanitarian locations</label>
      </div>

      <button class='btn-reset' on:click={reset}>Reset</button>
    </div>

    <!-- <div class='filters-secondary'></div> -->
  </header>
  
  <div class='map'>
    <Map bind:this={map} {signalsData} headerHeight={headerHeight} />
  </div>
</main>

<style lang='scss'>
  main {
    position: relative;
  }
</style>
