---
title: "Travel"
permalink: /travel/
layout: page
author_profile: true
---

# Travel

<div id="globeViz"></div>
<style>
  #globeViz {
    width: 100%;
    height: 600px;
    cursor: grab;
  }
  #globeViz:active {
    cursor: grabbing;
  }
</style>

<script src="https://cdn.jsdelivr.net/npm/globe.gl"></script>
<script>
  // Set of visited regions (ISO 3166-2 codes)
  const visited = new Set(['CN-BJ', 'CN-HB', 'US-MI', 'DE-BY', 'JP-13']);

  // Load GeoJSON data for world provinces/states boundaries (Natural Earth)
  fetch('https://d2ad6b4ur7yvpq.cloudfront.net/naturalearth-3.3.0/ne_50m_admin_1_states_provinces.geojson')
    .then(response => response.json())
    .then(data => {
      // Initialize globe
      const globe = new Globe(document.getElementById('globeViz'))
        .globeImageUrl(null)  // black base globe
        .polygonsData(data.features)
        .polygonAltitude(0.01)
        .polygonCapColor(feature => visited.has(feature.properties.iso_3166_2) ? 'red' : 'white')
        .polygonSideColor(feature => visited.has(feature.properties.iso_3166_2) ? 'rgba(255, 0, 0, 0.4)' : 'rgba(200, 200, 200, 0.15)')
        .polygonStrokeColor(() => '#000')
        .polygonLabel(({ properties: p }) => `${p.name}, ${p.admin}`);
    });
</script>
