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

	// Define the equity tags based on your standardized mapping
	const equityTags = [
		'Persons with Disabilities', 
		'Visible Minorities',
		"2SLGBTQI+",
		'Women',
		"Newcomers, immigrants, or refugees",
		"Youth (39 and under)",
		"Members of Official Language Minority Communities",
	];

	// Create a reactive store for selected filters
	let selectedFilters = new Set();

		// Function to handle checkbox changes
		function handleFilterChange(tag) {
		if (selectedFilters.has(tag)) {
			selectedFilters.delete(tag);
		} else {
			selectedFilters.add(tag);
		}
		selectedFilters = selectedFilters; // trigger reactivity
		updateMapFilter();
	}

		// Function to check if a feature has all selected tags
		function featureHasAllTags(featureTags, selectedTags) {
		// Convert feature tags from string to array if needed
		const tagArray = typeof featureTags === 'string' 
			? featureTags.split(';').map(tag => tag.trim())
			: featureTags || [];
			
		// Check if all selected tags are present in the feature's tags
		return Array.from(selectedTags).every(tag => tagArray.includes(tag));
	}

	// Function to update the map filter with AND logic
	function updateMapFilter() {
		if (!map || !mapLoaded) return;

		if (selectedFilters.size === 0) {
			// If no filters selected, show all features
			map.setFilter('mms-business', null);
			return;
		}

		// Create a filter expression that requires all selected tags to be present
		const filterExpression = [
			"all",
			...Array.from(selectedFilters).map(tag => [
				"in",
				tag,
				["get", "equity_led"]
			])
		];

		map.setFilter('mms-business', filterExpression);
	}

	mapboxgl.accessToken =
		"pk.eyJ1IjoiY2FuYWRpYW51cmJhbmluc3RpdHV0ZSIsImEiOiJjbG95bzJiMG4wNW5mMmlzMjkxOW5lM241In0.o8ZurilZ00tGHXFV-gLSag";

	let map;
	let mapLoaded = false;

	const bounds = [
		[-84.64951046, 40.17694922],
		[-73.82038089, 47.7933883],
	];

	function handleSelect(e) {
		if (map.getLayer("highlighted-cd")) {
			map.removeLayer("highlighted-cd");
		}
		if (map.getSource("highlighted-cd")) {
			map.removeSource("highlighted-cd");
		}

		cdSelected = e.detail.value;

		const filteredFeature = cds.features.find(
			(feature) => feature.properties.CDNAME === cdSelected,
		);

		if (filteredFeature) {
			const bbox = turf.bbox(filteredFeature);
			map.fitBounds(bbox, { padding: 20 });
		}

		map.addSource("highlighted-cd", {
			type: "geojson",
			data: {
				type: "FeatureCollection",
				features: [filteredFeature],
			},
		});

		map.addLayer({
			id: "highlighted-cd",
			type: "line",
			source: "highlighted-cd",
			paint: {
				"line-color": "#333333",
				"line-width": 2,
			},
		});
	}

		// Initial map settings
		const INITIAL_CENTER = [-79.1, 44.07];
	const INITIAL_ZOOM = 5.8;

	// ... (previous variables and setup remain the same)

	// Function to reset the map
	function resetMap() {
		// Reset map position and zoom
		map.flyTo({
			center: INITIAL_CENTER,
			zoom: INITIAL_ZOOM,
			duration: 1000 // animation duration in milliseconds
		});

		// Clear selected region
		cdSelected = "";

		// Remove highlighted CD if it exists
		if (map.getLayer("highlighted-cd")) {
			map.removeLayer("highlighted-cd");
		}
		if (map.getSource("highlighted-cd")) {
			map.removeSource("highlighted-cd");
		}

		// Clear all filters
		selectedFilters.clear();
		selectedFilters = selectedFilters; // trigger reactivity
		updateMapFilter();
	}

	onMount(() => {
		map = new mapboxgl.Map({
			container: "map",
			style: "mapbox://styles/canadianurbaninstitute/clwqlh0cl011j01qefx7o5ddn?fresh=true",
			center: [-79.1, 44.07],
			zoom: 5.8,
			minZoom: 5.8,
			maxZoom: 16,
			scrollZoom: true,
			attributionControl: false,
		});

		map.addControl(new mapboxgl.NavigationControl(), "top-right");

		map.addControl(
			new mapboxgl.AttributionControl({
				customAttribution: "Canadian Urban Institute",
			}),
		);

		map.on("load", () => {
			mapLoaded = true;

			map.addSource("cds", {
				type: "geojson",
				data: cds,
			});

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
				source: "cds",
				layout: {},
				paint: {
					"fill-color": "#eee",
					"fill-opacity": 0.1,
				},
			});

			var popup = new mapboxgl.Popup({
				closeButton: false,
				closeOnClick: false,
			});

			map.on("mouseenter", "mms-business", function (e) {
				map.getCanvas().style.cursor = "pointer";
				var company = e.features[0].properties.grant_name;
				popup
					.setLngLat(e.lngLat)
					.setHTML("<h4>" + company + "</h4>")
					.addTo(map);
			});

			map.on("mouseleave", "mms-business", function () {
				map.getCanvas().style.cursor = "";
				popup.remove();
			});
		});
	});
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
		<h5>Select Region</h5>
		<div class="select-container">
			<Select
				id="select"
				items={cdnames}
				value={cdSelected}
				placeholder="Select a region"
				on:input={handleSelect}
				--background="white"
				--item-color="black"
				--item-is-active-color="black"
				--item-is-active-bg="#eee"
			/>
		</div>
		<h5>Filter projects</h5>
		<div class="checkbox-group">
			{#each equityTags as tag}
				<label class="checkbox-label">
					<input
						type="checkbox"
						checked={selectedFilters.has(tag)}
						on:change={() => handleFilterChange(tag)}
					/>
					{tag}
				</label>
			{/each}
		</div>
		<div class="legend">
			<h4>Legend</h4>
			<LegendItem
				variant={"circle"}
				label={"Business Sustainability"}
				bgcolor={"#5E32BD"}
				bordercolor={"#fff"}
			/>
		</div>
		<button class="reset-button" on:click={resetMap}>
			Reset Map
		</button>
	</div>
	<div id="map" />
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
		gap: 0.5em;
	}

	.checkbox-label {
		display: flex;
		align-items: center;
		gap: 0.5em;
		cursor: pointer;
	}

	.checkbox-label input[type="checkbox"] {
		cursor: pointer;
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

	.reset-button {
		background-color: #f0f0f0;
		border: 1px solid #ddd;
		border-radius: 4px;
		padding: 0.5em 1em;
		cursor: pointer;
		font-size: 0.9em;
		transition: all 0.2s ease;
	}

	.reset-button:hover {
		background-color: #e0e0e0;
		border-color: #ccc;
	}

	.reset-button:active {
		background-color: #d0d0d0;
		transform: translateY(1px);
	}
</style>