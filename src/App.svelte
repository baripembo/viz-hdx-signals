<script>
  import * as d3 from 'd3';
  import Papa from 'papaparse'
  import { onMount } from 'svelte';
  import RangeSlider from 'svelte-range-slider-pips'
  import Map from './lib/Map.svelte'

  let data, map, dateSlider, regions, indicators, latestDate, startDate, headerHeight;
  let filters = {region: [], indicator_name: []};
  let sliderDates = [];
  let coordsData = [];
  
  $: signalsData = [];
  $: errorMsg = '';
  $: hasHRP = true;

  const numMonths = 12;

  const coordsURL = 'https://raw.githubusercontent.com/OCHA-DAP/hdx-signals-alerts/DSCI-21-HDX-signals-alerts-pipeline/metadata/location_metadata.csv';
  const signalsURL = 'https://raw.githubusercontent.com/OCHA-DAP/hdx-signals-alerts/main/metadata/signals.csv';

  //set slider filter to show last 3 months of data
  let sliderDefault = [numMonths-3,numMonths];


  Promise.all([loadCSV(coordsURL), loadCSV(signalsURL)])
    .then(([coords, signals]) => {
      const cleanedSignals = signals.filter(d => d.iso3 !== '');
      dataLoaded(coords, cleanedSignals);
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
    // signals.sort(function(a,b) {
    //   return new Date(b.date) - new Date(a.date);
    // });
    // latestDate = new Date(signals[0].date);
    latestDate = new Date();
    console.log(latestDate);

    //add coords to matched signals
    signals.forEach((signal) => {
      let coords = findLatLong(signal.iso3);
      signal.lat = (coords!==null) ? coords.lat : null;
      signal.lon = (coords!==null) ? coords.lon : null;
    });

    //set start date 3 months prior to latest date
    startDate = new Date(latestDate);
    startDate.setMonth(startDate.getMonth() - numMonths);
    console.log('startDate',startDate, 'latestDate',latestDate)
    
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
    if (match && (match.Latitude!=='NA' || match.Longitude!=='NA')) {
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
    filters.hrp_location = (e.target.checked) ? 'TRUE' : '';
    filterData();
  }

  function onDateSelect(e) {
    const selected = e.detail.values;
    let startTime = new Date(sliderDates[selected[0]]).getTime();
    let endTime = new Date(sliderDates[selected[1]]).getTime();
    filters.date = [startTime, endTime];
    //filterData();
  }

  function onCheck(e) {
    let target = e.target;
    if (target.id==='All regions' || target.id==='All datasets') {
      let group = target.name.split('-')[0];
      let checks = d3.selectAll(`input[name="${group}"]`).nodes();
      checks.forEach(check => check.checked = target.checked);
    }
    else {
      const checks = d3.selectAll(`input[name="${target.name}"]`).nodes();
      d3.select(`input[name="${target.name}-all"]`).node().checked = Array.from(checks).every(check => check.checked);
    }
  }

  //formatter for date slider
  function formatter(value) {
    return d3.utcFormat('%b %Y')(sliderDates[value]);
  }

  function apply() {
    filters.region = [];
    filters.indicator_name = [];

    //get regions
    const regionChecks = d3.selectAll(`input[name="selectRegion"]`).nodes();
    regionChecks.forEach(check => {
      if (check.checked) filters.region.push(check.id);
    })

    //get indicators
    const indicatorChecks = d3.selectAll(`input[name="selectIndicator"]`).nodes();
    indicatorChecks.forEach(check => {
      if (check.checked) filters.indicator_name.push(check.id);
    })

    //get saved date values from onDateSelect

    //get hrp
    filters.hrp_location = (d3.select('#onlyHRP').node().checked) ? 'True' : '';

    //apply filters
    if (filters.region.length<1 || filters.indicator_name.length<1) {
      if (filters.region.length<1) {
        errorMsg = 'Please select at least one region';
      }
      if (filters.indicator_name.length<1) {
        errorMsg = 'Please select at least one dataset';
      }
    }
    else {
      filterData();
    }
  }

  function reset() {
    const checks = d3.selectAll('input[type="checkbox"]').nodes();
    checks.forEach(check => check.checked = true);
    sliderDefault = [numMonths-3,numMonths];
    filters = {region: '', indicator_name: '', date: [startDate, latestDate]};
    d3.select('#onlyHRP').node().checked = false;
    filterData();
  }

  function filterData() {
    errorMsg = '';
    map.closePopup();

    let result = data.filter(d => {
      let validSignal = true;
      
      for (const [key, value] of Object.entries(filters)) {
        if (Array.isArray(value)) {
          if (key==='region' || key==='indicator_name') {
            if (!value.includes(d[key])) validSignal = false;
          }
          if (key==='date') {
            let signalTime = new Date(d[key]).getTime();
            if (signalTime < value[0] || signalTime > value[1])
              validSignal = false;
            }
        }
        else {
          if (value!=='' && value.toLowerCase()!==d[key].toLowerCase())
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
    hasHRP = result.some(d => d['hrp_location'] === 'True');

    if (result.length>0) {
      signalsData = result;
    }
    else {
      errorMsg = 'There are no signals to display, please widen your selection';
    }
  }

  onMount(() => {
    d3.select('.filter-list').node().style.height = (map.isMobile()) ? ((window.innerHeight - headerHeight)*0.4) + 'px' : window.innerHeight - (d3.select('header').node().getBoundingClientRect().height) + 'px';
  });

</script>

<main>
  <header bind:clientHeight={headerHeight}>
    <div class='grid-container'>
      <div class='col-3 logo'><a href='https://data.humdata.org/signals' target='_blank'><img src='HDXSignalsLogo_white.png' alt='HDX Signals' /></a></div>
      <div class='col-7 description'>
        <p><a href='https://data.humdata.org/signals' target='_blank'>HDX Signals</a> monitors key datasets and generates automated emails when significant, negative changes are detected.</p>
        <p>Explore recent and historical signals in the map below, and click on any country bubble to see signals content and links to the original email. Read more about HDX Signals on <a href='https://data.humdata.org/signals' target='_blank'>our website</a>.</p>
      </div>
    </div>
  </header>

  <div class='grid-container'>
    <div class='col-3 filter-list'>
      <h5>Filter by:</h5>
      {#if regions}
        <ul>
          {#each regions as region, i}
            <li><label><input type='checkbox' id={region} name={i===0 ? 'selectRegion-all' : 'selectRegion'} on:change={onCheck} checked> {region}</label></li>
          {/each}
        </ul>
      {/if}

      <div class='input-wrapper'>
        <label class={hasHRP ? '' : 'is-disabled'}><input type='checkbox' id='onlyHRP' disabled={!hasHRP}> Only priority humanitarian locations</label>
      </div>

      {#if indicators}
        <ul>
          {#each indicators as indicator, i}
            <li><label><input type='checkbox' id={indicator} name={i===0 ? 'selectIndicator-all' : 'selectIndicator'} on:change={onCheck} checked> {map.capitalizeFirstLetter(indicator.replace('_', ' '))}</label></li>
          {/each}
        </ul>
      {/if}

      <div class='slider-container'>
        {#if sliderDates.length>0}
          <RangeSlider
            all='pips'
            id='dateSlider'
            first='label'
            float
            last='label'
            pips 
            max={numMonths}
            range
            step={1}
            bind:values={sliderDefault}
            on:change={onDateSelect} 
            {formatter} 
          />
        {/if}
      </div>

      <span class='error-msg'>{errorMsg}</span>
      
      <div class='buttons'>
        <button class='btn-reset' on:click={reset}>Reset</button>
        <button class='btn-apply' on:click={apply}>Apply</button>
      </div>

      <div class='logos'>
        <a href='https://www.unocha.org' target='_blank'><img src='logo-ocha-blue.png' alt='OCHA' width='110'></a>
        <a href='https://centre.humdata.org' target='_blank'><img class='centre' src='logo-centre-green.png' width='150' alt='Centre for Humanitarian Data'></a>
      </div>
    </div>
    
    <div class='col-9 map'>
      <Map bind:this={map} {signalsData} headerHeight={headerHeight} />
    </div>
  </div>

</main>

<style lang='scss'>
  main {
    position: relative;
  }
  .logos {
    align-items: center;
    display: flex;
    margin-top: 30px;
    img {
      &.centre {
        margin-left: 25px;
        margin-top: 12px;
      }
    }
  }
</style>