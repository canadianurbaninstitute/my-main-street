<script>
	import { onMount } from "svelte";
	import mapboxgl from "mapbox-gl";
	import "../routes/styles.css";

	import LegendItem from "../lib/ui/legends/LegendItem.svelte";
	import Legend from "../lib/ui/legends/Legend.svelte";
	import cds from "../routes/assets/mms-cds.geo.json";
	import Select from "svelte-select";
	import * as turf from "@turf/turf";

	const cdnames = cds.features.map((feature) => feature.properties.CDNAME);

	let cdSelected = "";

	mapboxgl.accessToken =
		"pk.eyJ1IjoiY2FuYWRpYW51cmJhbmluc3RpdHV0ZSIsImEiOiJjbG95bzJiMG4wNW5mMmlzMjkxOW5lM241In0.o8ZurilZ00tGHXFV-gLSag";

	let map;

	let geocoder;

	const bounds = [
		[-84.64951046, 40.17694922],
		[-73.82038089, 47.7933883],
	];

	let ruralChecked = false;
	let equityChecked = false;
	let indigenousChecked = false;
	let mapLoaded = false; // Track the map's load state

	// Function to update the map filter based on checkbox states
	function updateMapFilter() {
		if (!mapLoaded) return; // Ensure the map has loaded
		const filters = ["all"];
		if (ruralChecked) {
			filters.push(["==", ["get", "Rural?"], "Yes"]);
		}
		if (equityChecked) {
			filters.push(["==", ["get", "PRIORITY - Equity"], "YES"]);
		}
		if (indigenousChecked) {
			filters.push(["==", ["get", "PRIORITY - Indigenous"], "YES"]);
		}
		map.setFilter("mms-business", filters);
	}

	function handleSelect(e) {

		if (map.getLayer('highlighted-cd')) {
      map.removeLayer('highlighted-cd');
    }
    if (map.getSource('highlighted-cd')) {
      map.removeSource('highlighted-cd');
    }

		// reset cma selected variable
		cdSelected = e.detail.value;

		const filteredFeature = cds.features.find(feature => feature.properties.CDNAME === cdSelected);

		console.log(filteredFeature);

		if (filteredFeature) {
			const bbox = turf.bbox(filteredFeature);
			map.fitBounds(bbox, { padding: 20 });
		}

		map.addSource('highlighted-cd', {
        type: 'geojson',
        data: {
          type: 'FeatureCollection',
          features: [filteredFeature]
        }
     	 });

      map.addLayer({
        id: 'highlighted-cd',
        type: 'line',
        source: 'highlighted-cd',
        paint: {
          'line-color': '#333333',
          'line-width': 2
        }
      });

		// not working with other filters yet
	
		// map.setFilter('mms-business', ['==', ['get', 'Region/Municipality'], cdSelected]);
	}

	onMount(() => {
		map = new mapboxgl.Map({
			container: "map",
			style: "mapbox://styles/canadianurbaninstitute/clwqlh0cl011j01qefx7o5ddn?fresh=true",
			center: [-79.1, 44.07],
			zoom: 5.8,
			minZoom: 5.8,
			maxZoom: 16,
			// maxBounds: bounds,
			scrollZoom: true,
			attributionControl: false,
		});

		map.addControl(new mapboxgl.NavigationControl(), "top-right");

		map.addControl(
			new mapboxgl.AttributionControl({
				customAttribution: "Canadian Urban Institute",
			}),
		);
		// Geocoder Search

		map.on("load", () => {
			mapLoaded = true; // Set the map's load state to true when loaded

			map.addSource("cds", {
				type: "geojson",
				data: cds,
			});

			// Add a new layer to visualize the polygon.
			map.addLayer(
				{
					id: "cds",
					type: "line",
					source: "cds",
					layout: {},
					paint: {
						"line-color": "#ddd",
						"line-width": 1,
					},
				},
				"mms-business",
			);

			map.addLayer({
				id: "cds-fill",
				type: "fill",
				source: "cds", // reference the data source
				layout: {},
				paint: {
					"fill-color": "#eee", // blue color fill
					"fill-opacity": 0.1,
				},
			});

			var popup = new mapboxgl.Popup({
				closeButton: false,
				closeOnClick: false,
			});

			// map.on('mouseenter', 'mms-business', function(e) {
			//     // Change the cursor to a pointer when the mouse is over the business-mms layer
			//     map.getCanvas().style.cursor = 'pointer';

			//     // Get the company name from the feature's properties
			//     var company = e.features[0].properties.Company;

			//     // Set the popup content and location
			//     popup.setLngLat(e.lngLat)
			//          .setHTML('<h4>' + company + '</h4>')
			//          .addTo(map);
			// });

			// map.on('mouseleave', 'mms-business', function() {
			//     // Reset the cursor when it leaves the business-mms layer
			//     map.getCanvas().style.cursor = '';

			//     // Remove the popup
			//     popup.remove();
			// });
		});
	});

	$: ruralChecked, updateMapFilter();
	$: equityChecked, updateMapFilter();
	$: indigenousChecked, updateMapFilter();
