---
layout: globe
title: Travel
nav: true
nav_order: 3
permalink: /travel/
---

<head>
  <style>
    body { margin: 0; overflow: hidden; }
    #globeViz { position: absolute; top: 0; left: 0; z-index: -1; }
    #loading-overlay {
        position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
        background: #111; color: #fff; display: flex;
        align-items: center; justify-content: center;
        font-family: sans-serif; font-size: 1.5em; z-index: 10;
    }
  </style>
  <script src="https://unpkg.com/three@0.150.1/build/three.min.js"></script>
  <script src="https://unpkg.com/globe.gl"></script>
</head>

<body>
  <div id="loading-overlay"><div id="loading-text">Loading...</div></div>
  <div id="globeViz"></div>

  <script>
    (async () => {
      const visitedUrl = "{{ '/assets/travel-data/visited.json' | relative_url }}";
      const countriesUrl = "{{ '/assets/travel-data/data/countries.geojson' | relative_url }}";
      const provincesUrl = "{{ '/assets.travel-data/data/provinces.geojson' | relative_url }}";

      const [visited, countries, provinces] = await Promise.all([
        fetch(visitedUrl).then(res => res.json()),
        fetch(countriesUrl).then(res => res.json()),
        fetch(provincesUrl).then(res => res.json())
      ]);

      const visitedSet = new Set(visited.map(v => v.toLowerCase()));

      // --- THE FINAL, SIMPLE FIX: Combine ALL data without filtering ---
      const allFeatures = countries.features.concat(provinces.features);

      const globe = Globe()
        (document.getElementById('globeViz'))
        .globeImageUrl('//unpkg.com/three-globe/example/img/earth-dark.jpg')
        .polygonsData(allFeatures)
        .polygonCapColor(f => visitedSet.has((f.properties.name || "").toLowerCase()) ? 'rgba(255, 0, 0, 0.7)' : 'rgba(255, 255, 255, 0.3)')
        .polygonSideColor(() => 'rgba(0, 0, 0, 0.05)')
        .polygonStrokeColor(() => '#333')
        .polygonAltitude(f => visitedSet.has((f.properties.name || "").toLowerCase()) ? 0.02 : 0.01);

      // Fade out loading screen
      document.getElementById('loading-overlay').style.transition = 'opacity 0.5s';
      document.getElementById('loading-overlay').style.opacity = 0;
      setTimeout(() => { document.getElementById('loading-overlay').style.display = 'none'; }, 500);

    })();
  </script>
</body>