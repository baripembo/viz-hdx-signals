<script>
  import * as d3 from 'd3';
  import Papa from 'papaparse'
  import { onMount } from 'svelte';
  import RangeSlider from 'svelte-range-slider-pips'
  import Map from './lib/Map.svelte'

  let data, map, dateSlider, regions, indicators, defaultStartDate, startDate, endDate, headerHeight;
  let filters = {region: [], indicator_title: []};
  let sliderDates = [];
  let coordsData = [];
  let namesData = [];
  
  $: signalsData = [];
  $: errorMsg = '';
  $: hasHRP = true;

  //set number of months to show
  const numMonths = 12;

  const coordsURL = 'https://raw.githubusercontent.com/OCHA-DAP/hdx-signals-alerts/DSCI-21-HDX-signals-alerts-pipeline/metadata/location_metadata.csv';
  const signalsURL = 'https://raw.githubusercontent.com/OCHA-DAP/hdx-signals-alerts/main/metadata/signals.csv';
  const indicatorsURL = 'https://raw.githubusercontent.com/OCHA-DAP/hdx-signals-alerts/main/metadata/signals_indicators.csv';

  //set slider filter to show last 3 months of data
  let sliderDefault = [numMonths-3,numMonths];


  Promise.all([loadCSV(coordsURL), loadCSV(signalsURL), loadCSV(indicatorsURL)])
    .then(([coords, signals, indicator_titles]) => {
      const cleanedSignals = signals.filter(d => d.iso3 !== '');
      dataLoaded(coords, cleanedSignals, indicator_titles);
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

  function dataLoaded(coords, signals, indicator_titles) {
    coordsData = coords;
    namesData = indicator_titles;

    //sort data
    signals.sort(function(a,b) {
      return new Date(b.date) - new Date(a.date);
    });

    //add coords and indicator names to matched signals
    signals.forEach((signal) => {
      let coords = findLatLong(signal.iso3);
      signal.lat = (coords!==null) ? coords.lat : null;
      signal.lon = (coords!==null) ? coords.lon : null;

      signal.indicator_title = findIndicatorName(signal.indicator_id);
    });

    //set available date range
    endDate = new Date();
    startDate = new Date(endDate);
    startDate.setMonth(endDate.getMonth() - numMonths);

    //set starting date range - 3 months prior to latest date by default
    defaultStartDate = new Date(endDate);
    defaultStartDate.setMonth(endDate.getMonth() - 3);
    
    //filter full data within available date range
    data = signals.filter(d => new Date(d.date).getTime() >= startDate.getTime());

    //filter set of data within starting date range
    filters = {region: [], indicator_title: [], date: [defaultStartDate, endDate]};
    signalsData = signals.filter(d => new Date(d.date).getTime() >= defaultStartDate.getTime());
    
    createFilters();
    createDateSlider();
  }

  function findLatLong(iso3) {
    const match = coordsData.find(row => row['Alpha-3 code'] === iso3);
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

  function findIndicatorName(indicator_id) {
    const match = namesData.find(row => row['indicator_id'] === indicator_id);
    return (match) ? match.indicator_subject : null;
  }

  function createFilters() {
    if (filters.region=='') {
      //get list of regions
      let regionArray = signalsData.map(d => d.region);
      regionArray.sort();
      regionArray.unshift('All regions');
      regions = [... new Set(regionArray)];
    }
    if (filters.indicator_title=='') {
      //get list of indicators
      let indicatorArray = signalsData.map(d => d.indicator_title);
      indicatorArray.sort();
      indicatorArray.unshift('All datasets');
      indicators = [... new Set(indicatorArray)];
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
    filters.indicator_title = (e.target.value=='All datasets') ? '' : e.target.value.toLowerCase();
    filterData();
  }

  function onHRPSelect(e) {
    filters.hrp_location = (e.target.checked) ? 'True' : '';
    filterData();
  }

  function onDateSelect(e) {
    const selected = e.detail.values;
    let startTime = new Date(sliderDates[selected[0]]).getTime();
    let endTime = new Date(sliderDates[selected[1]]).getTime();
    filters.date = [startTime, endTime];
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
    filters.indicator_title = [];

    //get regions
    const regionChecks = d3.selectAll(`input[name="selectRegion"]`).nodes();
    regionChecks.forEach(check => {
      if (check.checked) filters.region.push(check.id);
    })

    //get indicators
    const indicatorChecks = d3.selectAll(`input[name="selectIndicator"]`).nodes();
    indicatorChecks.forEach(check => {
      if (check.checked) filters.indicator_title.push(check.id);
    })

    //get saved date values from onDateSelect

    //get hrp
    filters.hrp_location = (d3.select('#onlyHRP').node().checked) ? 'True' : '';

    //apply filters
    if (filters.region.length<1 || filters.indicator_title.length<1) {
      if (filters.region.length<1) {
        errorMsg = 'Please select at least one region';
      }
      if (filters.indicator_title.length<1) {
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
    sliderDefault = [numMonths-3, numMonths];
    filters = {region: '', indicator_title: '', date: [defaultStartDate, endDate]};
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
          if (key==='region' || key==='indicator_title') {
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

    //send mixpanel event with filter values
    mpTrack('map view', 'filtered', filters);
  }


  function initTracking() {
    //initialize mixpanel
    var MIXPANEL_TOKEN = window.location.hostname=='data.humdata.org'? '5cbf12bc9984628fb2c55a49daf32e74' : '99035923ee0a67880e6c05ab92b6cbc0';
    mixpanel.init(MIXPANEL_TOKEN);
    mixpanel.track('page view', {
      'page title': document.title,
      'page type': 'datavis'
    });
  }

  function mpTrack(view, content, filters) {
    //mixpanel event
    let eventObject = {
      'page title': document.title,
      'embedded in': window.location.href,
      'action': 'switch viz',
      'viz type': 'signals map',
      'current view': view,
      'content': content,
    }
    if (filters != undefined) {
      eventObject['region filter'] = filters.region.toString();
      eventObject['indicator filter'] = filters.indicator_title.toString();
      eventObject['is hrp filter'] = (filters.hrp_location === 'True') ? true : false;
      eventObject['date filter'] = formatDate(filters.date_value[0]) + ' - ' + formatDate(filters.date_value[1]);
    }
    mixpanel.track('viz interaction', eventObject);
  }

  function formatDate(date) {
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, '0');
    const day = String(date.getDate()).padStart(2, '0');
    return `${year}/${month}/${day}`;
  }

  onMount(() => {
    d3.select('.filter-list').node().style.height = (map.isMobile()) ? ((window.innerHeight - headerHeight)*0.4) + 'px' : window.innerHeight - (d3.select('header').node().getBoundingClientRect().height) + 'px';

    initTracking();
  });

</script>

<main>
  <header bind:clientHeight={headerHeight}>
    <div class='grid-container'>
      <div class='col-3 logo'><a href='https://data.humdata.org/signals' target='_blank'><img src='HDXSignalsLogo_white.png' alt='HDX Signals' /></a></div>
      <div class='col-7 description'>
        <p><a href='https://data.humdata.org/signals' target='_blank'>HDX Signals</a> monitors key datasets and generates automated emails when significant, negative changes are detected.</p>
        <p>Explore recent and historical signals in the map below, and click on any country bubble to see signals content and links to the original email. Access all Signals data directly on <a href='https://data.humdata.org/dataset/hdx-signals' target='_blank'>HDX</a>.</p>
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
            <li><label><input type='checkbox' id={indicator} name={i===0 ? 'selectIndicator-all' : 'selectIndicator'} on:change={onCheck} checked> {indicator}</label></li>
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
    margin-bottom: 20px;
    margin-top: auto;
    img {
      &.centre {
        margin-left: 25px;
        margin-top: 12px;
      }
    }
  }
</style>