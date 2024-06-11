<script>
	import mapboxgl from 'mapbox-gl';
	import * as d3 from 'd3';
	import { onMount, onDestroy } from 'svelte';
  import Legend from './Legend.svelte';
	import '../../node_modules/mapbox-gl/dist/mapbox-gl.css'

	mapboxgl.baseApiUrl = 'https://data.humdata.org/mapbox';
	mapboxgl.accessToken = 'cacheToken';

	export let signalsData;
	export let center = [20, 10];
	export let zoom = 2;
	export let headerHeight = 0;

	$: maxCount = 0;

	let map, mapContainer, signalsGeoData, hoverTimer, currentSignals, countByCountry, hoveredStateId, tooltip, tooltipError;
	let numFormat = d3.format(',');
	let dateFormat = d3.utcFormat('%b %d, %Y');
	let minMarkerSize = 8;
	let maxMarkerSize = 20;
	let minZoom = 1;

	$: if (mapContainer) {
		mapContainer.style.height = (isMobile()) ? ((window.innerHeight - headerHeight)*0.6) + 'px' : (window.innerHeight - headerHeight) + 'px';
	}

	let data = signalsData;
	$: if (data != signalsData) {  
		data = signalsData;
    updateFeatures();
	}


	export const showPopup = (msg, isError) => {
		const popup = d3.select('.popup');
		popup.select('.popup-content').html(msg);
		popup.style('display', 'block');
		popup.style('opacity', 1);
	}

	export const closePopup = () => {
		const popup = d3.select('.popup');
		popup.style('display', 'none');
		popup.style('opacity', 0);
	}

	export const capitalizeFirstLetter = (word) => {
    let firstLetter = word.replace('_', ' ').charAt(0).toUpperCase();
    let remainingLetters = word.substring(1);
    return firstLetter + remainingLetters;
	}

	export const isMobile = () => {
    const userAgentCheck = /Mobi|Android/i.test(navigator.userAgent);
    const screenSizeCheck = window.matchMedia("(max-width: 767px)").matches;
    return userAgentCheck || screenSizeCheck;
	}

	onMount(() => {
		mapContainer.style.height = (isMobile()) ? ((window.innerHeight - headerHeight)*0.6) + 'px' : (window.innerHeight - headerHeight) + 'px';

		//init map
	  map = new mapboxgl.Map({
	    container: mapContainer,
	    style: `mapbox://styles/humdata/cl3lpk27k001k15msafr9714b`,
	    center: center,
	    zoom: zoom,
	    minZoom: minZoom,
	    maxZoom: 5.5
	  });

	  map.addControl(new mapboxgl.NavigationControl({showCompass: false}))
	     .addControl(new mapboxgl.AttributionControl(), 'bottom-left');


	  map.on('load', function() {
	    console.log('Map loaded')
		  console.log('in map', data)

		  //initPolys();
		  loadFeatures();
	  });
	});

	// function initPolys() {
	//   //data join
	//   var expression = ['match', ['get', 'ISO3_CODE']];
	//   data.forEach(function(d, i) {
	//   	if (d.value!==undefined && i<3) {
	// 	    var val = +d.value;
	// 	    var color = '#888888';
	// 	    expression.push(d.iso3, color);
	//   	}
	//   });

	//   //default value for no data
	//   expression.push('#EEEEEE');
	  
	//   //set properties
	//   map.setPaintProperty('wrl polbnda 1m', 'fill-color', expression);
	// }

	function loadFeatures() {
	  //get alert count by country
    countByCountry = Object.values(data.reduce((a, {iso3, lat, lon}) => {
		  a[iso3] = a[iso3] || {iso3, alert_count: 0, lat, lon};
		  a[iso3].alert_count++;
		  return a;
		}, Object.create(null)));

		let signals = [];
		let hrpList = [];

		countByCountry.forEach(function(signal, i) {
			signals.push({
				'type': 'Feature',
				'geometry': {
					'type': 'Point',
					'coordinates': [signal.lon, signal.lat]
				},
				'properties': signal
			});
		});

		//create geojson
		signalsGeoData = {
			'type': 'FeatureCollection',
			'features': signals
		}
		map.addSource('signals-source', {
	    type: 'geojson',
	    data: signalsGeoData,
	    generateId: true 
	  });

		//create size scale for markers
		maxCount = d3.max(countByCountry, d => d.alert_count);
    let sizeScale = (maxCount <= 1) ? minMarkerSize : [
		  'interpolate',
		  ['linear'],
		  ['sqrt', ['get', 'alert_count']],
		  Math.sqrt(1), minMarkerSize,
		  Math.sqrt(maxCount), maxMarkerSize
		];

    let signalColor = '#007CE0';

		//add signal markers
	  map.addLayer({
	    id: 'signals-dots',
	    type: 'circle',
	    source: 'signals-source',
	    paint: {
	      'circle-color': signalColor,
	      'circle-stroke-color': signalColor,
	      'circle-opacity': 0.6,
	      'circle-radius': sizeScale,
	      'circle-stroke-width': [
          'case',
          ['boolean', ['feature-state', 'hover'], false],
          5,
          1
        ]
	    }
	  });

	  // //highlight hrp countries
	  // let expression = ['match', ['get', 'ISO3_CODE']];
	  // let hrps = [... new Set(hrpList)]
	  // hrps.forEach(function(d) {
		//    expression.push(d, '#CCE5F9');
	  // });
	  // expression.push('#EEEEEE');
	  // map.setPaintProperty('wrl polbnda 1m', 'fill-color', expression);

	  //mouse events

	  let hoveredStateId = null;

		map.on('mousemove', 'signals-dots', mousemove);
    map.on('mouseout', 'signals-dots', mouseout);
	  map.on('click', 'signals-dots', (e) => {
	  	mouseclick(e);
	  });

	  //zoom map to marker bounds
	  zoomToBounds();
	}

	function updateFeatures() {
		if (map.getSource('signals-source')) {
			map.removeLayer('signals-dots');
			map.removeSource('signals-source');
			loadFeatures();
		}
	}

	function zoomToBounds() {
		//zoom map to bounds
		let mapPadding = (isMobile()) ? {top: 20, right: 20, bottom: 20, left: 20} : {top: 100, right: 80, bottom: 150, left: 80};
		if (signalsGeoData !== undefined) {
		  let bounds = new mapboxgl.LngLatBounds();
			signalsGeoData.features.forEach(function(feature) {
			  bounds.extend(feature.geometry.coordinates);
			});
			map.fitBounds(bounds, {padding: mapPadding, duration: 500});
		}
	}

	function getNumAlerts(iso3) {
		return countByCountry.filter(d => d.iso3 == iso3);
	}

	function getAlerts(iso3) {
		//get all alerts for a country and sort by date
		let alerts = data.filter(d => d.iso3 == iso3);
		alerts.sort((a,b) => {
			return new Date(b.date) - new Date(a.date);
		});
		return alerts;
	}


	function mousemove(e) {
	  map.getCanvas().style.cursor = 'pointer';
	  if (e.features.length > 0) {
	    if (hoveredStateId) {
	      map.setFeatureState({
	        source: 'signals-source',
	        id: hoveredStateId
	      }, {
	        hover: false
	      });
	    }

	    hoveredStateId = e.features[0].id;
	    map.setFeatureState({
	      source: 'signals-source',
	      id: hoveredStateId
	    }, {
	      hover: true
	    });
	  }
  }

  function mouseout() {
		map.getCanvas().style.cursor = '';
		map.setFeatureState({
      source: 'signals-source',
      id: hoveredStateId
    }, {
      hover: false
    });
  }

  function mouseclick(e) {
		let iso3 = e.features[0].properties.iso3;
		currentSignals = getAlerts(iso3);

    let numSignals = getNumAlerts(currentSignals[0].iso3)[0].alert_count;
    let content = `<h2>${currentSignals[0].location} <span>(${numSignals} ${numSignals>1 ? 'signals' : 'signal'})</span></h2>`;

    content += '<div class="signal-container">';
    currentSignals.forEach(function(signal) {
    	let signalDate = dateFormat(new Date(signal.date));
	    content += `<div class="signal"><h3>${signalDate}: ${signal.indicator_title}</h3>`;
	    if (isValid(signal.summary_short)) content += `<p>${signal.summary_short}</p>`;
	    if (isValid(signal.plot)) content += `<img class="plot" src="${signal.plot}" />`;
	    if (isValid(signal.map)) content += `<img class="map" src="${signal.map}" />`;
	    if (isValid(signal.campaign_url)) content += `<p><a href="${signal.campaign_url}" target="_blank">Go to the email</a></p>`;
	    if (isValid(signal.further_information)) content += `${signal.further_information}`;
	    content += '</div>';
    })
    content += '</div>';

    //set popup content
    showPopup(content);
  }

  function isValid(url) {
  	return (url!=='NA' && url!=='nan' && url!==undefined && url!=='');
  }

  onMount(() => {
  	map.on('click', (e) => {
  		//close the popup if clicked on anywhere outside of signal-dots layer
      let featureSelected = map.queryRenderedFeatures(e.point, {layers: ['signals-dots']});
  		if (featureSelected.length<=0) {
  			closePopup();
  		}
    })
  });

	onDestroy(() => {
	  map.remove();
	});
</script>


<div bind:this={mapContainer} />
{#if maxCount}
	<Legend {minMarkerSize} {maxMarkerSize} {maxCount} />
{/if}

<div class='popup'>
	<i class='close-btn humanitarianicons-Exit-Cancel' on:click={closePopup} />
	<div class='popup-content'></div>
</div>


<style lang='scss'>
</style>