---
layout: default
title: Travel
nav: true
nav_order: 5 
permalink: /travel/
---

<style>
  /* 强制内容区域占满屏幕，覆盖主题的 padding */
  .post-container {
    max-width: 100% !important;
    padding: 0 !important;
    margin: 0 !important;
  }
  /* 隐藏这个页面的页脚 */
  .footer {
    display: none;
  }
  /* 让地球仪占满屏幕 
    我们减去 56px 是为了给顶部的导航栏留出空间
  */
  #globeViz {
    width: 100vw; 
    height: calc(100vh - 56px); 
    margin-top: 56px; /* 为固定的导航栏留出顶部空间 */
  }
  /* 确保在加载时，页面本身不会有滚动条 */
  html, body {
    overflow: hidden;
  }
</style>

<div id="loading-overlay" style="
  position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
  background: #111; color: #fff; display: flex;
  align-items: center; justify-content: center; flex-direction: column;
  font-family: sans-serif; font-size: 1.5em; z-index: 10;">
  <div id="loading-text">Loading... 0%</div>
</div>

<div id="globeViz" style="display: none;"></div>

<script src="https://unpkg.com/three@0.150.1/build/three.min.js"></script>
<script src="https://unpkg.com/globe.gl"></script>
<script src="https://unpkg.com/topojson-client@3"></script>

<script>
  const setProgress = percent => {
    document.getElementById('loading-text').textContent = `Loading... ${percent}%`;
  };

  const delay = ms => new Promise(resolve => setTimeout(resolve, ms));

  const run = async () => {
    setProgress(10);

    const [visitedRes, countriesRes, provincesRes] = await Promise.all([
      // 关键修改：使用 Jekyll 的正确方式获取 assets 里的文件
      fetch('{{ "/assets/travel/visited.json" | relative_url }}'),
      fetch('{{ "/assets/travel/data/countries.geojson" | relative_url }}'),
      fetch('{{ "/assets/travel/data/provinces.geojson" | relative_url }}')
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
