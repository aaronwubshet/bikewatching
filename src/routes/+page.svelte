<script>
	import * as d3 from 'd3';
	import mapboxgl from "mapbox-gl";
	import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
	mapboxgl.accessToken = "pk.eyJ1IjoiYXd1YnNoZXQiLCJhIjoiY2x1b2tiamFpMjAzMjJqbzN6NGQ3bjdwcyJ9.ipL3t_Mw0hLMfZJCF4oW3w";
	import { onMount } from "svelte";
	
	let mapViewChanged = 0;
    $: map?.on("move", evt => mapViewChanged++);

	let map;
	let stations= [];
    let trips = [];
    let departures;
    let arrivals;

    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic)])
        .range([0, 25]);


	onMount( async() => {
		map  = new mapboxgl.Map({
			container: "map",
			style: "mapbox://styles/mapbox/streets-v12",
			zoom: 12.1,
			center: [-71.0589,42.3601],
			
		});
		await new Promise(resolve => map.on("load", resolve));
		
		map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        map.addLayer({
            id:"routes_boston", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "boston_route", // The id we specified in `addSource()`
            paint: {
                "line-color" : "green",
                "line-width" : 3,
                "line-opacity" : .4
            }
        });
        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });
        map.addLayer({
            id:"routes_cambridge", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "cambridge_route", // The id we specified in `addSource()`
            paint: {
                "line-color" : "green",
                "line-width" : 3,
                "line-opacity" : .4
            }
        });

        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv")
        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);
        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;   
            return station;
        });

	})
	
	function getCoords (station) {
		let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		let {x, y} = map.project(point);
		return {cx: x, cy: y};
	}
	
	



</script>

<style>
	@import url("$lib/global.css");
	
	#map {
		flex: 1;
	}
	#map svg {
		position: absolute;
		z-index: 1;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}
	
	circle {
        fill-opacity: .6;
        stroke: white;
		transition: 200ms;
		transform-origin: center;
		transform-box: fill-box;
		&:hover {
			transform: scale(2);

		}
        pointer-events: auto;
	}

</style>

<svelte:head>
	<title>Boston Bikers</title>
</svelte:head>

<h1>Bike Map </h1>
	

<div id="map">	
	<svg>
        {#key mapViewChanged}
            {#each stations as station }
                <circle cx={ getCoords(station).cx }
                cy={ getCoords(station).cy }
                r={radiusScale(station.totalTraffic)}
                fill="steelblue" />
            {/each}
        {/key}
	</svg>

</div>

