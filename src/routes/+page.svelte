<script>
  import mapboxgl from "mapbox-gl";
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  import { onMount } from "svelte";
  import * as d3 from "d3";
  //   import { MAPBOX_ACCESS_TOKEN } from "$env/static/private";
  const MAPBOX_ACCESS_TOKEN = import.meta.env.VITE_MAPBOX_ACCESS_TOKEN;
  mapboxgl.accessToken = MAPBOX_ACCESS_TOKEN;

  let map;
  let stations = [];
  let departures = [];
  let arrivals = [];
  let totalTraffic = [];
  let mapViewChanged = 0;
  let radiusScale;
  let timeFilter = -1;
  let filteredTrips = [];
  let filteredArrivals = [];
  let filteredDepartures = [];
  let filteredStations = [];
  let trips = [];
  let selectedStation = null;
  const departuresByMinute = Array.from({ length: 1440 }, () => []);
  const arrivalsByMinute = Array.from({ length: 1440 }, () => []);

  function geoJSONPolygonToPath(feature) {
    const path = d3.path();
    const rings = feature.geometry.coordinates;

    for (const ring of rings) {
      for (let i = 0; i < ring.length; i++) {
        const [lng, lat] = ring[i];
        const { x, y } = map.project([lng, lat]);
        if (i === 0) path.moveTo(x, y);
        else path.lineTo(x, y);
      }
      path.closePath();
    }
    return path.toString();
  }
  let stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);
  onMount(() => {
    loadData();
    initMap();
    // getIso(-71.09415, 42.36027);
    // console.log(isochrone);
  });

  function filterByMinute(tripsByMinute, minute) {
    // Normalize both to the [0, 1439] range
    // % is the remainder operator: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder
    let minMinute = (minute - 60 + 1440) % 1440;
    let maxMinute = (minute + 60) % 1440;

    if (minMinute > maxMinute) {
      let beforeMidnight = tripsByMinute.slice(minMinute);
      let afterMidnight = tripsByMinute.slice(0, maxMinute);
      return beforeMidnight.concat(afterMidnight).flat();
    } else {
      return tripsByMinute.slice(minMinute, maxMinute).flat();
    }
  }

  function minutesSinceMidnight(date) {
    return date.getHours() * 60 + date.getMinutes();
  }
  function getCoords(station) {
    let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
    let { x, y } = map.project(point);
    return { cx: x, cy: y };
  }

  async function loadData() {
    stations = await d3.csv(
      "https://vis-society.github.io/labs/8/data/bluebikes-stations.csv"
    );
    trips = await d3
      .csv(
        "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv"
      )
      .then((trips) => {
        for (let trip of trips) {
          trip.started_at = new Date(trip.started_at);
          trip.ended_at = new Date(trip.ended_at);
          let startedMinutes = minutesSinceMidnight(trip.started_at);
          departuresByMinute[startedMinutes].push(trip);
          let endedMinutes = minutesSinceMidnight(trip.ended_at);
          arrivalsByMinute[endedMinutes].push(trip);
        }
        return trips;
      });
    departures = d3.rollup(
      trips,
      (v) => v.length,
      (d) => d.start_station_id
    );
    arrivals = d3.rollup(
      trips,
      (v) => v.length,
      (d) => d.end_station_id
    );
    totalTraffic = new Map();
    for (const [stationId, count] of departures) {
      totalTraffic.set(stationId, count + (arrivals.get(stationId) || 0));
    }

    stations = stations.map((station) => {
      let id = station.Number;
      station.arrivals = arrivals.get(id) ?? 0;
      station.departures = departures.get(id) ?? 0;
      station.totalTraffic = totalTraffic.get(id) ?? 0;
      return station;
    });
    // console.log(stations);

    // TODO: Same for arrivals
  }

  async function initMap() {
    map = new mapboxgl.Map({
      container: "map",
      style: "mapbox://styles/mapbox/streets-v12",
      center: [-71.09415, 42.36027],
      zoom: 12,
    });
    await new Promise((resolve) => map.on("load", resolve));
    map.addSource("boston_route", {
      type: "geojson",
      data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
    });
    map.addLayer({
      id: "bike_lanes", // A name for our layer (up to you)
      type: "line", // one of the supported layer types, e.g. line, circle, etc.
      source: "boston_route", // The id we specified in `addSource()`
      paint: {
        "line-color": "#0000FF",
        "line-width": 2,
        "line-opacity": 0.4,
      },
    });
    map.addSource("cambridge_route", {
      type: "geojson",
      data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
    });
    map.addLayer({
      id: "bike_lanes2", // A name for our layer (up to you)
      type: "line", // one of the supported layer types, e.g. line, circle, etc.
      source: "cambridge_route", // The id we specified in `addSource()`
      paint: {
        "line-color": "#0000FF",
        "line-width": 2,
        "line-opacity": 0.4,
      },
    });
  }
  $: map?.on("move", (evt) => mapViewChanged++);
  $: radiusScale = d3
    .scaleSqrt()
    .domain([0, d3.max(stations, (d) => d.totalTraffic) || 0])
    .range(timeFilter === -1 ? [0, 25] : [3, 30]);
  $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {
    timeStyle: "short",
  });
  $: filteredDepartures = d3.rollup(
    filterByMinute(departuresByMinute, timeFilter),
    (v) => v.length,
    (d) => d.start_station_id
  );
  $: filteredArrivals = d3.rollup(
    filterByMinute(arrivalsByMinute, timeFilter),
    (v) => v.length,
    (d) => d.end_station_id
  );

  $: filteredStations = stations.map((station) => {
    const id = station.Number;
    const arr = filteredArrivals.get(id) ?? 0;
    const dep = filteredDepartures.get(id) ?? 0;
    return {
      ...station,
      arrivals: arr,
      departures: dep,
      totalTraffic: arr + dep,
    };
  });

  const urlBase = "https://api.mapbox.com/isochrone/v1/mapbox/";
  const profile = "cycling";
  const minutes = [5, 10, 15, 20];
  const contourColors = ["03045e", "0077b6", "00b4d8", "90e0ef"];
  let isochrone = null;

  async function getIso(lon, lat) {
    const base = `${urlBase}${profile}/${lon},${lat}`;
    const params = new URLSearchParams({
      contours_minutes: minutes.join(","),
      contours_colors: contourColors.join(","),
      polygons: "true",
      access_token: mapboxgl.accessToken,
    });
    const url = `${base}?${params.toString()}`;

    const query = await fetch(url, { method: "GET" });
    isochrone = await query.json();
    console.log(isochrone);
  }
  $: if (selectedStation) {
    getIso(+selectedStation.Long, +selectedStation.Lat);
  } else {
    isochrone = null;
  }
