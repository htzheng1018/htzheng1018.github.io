---
layout: page
title: Travel
nav: true
nav_order: 3
permalink: /travel/
---

<style>
  .water { fill: #c1dff0; }
  .land { fill: #fff; stroke: #ccc; stroke-width: 0.7px; }
  .graticule { display: none; }
  .province {
    /* This class now ONLY defines the border style */
    stroke: #aaa;
    stroke-width: 0.5px;
  }
  #map { display: block; margin: 0 auto; max-width: 600px; }
  #visited_list ul { list-style-type: none; padding: 0; text-align: center; column-count: 3; }
</style>

<div id="map"></div>
<hr>
<div id="visited_list"></div>

<script src="https://d3js.org/d3.v3.min.js"></script>
<script src="https://d3js.org/topojson.v1.min.js"></script>

<script>
(function() {
  const width = 600, height = 600;
  const projection = d3.geo.orthographic().scale(280).translate([width / 2, height / 2]).clipAngle(90).precision(.1);
  const path = d3.geo.path().projection(projection);
  const svg = d3.select("#map").append("svg").attr("width", "100%").attr("height", "100%").attr("viewBox", `0 0 ${width} ${height}`);

  const zoom = d3.behavior.zoom().scaleExtent([1, 8]).on("zoom", zoomed);
  const drag = d3.behavior.drag()
      .origin(function() { const r = projection.rotate(); return {x: r[0], y: -r[1]}; })
      .on("drag", function() { projection.rotate([d3.event.x, -d3.event.y]); svg.selectAll("path").attr("d", path); });

  svg.call(drag).call(zoom);
  function zoomed() { projection.scale(d3.event.scale * 280); svg.selectAll("path").attr("d", path); }

  svg.append("path").datum({type: "Sphere"}).attr("class", "water").attr("d", path);

  d3.json("{{ '/assets/travel-data/data/countries.geojson' | relative_url }}", function(error, world) {
    if (error) throw error;
    d3.json("{{ '/assets/travel-data/data/provinces.geojson' | relative_url }}", function(error, provinces) {
      if (error) throw error;
      d3.json("{{ '/assets/travel-data/visited.json' | relative_url }}", function(error, visited) {
        if (error) throw error;

        const visitedProvincesSet = new Set(visited);
        const visitedCountriesSet = new Set();
        provinces.features.forEach(function(p) {
          if (visitedProvincesSet.has(p.properties.name) && p.properties.sovereignt) {
            visitedCountriesSet.add(p.properties.sovereignt);
          }
        });

        svg.insert("path", ".graticule").datum(world).attr("class", "land").attr("d", path);

        const provincesToDraw = provinces.features.filter(d => d.properties.sovereignt && visitedCountriesSet.has(d.properties.sovereignt));

        // --- THE FINAL FIX: Set styles directly in code ---
        svg.selectAll(".province")
          .data(provincesToDraw)
          .enter().append("path")
          .attr("class", "province") // All paths get the 'province' class for the border
          .style("fill", function(d) {
            // Apply fill color directly.
            // If visited, return red. If not, return transparent ('none').
            return visitedProvincesSet.has(d.properties.name) ? "#f06" : "none";
          })
          .attr("d", path);

        const listContainer = d3.select("#visited_list");
        listContainer.append("ul").selectAll("li").data(visited.sort()).enter().append("li").text(d => d);
      });
    });
  });
})();
</script>