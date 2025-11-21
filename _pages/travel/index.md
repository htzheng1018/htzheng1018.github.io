---
title: Travel
nav: true
nav_order: 5 
layout: null
permalink: /travel/
---

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Travel Globe</title>
  <style>
    html, body { margin: 0; height: 100%; overflow: hidden; }
    #globeViz { width: 100vw; height: 100vh; }
  </style>\n  <script src="https://unpkg.com/three@0.150.1/build/three.min.js"></script>
  <script src="https://unpkg.com/globe.gl"></script>
  <script src="https://unpkg.com/topojson-client@3"></script>

  <!-- Core theme CSS -->
  <link type="text/css" href="../static/css/styles.css" rel="stylesheet" />
  <link type="text/css" href="../static/css/main.css" rel="stylesheet" />
  <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.5.0/font/bootstrap-icons.css" rel="stylesheet" />
</head>
<body id="page-top">
  <!-- Navigation bar -->
  <nav class="header navbar navbar-expand-lg navbar-light fixed-top shadow-sm" id="mainNav">
    <div class="container px-5">
      <a class="navbar-brand fw-bold" href="../index.html">Haotian Zheng</a>
      <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarResponsive"
        aria-controls="navbarResponsive" aria-expanded="false" aria-label="Toggle navigation">
        MENU <i class="bi-list"></i>
      </button>
      <div class="collapse navbar-collapse" id="navbarResponsive">
        <ul class="navbar-nav ms-auto me-4 my-3 my-lg-0">
          <li class="nav-item">
            <a class="nav-link me-lg-3" href="../">HOME</a>
          </li>
          <li class="nav-item">
            <a class="nav-link me-lg-3" href="../publications/">PAPERS</a>
          </li>
          <li class="nav-item">
            <a class="nav-link me-lg-3" href="/travel/">TRAVEL</a>
          </li>
        </ul>
      </div>
    </div>
  </nav>

  <!-- Loading overlay -->
  <div id="loading-overlay" style="
    position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
    background: #111; color: #fff; display: flex;
    align-items: center; justify-content: center; flex-direction: column;
    font-family: sans-serif; font-size: 1.5em; z-index: 10;">
    <div id="loading-text">Loading... 0%</div>
  </div>

  <div id="globeViz" style="display: none;"></div>

  <script>
    const setProgress = percent => {
      document.getElementById('loading-text').textContent = `Loading... ${percent}%`;
    };

    const delay = ms => new Promise(resolve => setTimeout(resolve, ms));

    const run = async () => {
      setProgress(10);

      const [visitedRes, countriesRes, provincesRes] = await Promise.all([
        fetch('visited.json'),
        fetch('data/countries.geojson'),
        fetch('data/provinces.geojson')
      ]);
      setProgress(40);

      const [visited, countries, provinces] = await Promise.all([
        visitedRes.json(),
        countriesRes.json(),
        provincesRes.json()
      ]);
      setProgress(60);

      const visitedSet = new Set(visited.map(v => v.toLowerCase()));
      const countriesFeatures = countries.features;
      const provincesFeatures = provinces.features.filter(f =>
        visitedSet.has((f.properties.name || '').toLowerCase())
      );

      const allFeatures = countriesFeatures.concat(provincesFeatures);
      setProgress(70);

      const globe = Globe()
        .globeImageUrl('//unpkg.com/three-globe/example/img/earth-dark.jpg')
        .polygonsData(allFeatures)
        .polygonCapColor(f => visitedSet.has((f.properties.name || "").toLowerCase()) ? 'red' : 'white')
        .polygonSideColor(() => 'gray')
        .polygonStrokeColor(() => 'black')
        .polygonAltitude(f => visitedSet.has((f.properties.name || "").toLowerCase()) ? 0.01 : 0.005)
        .onPolygonHover(() => {})

      globe(document.getElementById('globeViz'));
      setProgress(90);

      const waitForFirstFrame = () => new Promise(resolve => {
        const canvas = document.querySelector('#globeViz canvas');
        const check = () => {
          if (!canvas) return resolve();
          const ctx = canvas.getContext('2d');
          if (!ctx) return resolve();
          const pixels = ctx.getImageData(0, 0, 1, 1).data;
          const nonTransparent = pixels[3] > 0;
          if (nonTransparent) return resolve();
          requestAnimationFrame(check);
        };
        requestAnimationFrame(check);
      });

      await waitForFirstFrame();
      setProgress(100);
      await delay(300);

      document.getElementById('loading-overlay').style.display = 'none';
      document.getElementById('globeViz').style.display = 'block';
    };

    run();
  </script>
</body>
</html>
