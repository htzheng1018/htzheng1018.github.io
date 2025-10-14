---
layout: page
title: Travel
nav: true
nav_order: 3
permalink: /travel/
---

<style>
  /* Styles for the globe and visited areas */
  .water { fill: #c1dff0; }
  .land { 
    fill: #fff;           /* --- CHANGE 1: Land is now white --- */
    stroke: #ccc;         /* --- CHANGE 2: Country borders are light grey --- */
    stroke-width: 0.7px;
  }
  .visited { fill: #f06; } /* Highlight color for visited places */
  .graticule { display: none; } /* Hide the grid lines for a cleaner look */
  .province {
    fill: none;           /* Provinces are transparent by default */
    stroke: #aaa;         /* Province borders are a slightly darker grey */
    stroke-width: 0.5px;
  }
  #map {
    display: block;
    margin: 0 auto;
    max-width: 600px;
  }
  #visited_list ul {
    list-style-type: none;
    padding: 0;
    text-align: center;
    column-count: 3;
  }
</style>

<div id="map"></div>
<hr>
<div id="visited_list"></div>

<script src="https://d3js.org/d3.v3.min.js"></script>
<script src="https://d3js.org/topojson.v1.min.js"></script>

<script>
(function() {
  const width = 600, height = 600;
  const projection = d3.geo.orthographic()
      .scale(280)
      .translate([width / 2, height / 2])
      .clipAngle(90)
      .precision(.1);

  const path = d3.geo.path().projection(projection);

  const svg = d3.select("#map").append("svg")
      .attr("width", "100%")
      .attr("height", "100%")
      .attr("viewBox", `0 0 ${width} ${height}`);

  // Zoom and drag functionality
  const zoom = d3.behavior.zoom().scaleExtent([1, 8]).on("zoom", zoomed);
  const drag = d3.behavior.drag()
      .origin(function() { const r = projection.rotate(); return {x: r[0], y: -r[1]}; })
      .on("drag", function() {
          projection.rotate([d3.event.x, -d3.event.y]);
          svg.selectAll("path").attr("d", path);
      });
  
  svg.call(drag).call(zoom);

  function zoomed() {
    projection.scale(d3.event.scale * 280);
    svg.selectAll("path").attr("d", path);
  }

  // Drawing the globe sphere
  svg.append("path").datum({type: "Sphere"}).attr("class", "water").attr("d", path);

  // Load all data
  d3.json("{{ '/assets/travel-data/data/countries.geojson' | relative_url }}", function(error, world) {
    if (error) throw error;
    d3.json("{{ '/assets/travel-data/data/provinces.geojson' | relative_url }}", function(error, provinces) {
      if (error) throw error;
      d3.json("{{ '/assets/travel-data/visited.json' | relative_url }}", function(error, visited) {
        if (error) throw error;

        // --- CHANGE 3: Logic to determine visited countries ---
        const visitedProvincesSet = new Set(visited);
        const visitedCountriesSet = new Set();
        provinces.features.forEach(function(province) {
            // This assumes your provinces.geojson has a 'country' property for each province.
            if (visitedProvincesSet.has(province.properties.name)) {
                visitedCountriesSet.add(province.properties.sovereignt); // Use 'sovereignt' property from your geojson
            }
        });

        // Draw base land layer
        svg.insert("path", ".graticule")
            .datum(world)
            .attr("class", "land")
            .attr("d", path);

        // --- CHANGE 4: Filter provinces before drawing ---
        const provincesToDraw = provinces.features.filter(function(d) {
            return visitedCountriesSet.has(d.properties.sovereignt);
        });
        
        // Draw provinces for visited countries ONLY
        svg.selectAll(".province")
          .data(provincesToDraw)
          .enter().append("path")
          .attr("class", "province")
          .attr("d", path);
        
        // Fill visited provinces with color
        svg.selectAll(".province-visited") // Use a new class to avoid conflicts
          .data(provinces.features.filter(d => visitedProvincesSet.has(d.properties.name)))
          .enter().append("path")
          .attr("class", "province-visited visited")
          .attr("d", path);

        // Display list of visited places
        const listContainer = d3.select("#visited_list");
        listContainer.append("ul")
          .selectAll("li")
          .data(visited.sort())
          .enter().append("li")
          .text(d => d);
      });
    });
  });
})();
</script>