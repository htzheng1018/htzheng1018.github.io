---
layout: globe   # Use the full-screen layout
title: Travel
nav: true
nav_order: 3
permalink: /travel/
---

<head>
  <style>
    body { margin: 0; overflow: hidden; }
    #globeViz { position: absolute; top: 0; left: 0; z-index: -1; width: 100vw; height: 100vh; } /* Ensure it covers full viewport */
    #loading-overlay {
        position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
        background: #111; color: #fff; display: flex;
        align-items: center; justify-content: center; flex-direction: column; /* Center loading text */
        font-family: sans-serif; font-size: 1.5em; z-index: 10;
        opacity: 1; /* Start visible */
        transition: opacity 0.5s ease-out; /* Smooth fade-out */
    }
    #loading-overlay.hidden { /* Class to hide the overlay */
        opacity: 0;
        pointer-events: none; /* Allow interaction with globe after hiding */
    }
  </style>
  <script src="https://unpkg.com/three@0.150.1/build/three.min.js"></script>
  <script src="https://unpkg.com/globe.gl"></script>
  <script src="https://unpkg.com/topojson-client@3"></script>
</head>

<body>
  <div id="loading-overlay"><div id="loading-text">Loading... 0%</div></div>
  <div id="globeViz"></div>

  <script>
    const setProgress = percent => {
      // Ensure percent is rounded and within 0-100
      const displayPercent = Math.max(0, Math.min(100, Math.round(percent)));
      document.getElementById('loading-text').textContent = `Loading... ${displayPercent}%`;
    };

    // Delay function from your old code
    const delay = ms => new Promise(resolve => setTimeout(resolve, ms));

    (async () => {
      setProgress(10);

      // Corrected paths using Jekyll's relative_url filter
      const visitedUrl = "{{ '/assets/travel-data/visited.json' | relative_url }}";
      const countriesUrl = "{{ '/assets/travel-data/data/countries.geojson' | relative_url }}";
      const provincesUrl = "{{ '/assets/travel-data/data/provinces.geojson' | relative_url }}";

      try {
          const [visitedRes, countriesRes, provincesRes] = await Promise.all([
            fetch(visitedUrl),
            fetch(countriesUrl),
            fetch(provincesUrl)
          ]);
          setProgress(40);

          if (!visitedRes.ok || !countriesRes.ok || !provincesRes.ok) {
              throw new Error('Failed to fetch map data');
          }

          const [visited, countries, provinces] = await Promise.all([
            visitedRes.json(),
            countriesRes.json(),
            provincesRes.json()
          ]);
          setProgress(60);

          const visitedSet = new Set(visited.map(v => v.toLowerCase()));

          // --- Reverting to your OLD logic for combining features ---
          const countriesFeatures = countries.features;
          // Filter provinces *before* combining, exactly like your old code
          const provincesFeatures = provinces.features.filter(f =>
            visitedSet.has((f.properties.name || '').toLowerCase())
          );
          const allFeatures = countriesFeatures.concat(provincesFeatures);
          // --- End of reverted logic ---

          setProgress(70);

          const globe = Globe()
            (document.getElementById('globeViz'))
            .globeImageUrl('//unpkg.com/three-globe/example/img/earth-dark.jpg')
            .polygonsData(allFeatures)
            // --- Using colors from your OLD code ---
            .polygonCapColor(f => visitedSet.has((f.properties.name || "").toLowerCase()) ? 'red' : 'white')
            .polygonSideColor(() => 'gray')
            .polygonStrokeColor(() => 'black')
            // --- Using altitude logic from your OLD code ---
            .polygonAltitude(f => visitedSet.has((f.properties.name || "").toLowerCase()) ? 0.01 : 0.005)
            // --- Using hover behavior from your OLD code ---
            .onPolygonHover(() => {}); // Empty function means no altitude change on hover

          // Wait slightly for globe initialization
          await delay(100);
          setProgress(90);

          // Hide loading overlay smoothly
          const loadingOverlay = document.getElementById('loading-overlay');
          loadingOverlay.style.opacity = 0;
          await delay(500); // Wait for fade-out transition
          loadingOverlay.style.display = 'none'; // Completely hide after fading

      } catch (error) {
          console.error("Error loading globe data:", error);
          document.getElementById('loading-text').textContent = 'Error loading map data.';
          // Keep loading overlay visible on error
      }

    })();
  </script>
</body>