</script>

<svelte:head>
	<link
		href="https://api.mapbox.com/mapbox-gl-js/v2.14.1/mapbox-gl.css"
		rel="stylesheet"
	/>
	<script
		src="https://api.mapbox.com/mapbox-gl-js/plugins/mapbox-gl-geocoder/v5.0.0/mapbox-gl-geocoder.min.js"
	></script>
	<link
		rel="stylesheet"
		href="https://api.mapbox.com/mapbox-gl-js/plugins/mapbox-gl-geocoder/v5.0.0/mapbox-gl-geocoder.css"
		type="text/css"
	/>
</svelte:head>
<div id="content">
	<div id="controls">
		<h4>My Main Street</h4>
		<h5>Filter projects by:</h5>
		<div>
			<div class="checkbox-group">
				<label>
					<input type="checkbox" bind:checked={ruralChecked} />
					Rural
				</label>
				<label>
					<input type="checkbox" bind:checked={equityChecked} />
					Equity-deserving
				</label>
				<label>
					<input type="checkbox" bind:checked={indigenousChecked} />
					Indigenous
				</label>
			</div>
		</div>
		<h5>Select Region</h5>
		<div class="select-container">
			<Select
				id="select"
				items={cdnames}
				value={cdSelected}
				on:input={handleSelect}
				--background="white"
				--item-color="black"
				--item-is-active-color="black"
				--item-is-active-bg="#eee"
			/>
		</div>
		<div class="legend">
			<h4>Legend</h4>
			<LegendItem
				variant={"circle"}
				label={"Business Sustainability"}
				bgcolor={"#5E32BD"}
				bordercolor={"#fff"}
			/>
			<LegendItem
				variant={"circle"}
				label={"Community Activator"}
				bgcolor={"#006501"}
				bordercolor={"#ffffff"}
			/>
		</div>
	</div>
	<div id="map"/>
</div>


<style>
	:global(body) {
		padding: 0px;
		margin: 0px;
		background-color: white;
		height: 100%;
	}

	#content {
		display: flex;
		flex-direction: row;
		width: 100%;
	}

	#controls {
		width: 30vw;
		padding: 1em;
		display: flex;
		flex-direction: column;
		gap: 1em;
	}

	.checkbox-group {
		display: flex;
		flex-direction: column;
		border: 1px solid #eee;
		padding: 1em;
	}

	.select-container {
		margin-bottom: 1em;
	}

	#map {
		height: 100vh;
		width: 100%;
		position: relative;
		border-left: 1px solid #eee;
	}

	.legend {
		position: relative;
		background-color: #fff;
		padding: 1em;
		border-radius: 0.6em;
		border: 1px solid #eee;
		margin: 0 0 0.5em 0;
	}
</style>
