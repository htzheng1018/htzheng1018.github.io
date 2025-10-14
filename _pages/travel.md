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
  .land { fill: #d3d3d3; stroke: #fff; }
  .visited { fill: #f06; } /* Highlight color for visited places */
  .graticule { fill: none; stroke: #777; stroke-width: .5px; stroke-opacity: .5; }
  #map {
    display: block;
    margin: 0 auto;
    max-width: 600px; /* Adjust size as needed */
  }
  #visited_list ul {
    list-style-type: none;
    padding: 0;
    text-align: center;
    column-count: 3; /* Adjust number of columns as needed */
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
  const graticule = d3.geo.graticule();

  const svg = d3.select("#map").append("svg")
      .attr("width", "100%")
      .attr("height", "100%")
      .attr("viewBox", `0 0 ${width} ${height}`);

  // Zoom and drag functionality
  const zoom = d3.behavior.zoom()
      .scaleExtent([1, 8])
      .on("zoom", zoomed);

  svg.call(zoom).call(zoom.event);

  function zoomed() {
    projection.scale(zoom.scale() * 280);
    svg.selectAll("path").attr("d", path);
  }
  
  const drag = d3.behavior.drag()
      .origin(function() { 
          const r = projection.rotate(); 
          return {x: r[0], y: -r[1]}; 
      })
      .on("drag", function() {
          projection.rotate([d3.event.x, -d3.event.y]);
          svg.selectAll("path").attr("d", path);
      });

  svg.call(drag);

  // Drawing the globe
  svg.append("path")
      .datum({type: "Sphere"})
      .attr("class", "water")
      .attr("d", path);

  svg.append("path")
      .datum(graticule)
      .attr("class", "graticule")
      .attr("d", path);

  // Load data and draw countries/provinces
  d3.json("{{ '/assets/travel-data/data/countries.geojson' | relative_url }}", function(error, world) {
    if (error) throw error;
    svg.insert("path", ".graticule")
        .datum(world)
        .attr("class", "land")
        .attr("d", path);

    d3.json("{{ '/assets/travel-data/data/provinces.geojson' | relative_url }}", function(error, provinces) {
      if (error) throw error;
      d3.json("{{ '/assets/travel-data/visited.json' | relative_url }}", function(error, visited) {
        if (error) throw error;
        
        svg.selectAll(".province")
          .data(provinces.features)
          .enter().append("path")
          .attr("class", d => visited.includes(d.properties.name) ? "province visited" : "province")
          .attr("d", path)
          .style("fill", function(d) {
             if (visited.includes(d.properties.name)) {
                return "#f06"; // Visited color
             }
          });

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