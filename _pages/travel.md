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
    fill: #fff;
    stroke: #ccc;
    stroke-width: 0.7px;
  }
  .visited { fill: #f06; } /* Highlight color for visited places */
  .graticule { display: none; }
  .province {
    fill: none;
    stroke: #aaa;
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

        // --- THE FIX: Create sets and handle potentially missing data ---
        const visitedProvincesSet = new Set(visited);
        const visitedCountriesSet = new Set();

        provinces.features.forEach(function(province) {
            // ONLY add the country if the 'sovereignt' property EXISTS
            if (visitedProvincesSet.has(province.properties.name) && province.properties.sovereignt) {
                visitedCountriesSet.add(province.properties.sovereignt);
            }
        });

        // Draw base land layer
        svg.insert("path", ".graticule")
            .datum(world)
            .attr("class", "land")
            .attr("d", path);

        // Filter provinces: ONLY include if 'sovereignt' EXISTS and is in our set
        const provincesToDraw = provinces.features.filter(function(d) {
            return d.properties.sovereignt && visitedCountriesSet.has(d.properties.sovereignt);
        });
        
        // Draw provinces for visited countries ONLY
        svg.selectAll(".province")
          .data(provincesToDraw)
          .enter().append("path")
          .attr("class", "province")
          .attr("d", path);
        
        // Fill visited provinces with color
        svg.selectAll(".province-visited")
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