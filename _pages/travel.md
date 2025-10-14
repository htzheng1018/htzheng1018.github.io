---
layout: globe   # <-- 关键改动：使用我们新建的 globe.html 布局
title: Travel
nav: true
nav_order: 3
permalink: /travel/
---

<head>
  <style>
    body { margin: 0; overflow: hidden; } /* Full-screen, no scrollbars */
    #globeViz { position: absolute; top: 0; left: 0; z-index: -1; } /* Place globe behind navbar */
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
  <div id="loading-overlay">
    <div id="loading-text">Loading... 0%</div>
  </div>

  <div id="globeViz"></div>

  <script>
    const setProgress = percent => {
      document.getElementById('loading-text').textContent = `Loading... ${Math.round(percent)}%`;
    };

    (async () => {
      setProgress(10);

      // Corrected paths using Jekyll's relative_url filter
      const visitedUrl = "{{ '/assets/travel-data/visited.json' | relative_url }}";
      const countriesUrl = "{{ '/assets/travel-data/data/countries.geojson' | relative_url }}";
      const provincesUrl = "{{ '/assets/travel-data/data/provinces.geojson' | relative_url }}";

      const [visitedRes, countriesRes, provincesRes] = await Promise.all([
        fetch(visitedUrl),
        fetch(countriesUrl),
        fetch(provincesUrl)
      ]);
      setProgress(40);

      const [visited, countries, provinces] = await Promise.all([
        visitedRes.json(),
        countriesRes.json(),
        provincesRes.json()
      ]);
      setProgress(60);

      const visitedSet = new Set(visited.map(v => v.toLowerCase()));

      // Filter provinces to only include those you've visited
      const visitedProvinces = provinces.features.filter(f => 
        visitedSet.has((f.properties.name || '').toLowerCase())
      );

      // Combine all countries with ONLY the visited provinces
      const allFeatures = countries.features.concat(visitedProvinces);
      setProgress(70);

      const globe = Globe()
        (document.getElementById('globeViz'))
        .globeImageUrl('//unpkg.com/three-globe/example/img/earth-dark.jpg')
        .polygonsData(allFeatures)
        .polygonCapColor(f => visitedSet.has((f.properties.name || "").toLowerCase()) ? 'rgba(255, 0, 0, 0.7)' : 'rgba(255, 255, 255, 0.3)')
        .polygonSideColor(() => 'rgba(0, 0, 0, 0.05)')
        .polygonStrokeColor(() => '#333')
        .polygonAltitude(f => visitedSet.has((f.properties.name || "").toLowerCase()) ? 0.02 : 0.01)
        .onPolygonHover(hoverD => {
            globe.polygonAltitude(d => d === hoverD ? 0.04 : (visitedSet.has((d.properties.name || "").toLowerCase()) ? 0.02 : 0.01));
        });

      setProgress(100);

      // Fade out loading screen
      setTimeout(() => {
        document.getElementById('loading-overlay').style.transition = 'opacity 0.5s';
        document.getElementById('loading-overlay').style.opacity = 0;
        setTimeout(() => { document.getElementById('loading-overlay').style.display = 'none'; }, 500);
      }, 500);

    })();
  </script>
</body>