</script>

<header>
  <h1>Welcome to Bikewatcher</h1>
  <p>We watch bikes here</p>
  <label>
    Filter by time
    <input type="range" min="-1" max="1440" bind:value={timeFilter} />
    {#if timeFilter !== -1}
      <time style="display: block">{timeFilterLabel}</time>
    {:else}
      <em style="display: block">(any time)</em>
    {/if}
  </label>
</header>
<div id="map">
  <svg>
    {#key mapViewChanged}
      {#if isochrone}
        {#each isochrone.features as feature}
          <path
            d={geoJSONPolygonToPath(feature)}
            fill={feature.properties.fillColor}
            fill-opacity="0.2"
            stroke="#000000"
            stroke-opacity="0.5"
            stroke-width="1"
          >
            <title>{feature.properties.contour} minutes of biking</title>
          </path>
        {/each}
      {/if}
      {#each timeFilter !== -1 ? filteredStations : stations as station}
        <circle
          {...getCoords(station)}
          r={radiusScale(station.totalTraffic)}
          fill="steelblue"
          style="--departure-ratio: {stationFlow(
            station.departures / station.totalTraffic
          )}"
          class={station?.Number === selectedStation?.Number ? "selected" : ""}
          on:mousedown={() =>
            (selectedStation =
              selectedStation?.Number !== station?.Number ? station : null)}
        >
          <title>
            {station.totalTraffic} trips ({station.departures} departures, {station.arrivals}
            arrivals)
          </title>
        </circle>
      {/each}
    {/key}
  </svg>
</div>
<div class="legend">
  <div style="--departure-ratio: 1; text-align: left">More departures</div>
  <div style="--departure-ratio: 0.5; text-align: center">Balanced</div>
  <div style="--departure-ratio: 0; text-align: right">More arrivals</div>
</div>

<style>
  @import url("$lib/global.css");
  #map circle,
  .legend > div {
    --color-departures: steelblue;
    --color-arrivals: darkorange;
    --color: color-mix(
      in oklch,
      var(--color-departures) calc(100% * var(--departure-ratio)),
      var(--color-arrivals)
    );
  }

  .legend {
    /* flex: 1; */
    margin-block: 1rem;
    gap: 1px;
    display: flex;
    flex-direction: row;
  }

  .legend > div {
    flex: 1;
    color: white;
    background-color: var(--color);
    padding-inline: 1rem;
    padding-block: 0.5rem;
    font-weight: bold;
  }

  #map {
    flex: 1;
    background-color: #f0f0f0;
  }
  #map svg {
    /* background: yellow;
    opacity: 50%; */
    position: absolute;
    z-index: 1;
    width: 100%;
    height: 100%;
    pointer-events: none;

    &:has(circle.selected) circle:not(.selected) {
      opacity: 0.3;
    }
  }
  #map svg circle {
    fill: steelblue;
    fill-opacity: 0.6;
    stroke: white;
    fill: var(--color);
    transition: opacity 0.2s ease;
    pointer-events: auto;
  }
  header {
    display: flex;
    gap: 1em;
    align-items: baseline;
  }
  header label {
    margin-left: auto;
    display: flex;
    flex-direction: column;
    gap: 0.25em;
  }

  time,
  em {
    display: block;
    min-height: 1.2em; /* Prevents layout shift */
  }

  em {
    font-style: italic;
    color: #666;
  }

  #map svg path {
    pointer-events: auto; /* BUT this re-enables hover/click on isochrone paths */
    /* cursor: help; */
  }
</style>
