
---
layout: page
title: Travel
permalink: /travel/
nav: true
nav_order: 5 
---

<div id="globeViz" style="width: 100%; height: 600px; cursor: grab; margin-top: 20px; border-radius: 10px; overflow: hidden;"></div>

<script src="//unpkg.com/globe.gl"></script>

<script>
  // 获取当前页面的主题颜色
  const isDarkMode = document.documentElement.getAttribute('data-theme') === 'dark';
  const baseColor = isDarkMode ? 'rgba(255, 255, 255, 0.05)' : 'rgba(0, 0, 0, 0.05)';
  const strokeColor = isDarkMode ? '#444' : '#ccc';

  // 你的专属足迹清单
  const myFootprints = [
    "Beijing", "Tianjin"
  ];

  // 字符串标准化函数：转小写并去除音标符号 (例如：Hokkaidō -> hokkaido, Zürich -> zurich)
  const normalizeStr = (str) => {
    return str ? str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toLowerCase() : "";
  };

  // 预处理足迹清单
  const normalizedFootprints = myFootprints.map(normalizeStr);

  const elem = document.getElementById('globeViz');
  const globe = Globe()(elem)
    .backgroundColor('rgba(0,0,0,0)')
    .showGlobe(false) 
    .polygonAltitude(0.01);

  // 加载 GeoJSON 数据
  Promise.all([
    fetch('/assets/json/countries.geojson').then(r => r.json()),
    fetch('/assets/json/provinces.geojson').then(r => r.json())
  ]).then(([countries, provinces]) => {
    
    const allFeatures = [...countries.features, ...provinces.features];

    globe
      .polygonsData(allFeatures)
      .polygonCapColor(feat => {
        // 获取 GeoJSON 中该区块的名字（通常存在 name 或 NAME 属性里）
        const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
        
        // 匹配逻辑：如果 GeoJSON 的名字包含了你的足迹名字，或者你的足迹名字包含了它
        const isVisited = normalizedFootprints.some(place => 
          featureName === place || 
          (place.length > 3 && featureName.includes(place)) || // 避免 DC 这种过短的字符引起误匹配
          (featureName.length > 3 && place.includes(featureName))
        );

        if (isVisited) {
          return 'rgba(220, 53, 69, 0.8)'; // 点亮去过的地方
        }
        return baseColor; // 未去过的地方留白
      })
      .polygonSideColor(() => 'rgba(0, 0, 0, 0.05)')
      .polygonStrokeColor(() => strokeColor)
      .polygonLabel(feat => {
        const name = feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || 'Unknown';
        return `
          <div style="background: rgba(0, 0, 0, 0.75); color: white; padding: 6px 10px; border-radius: 4px; font-size: 14px;">
            <b>${name}</b>
          </div>
        `;
      });

    globe.controls().autoRotate = true;
    globe.controls().autoRotateSpeed = 1.0;
    globe.controls().enableZoom = true;
  });

  window.addEventListener('resize', () => {
    globe.width(elem.clientWidth);
  });
</script>

<div class="mt-4">
  <p>The globe above highlights the provinces and regions I have visited so far. Feel free to drag to rotate and scroll to zoom in!</p>
</div>