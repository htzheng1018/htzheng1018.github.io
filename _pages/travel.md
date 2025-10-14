---
layout: page
title: Travel
nav: true
nav_order: 3
permalink: /travel/
---

<style>
  .visited {
    fill: #f06; /* This is the highlight color for visited places */
  }
  #map {
    background: #fff;
    border: 1px solid #ccc;
    display: block;
    margin: 0 auto;
    max-width: 900px; /* Limit map width on large screens */
  }
  #visited_list ul {
    list-style-type: none;
    padding: 0;
    text-align: center;
    column-count: 3; /* Display list in 3 columns */
  }
</style>

<div id="map"></div>
<hr>
<div id="visited_list"></div>

<script src="https://d3js.org/d3.v5.min.js"></script>
<script src="https://unpkg.com/d3-geo-projection@2"></script>
<script src="https://d3js.org/topojson.v2.min.js"></script>

<script>
  (async () => {
    const width = 900, height = 500;
    const projection = d3.geoMercator().scale(140).translate([width / 2, height / 1.6]);
    const path = d3.geoPath().projection(projection);

    const svg = d3.select("#map").append("svg")
      .attr("width", "100%")
      .attr("height", "100%")
      .attr("viewBox", `0 0 ${width} ${height}`);

    const countries = await d3.json("{{ '/assets/travel-data/data/countries.geojson' | relative_url }}");
    const provinces = await d3.json("{{ '/assets/travel-data/data/provinces.geojson' | relative_url }}");
    const visited = await d3.json("{{ '/assets/travel-data/visited.json' | relative_url }}");

    svg.selectAll(".country")
      .data(countries.features)
      .enter().append("path")
      .attr("class", "country")
      .attr("d", path);

    svg.selectAll(".province")
      .data(provinces.features)
      .enter().append("path")
      .attr("class", d => visited.includes(d.properties.name) ? "province visited" : "province")
      .attr("d", path);

    const listContainer = d3.select("#visited_list");
    listContainer.append("ul")
      .selectAll("li")
      .data(visited.sort())
      .enter().append("li")
      .text(d => d);
  })();
</script>
