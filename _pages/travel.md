---
layout: travel # <-- 关键改动：使用我们新建的 travel.html 布局
title: Travel
nav: true
nav_order: 3
permalink: /travel/
---

<style>
  body { overflow: hidden; } /* Hide scrollbars for a full-screen feel */
  .water { fill: #c1dff0; }
  .land { fill: #d3d3d3; stroke: #fff; }
  .visited { fill: #f06; }
  .graticule { fill: none; stroke: #777; stroke-width: .5px; stroke-opacity: .5; }
  #map-container {
    width: 100vw;
    height: calc(100vh - 56px); /* Full height minus navbar height */
    display: flex;
    justify-content: center;
    align-items: center;
  }
  #map svg {
    width: 100%;
    height: 100%;
  }
  #visited_list { display: none; } /* We hide the list for the full-screen globe */
</style>

<div id="map-container">
  <div id="map"></div>
</div>

<script src="https://d3js.org/d3.v3.min.js"></script>
<script src="https://d3js.org/topojson.v1.min.js"></script>

<script>
(function() {
  const container = document.getElementById('map-container');
  const width = container.offsetWidth;
  const height = container.offsetHeight;
  const scale = Math.min(width, height) / 2 * 0.9;

  const projection = d3.geo.orthographic()
      .scale(scale)
      .translate([width / 2, height / 2])
      .clipAngle(90)
      .precision(.1);

  const path = d3.geo.path().projection(projection);

  const svg = d3.select("#map").append("svg")
      .attr("width", width)
      .attr("height", height);

  // Zoom and drag functionality
  const zoom = d3.behavior.zoom()
      .scaleExtent([1, 8])
      .on("zoom", function() {
          const newScale = d3.event.scale * scale;
          projection.scale(newScale);
          svg.selectAll("path").attr("d", path);
      });

  const drag = d3.behavior.drag()
      .origin(function() { const r = projection.rotate(); return {x: r[0], y: -r[1]}; })
      .on("drag", function() {
          projection.rotate([d3.event.x, -d3.event.y]);
          svg.selectAll("path").attr("d", path);
      });

  svg.call(drag).call(zoom);

  // Drawing the globe
  svg.append("path").datum({type: "Sphere"}).attr("class", "water").attr("d", path);

  // Load data and draw countries/provinces
  d3.json("{{ '/assets/travel-data/data/countries.geojson' | relative_url }}", function(error, world) {
    if (error) throw error;
    svg.insert("path", ".graticule").datum(world).attr("class", "land").attr("d", path);

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
             if (visited.includes(d.properties.name)) { return "#f06"; }
          });
      });
    });
  });
})();
</script>