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

	$: maxCount = 0;

	let map, mapContainer, signalsGeoData, hoverTimer, currentSignals, countByCountry, fHover, tooltip, tooltipError;
	let numFormat = d3.format(',');
	let dateFormat = d3.utcFormat('%b %d, %Y');
	let mapHeight = 500;
	let minMarkerSize = 8;
	let maxMarkerSize = 20;	

	let data = signalsData;
	$: if (data != signalsData) {  
		data = signalsData;
    updateFeatures();
	}

	export const showPopup = (msg) => {
    tooltipError.setHTML(msg);
    tooltipError
      .addTo(map)
    	.setLngLat(map.getCenter());
	}  

	export const capitalizeFirstLetter = (word) => {
    let firstLetter = word.replace('_', ' ').charAt(0).toUpperCase();
    let remainingLetters = word.substring(1);
    return firstLetter + remainingLetters;
	}

	onMount(() => {
		mapContainer.style.height = mapHeight + 'px';

		//init map
	  map = new mapboxgl.Map({
	    container: mapContainer,
	    style: `mapbox://styles/humdata/cl3lpk27k001k15msafr9714b`,
	    center: center,
	    zoom: zoom,
	    minZoom: 1,
	    maxZoom: 5.5
	  });

	  map.addControl(new mapboxgl.NavigationControl({showCompass: false}))
	     .addControl(new mapboxgl.AttributionControl(), 'bottom-left');

	  tooltip = new mapboxgl.Popup({
			closeButton: true,
			closeOnClick: true,
			//closeOnMove: true,
			className: 'map-tooltip'
		});

		tooltipError = new mapboxgl.Popup({
			closeButton: true,
			closeOnClick: true,
			closeOnMove: true,
			className: 'map-tooltip-error'
		});

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
    // let countByCountry = data.reduce((a, {iso3}) => {
    //   a[iso3] = a[iso3] || {iso3, count: 0};
    //   a[iso3].count++;
    //   return a;
    // }, {});

    countByCountry = Object.values(data.reduce((a, {iso3, lat, lon}) => {
		  a[iso3] = a[iso3] || {iso3, alert_count: 0, lat, lon};
		  a[iso3].alert_count++;
		  return a;
		}, Object.create(null)));

		let signals = [];
		let hrpList = [];
		//create features from data array
		// data.forEach(function(signal, i) {
		// 	if (signal.iso3 !== '') {
		// 		//add alert count to properties
		// 		signal.alert_count = countByCountry[signal.iso3].count;

		// 		signals.push({
		// 			'type': 'Feature',
		// 			'geometry': {
		// 				'type': 'Point',
		// 				'coordinates': [signal.lon, signal.lat]
		// 			},
		// 			'properties': signal
		// 		});

		// 		//save list of hrps
		// 		if (signal.hrp_country == 'TRUE') hrpList.push(signal.iso3);
		// 	}
		// });
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
		let sizeScale = (maxCount<=1) ? minMarkerSize : [
      'interpolate',
      ['linear'],
      ['get', 'alert_count'],
      1, minMarkerSize,
      maxCount,
      maxMarkerSize
    ];

    let signalColor = '#007CE0';//#F2645A

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
          2,
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
	  // map.on('mouseenter', 'signals-dots', onMouseEnter);
	  // map.on('mouseleave', 'signals-dots', onMouseLeave);
	  map.on('mouseenter', 'signals-dots', (e) => {
       mouseover(e.features[0]);
    });
    map.on('mouseleave', 'signals-dots', mouseout);
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
		// if (map.getSource('signals-source')) {
	  // 	countByCountry = Object.values(data.reduce((a, {iso3, lat, lon}) => {
		// 	  a[iso3] = a[iso3] || {iso3, alert_count: 0, lat, lon};
		// 	  a[iso3].alert_count++;
		// 	  return a;
		// 	}, Object.create(null)));

		// 	let signals = [];
		// 	countByCountry.forEach(function(signal, i) {
		// 		signals.push({
		// 			'type': 'Feature',
		// 			'geometry': {
		// 				'type': 'Point',
		// 				'coordinates': [signal.lon, signal.lat]
		// 			},
		// 			'properties': signal
		// 		});
		// 	});

		// 	//update size scale for markers
		// 	maxCount = d3.max(countByCountry, d => d.alert_count);
		// 	console.log('maxCount', maxCount)
		// 	let sizeScale = [
	  //     'interpolate',
	  //     ['linear'],
	  //     ['get', 'alert_count'],
	  //     1, minMarkerSize,
	  //     maxCount,
	  //     maxMarkerSize
	  //   ];

		// 	//update geojson
		// 	signalsGeoData = {
		// 		'type': 'FeatureCollection',
		// 		'features': signals
		// 	}
	  // 	map.getSource('signals-source').setData(signalsGeoData);
	  // }

	  // //zoom map to marker bounds
	  // zoomToBounds();
	}

	function zoomToBounds() {
		//zoom map to bounds
		if (signalsGeoData !== undefined) {
		  let bounds = new mapboxgl.LngLatBounds();
			signalsGeoData.features.forEach(function(feature) {
			  bounds.extend(feature.geometry.coordinates);
			});
			map.fitBounds(bounds, {padding: {top: 100, right: 80, bottom: 175, left: 80}, duration: 500});
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

	//mouse event/leave events
	// function onMouseEnter(e) {
	// 	//clearTimeout(hoverTimer);
	// 	let iso3 = e.features[0].properties.iso3;
	// 	currentSignals = getAlerts(iso3);

	//   map.getCanvas().style.cursor = 'pointer';
	//   tooltip.addTo(map);
	// }
	// function onMouseLeave(e) {
	// 	// hoverTimer = setTimeout(() => {
	// 	//   map.getCanvas().style.cursor = '';
	// 	//   tooltip.remove();
	// 	// }, 2000);
	// }


	function mouseover(feature) {
    fHover = feature;
    map.getCanvasContainer().style.cursor = 'pointer';

    map.setFeatureState({
      source: 'signals-source',
      id: fHover.id
    }, {
      hover: true
    });
  }

  function mouseout() {
    if (!fHover) return;
    map.getCanvasContainer().style.cursor = '';
    map.setFeatureState({
      source: 'signals-source',
      id: fHover.id
    }, {
      hover: false
    });
    fHover = null;
  }

  function mouseclick(e) {
		let iso3 = e.features[0].properties.iso3;
		currentSignals = getAlerts(iso3);

    let numSignals = getNumAlerts(currentSignals[0].iso3)[0].alert_count;
    let content = `<h2>${currentSignals[0].country} <span>(${numSignals} ${numSignals>1 ? 'signals' : 'signal'})</span></h2>`;

    content += '<div class="signal-container">';
    currentSignals.forEach(function(signal) {
    	let signalDate = dateFormat(new Date(signal.date));
	    content += `<div class="signal">${signalDate}<br>`;
	    content += `${capitalizeFirstLetter(signal.indicator_name.replace('_', ' '))}, <span class="alert-level ${signal.alert_level.split(' ')[0]}">${signal.alert_level}</span><br>`;
	    if (signal.plot_url!=='NA') content += `<img class="plot" src="${signal.plot_url}" />`;
	    if (signal.plot_url!=='NA') content += `${signal.further_information}`;
	    content += '</div>';
    })
    content += '</div>';
    tooltip.setHTML(content);
    tooltip
      .addTo(map)
      .setLngLat(e.lngLat);
  }

	onDestroy(() => {
	  map.remove();
	});
</script>


<div bind:this={mapContainer} />
{#if maxCount}
	<Legend {minMarkerSize} {maxMarkerSize} {maxCount} />
{/if}


<style lang='scss'>
</style>