<script>
	import mapboxgl from 'mapbox-gl';
	import * as d3 from 'd3';
	import { legendColor } from 'd3-svg-legend';
	import { onMount, onDestroy } from 'svelte';
	import '../../node_modules/mapbox-gl/dist/mapbox-gl.css'

	mapboxgl.baseApiUrl = 'https://data.humdata.org/mapbox';
	mapboxgl.accessToken = 'cacheToken';

	export let signalsData;
	export let center = [20, 10];
	export let zoom = 2;

	let map, mapContainer, signalsGeoData, legend, colorScale, hoverTimer;

	let colorRange = ['#C25048','#F2645A','#F7A29C','#FCE0DE'];
	let tooltip = d3.select('.tooltip');
	let numFormat = d3.format(',');
	let dateFormat = d3.utcFormat('%b %d, %Y');


	onMount(() => {
		//calculate available space for map
		let mapHeight = 700;
		mapContainer.style.height = mapHeight + 'px';

		//init map
	  map = new mapboxgl.Map({
	    container: mapContainer,
	    style: `mapbox://styles/humdata/cl3lpk27k001k15msafr9714b`,
	    center: center,
	    zoom: zoom
	  });

	  map.addControl(new mapboxgl.NavigationControl({showCompass: false}))
	     .addControl(new mapboxgl.AttributionControl(), 'bottom-left');

	  tooltip = new mapboxgl.Popup({
			closeButton: true,
			closeOnClick: true,
			closeOnMove: true,
			className: 'map-tooltip'
		});

	  map.on('load', function() {
	    console.log('Map loaded')
		  console.log('in map', signalsData)
			
		  //set alert level color scale
			colorScale = d3.scaleOrdinal()
      .domain(['High concern', 'Medium concern', 'Low concern'])
      .range(colorRange);

		  //initPolys();
		  loadFeatures();
		  createLegend();
	  });


	});


	function createLegend() {
		const colorLegend = legendColor()
      .shape('path', d3.symbol().type(d3.symbolCircle).size(120)())
			.scale(colorScale)
		
		d3.select(legend)
			.call(colorLegend);
	}

	// function initPolys() {
	//   //data join
	//   var expression = ['match', ['get', 'ISO3_CODE']];
	//   signalsData.forEach(function(d, i) {
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
		let signals = [];
		let hrpList = [];
		signalsData.forEach(function(signal, i) {
			if (signal.iso3 !== '') {
				signals.push({
					'type': 'Feature',
					'geometry': {
						'type': 'Point',
						'coordinates': [signal.lon, signal.lat]
					},
					'properties': signal
				});

				//save list of hrps
				if (signal.hrp_country == 'TRUE') hrpList.push(signal.iso3);
			}
		});

		signalsGeoData = {
			'type': 'FeatureCollection',
			'features': signals
		}
		map.addSource('signals-source', {
	    type: 'geojson',
	    data: signalsGeoData,
	    generateId: true 
	  });


		//create color scale for markers
		let indicatorColorScale = [
	    'match',
	    ['get', 'alert_level'],
	    'High concern',
	    colorRange[0],
	    'Medium concern',
	    colorRange[1],
	    'Low concern',
	    colorRange[2],
	    /* other */ '#ccc'
	  ];

		//add signal markers
	  map.addLayer({
	    id: 'signal-dots',
	    type: 'circle',
	    source: 'signals-source',
	    paint: {
	      'circle-color': indicatorColorScale,
	      'circle-stroke-color': indicatorColorScale,
	      'circle-opacity': 0.6,
	      'circle-radius': 6,
	      'circle-stroke-width': 1,
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
	  map.on('mouseenter', 'signal-dots', onMouseEnter);
	  map.on('mouseleave', 'signal-dots', onMouseLeave);
	  map.on('mousemove', 'signal-dots', function(e) {
	    map.getCanvas().style.cursor = 'pointer';
	    let prop = e.features[0].properties;
	    let alertDate = dateFormat(new Date(prop.date));
	    let content = `<h2>${prop.country}</h2>`;
	    content += `${alertDate}<br>`;
	    content += `<div class="alert-table"><div>Indicator: <span class="stat">${prop.indicator_name}</span></div>`;
	    content += `<div>Alert Level: ${prop.alert_level}</div></div>`;
	    // content += `Value: <span class="stat">${numFormat(prop.value)}</span><br>`;
	    // content += `Source: <span class="stat"><a href='${prop.source_url}' target='_blank'>${prop.indicator_source}</a></span><br>`;
	    content += `<img class="plot" src="${prop.plot_url}" />`;
	    content += `${prop.further_information}`;
	    tooltip.setHTML(content);
	    tooltip
	      .addTo(map)
	      .setLngLat(e.lngLat);
	  });
	}

	//mouse event/leave events
	function onMouseEnter(e) {
		clearTimeout(hoverTimer);
	  map.getCanvas().style.cursor = 'pointer';
	  tooltip.addTo(map);
	  console.log(d3.select('.mapboxgl-popup-content'));
	  console.log(d3.select('.map-tooltip'));
	}
	function onMouseLeave(e) {
		// hoverTimer = setTimeout(() => {
		//   map.getCanvas().style.cursor = '';
		//   tooltip.remove();
		// }, 2000);
	}

	onDestroy(() => {
	  map.remove();
	});
</script>


<div bind:this={mapContainer} />
<div class='legend-container'>
	<h4>Map Legend</h4>
	<svg>
		<g bind:this={legend} />
	</svg>
</div>

<style lang='scss'>
	.legend-container {
		background-color: rgba(255,255,255,0.7);
		bottom: 20px;
		font-size: 14px;
		height: 150px;
		position: absolute;
		padding: 15px;
		right: 20px;
		width: 200px;
		z-index: 2;
		svg g {
			transform: translate(7px, 7px);
		}
	}
</style>