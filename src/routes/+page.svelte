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
    let filterTime = -1;
    let departuresFiltered;
    let arrivalsFiltered;
    let stationsFiltered = [];
    let departuresByMinute = Array.from({length: 1440}, () => []);
    let arrivalsByMinute = Array.from({length: 1440}, () => []);

    let stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);


    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stationsFiltered, d => d.totalTraffic)])
        .range([0, 25]);
    
        function getCoords (station) {
		let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		let {x, y} = map.project(point);
		return {cx: x, cy: y};
	}
	
    $: timeFilterLabel = new Date(0, 0, 0, 0, filterTime)
                     .toLocaleString("en", {timeStyle: "short"});

    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    $: filteredTrips = filterTime === -1? trips : filterByMinute(departuresByMinute, filterTime);

    function filterByMinute (tripsByMinute, minute) {
	// Normalize both to the [0, 1439] range
	// % is the remainder operator: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder
	let minMinute = (minute - 60 + 1440) % 1440;
	let maxMinute = minute + 60 % 1440;

	if (minMinute > maxMinute) {
		let beforeMidnight = tripsByMinute.slice(minMinute);
		let afterMidnight = tripsByMinute.slice(0, maxMinute);
		return beforeMidnight.concat(afterMidnight).flat();
	}
	else {
		return tripsByMinute.slice(minMinute, maxMinute).flat();
	}
}

    // trips.filter(trip => {
    //     let startedMinutes = minutesSinceMidnight(trip.started_at);
    //     let endedMinutes = minutesSinceMidnight(trip.ended_at);
    //     return Math.abs(startedMinutes - filterTime) <= 60
    //         || Math.abs(endedMinutes - filterTime) <= 60;
    // });

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

        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at)
                trip.ended_at = new Date(trip.ended_at)
                let startedMinutes = minutesSinceMidnight(trip.started_at);
                departuresByMinute[startedMinutes].push(trip);
                let endedMinutes = minutesSinceMidnight(trip.ended_at);
                arrivalsByMinute[endedMinutes].push(trip);
            }
            return trips;
            
        });
        

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

 
    $:{
        departuresFiltered = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
        arrivalsFiltered = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);
        stationsFiltered = stations.map(station => {
            let stationCopy = {...station};
            let id = stationCopy.Number;
            stationCopy.arrivals = arrivalsFiltered.get(id) ?? 0;
            stationCopy.departures = departuresFiltered.get(id) ?? 0;
            stationCopy.totalTraffic = stationCopy.arrivals + stationCopy.departures;   
            return stationCopy;
        });
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
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);
	}

    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
    }
    label {
        margin-left: auto;
    }
    time, em {
        display: block;
    }
    em {
        color: #888;
        font-style: italic;
    }

    /* .legend{
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);
        display: flex;
        align-items: center;
       }   margin-block: 20px; /* Adjust this value as needed */
     

    .legend {
        display: flex;
        gap: 1px;
        margin-block: 20px; /* Adjust this value as needed */
    }

    .legend div {
        flex: 1;
        padding: 5px 20px; /* Adjust these values as needed */
    }

    .legend .departures {
        --color: steelblue;
        background-color: var(--color);
        color: white; /* Adjust this value as needed for contrast */
        text-align: left;
    }

    .legend .arrivals {
        --color: darkorange;
        background-color: var(--color);
        color: black; /* Adjust this value as needed for contrast */
        text-align: right;
    }

    .legend .mix {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        background-color: var(--color);
        color: black; /* Adjust this value as needed for contrast */
        text-align: center;
    }
</style>

<svelte:head>
	<title>Boston Bikers</title>
</svelte:head>

<header>
    <h1>Bike Map</h1>
    <label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={filterTime} />
        {#if filterTime === -1}
            <em>(any time)</em>
        {:else}
            <time>{timeFilterLabel}</time>
        {/if}
    </label>
</header>
<div id="map">	
	<svg>
        {#key mapViewChanged}
            {#each stationsFiltered as station }
                <circle cx={ getCoords(station).cx }
                cy={ getCoords(station).cy }
                r={radiusScale(station.totalTraffic)}
                fill="steelblue" 
                style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }"
                />
            {/each}
        {/key}
	</svg>

</div>

<div class="legend">
    <div class="departures">Departures</div>
    <div class="mix">Mix</div>
    <div class="arrivals">Arrivals</div>
</div>
