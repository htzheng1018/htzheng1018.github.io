
---
layout: page
title: Travel
permalink: /travel/
nav: true
nav_order: 5 
---

<!-- prettier-ignore-start -->

<!-- 3D 地球仪容器 -->
<div id="globeViz" style="width: 100%; height: 600px; cursor: grab; margin-top: 20px; border-radius: 10px; overflow: hidden;"></div>

<!-- 引入库 -->
<script src="https://unpkg.com/globe.gl"></script>

<script>
  // 获取当前页面的主题颜色 (适配暗色/亮色模式)
  const isDarkMode = document.documentElement.getAttribute('data-theme') === 'dark';
  const baseColor = isDarkMode ? 'rgba(255, 255, 255, 0.05)' : 'rgba(0, 0, 0, 0.05)';
  const strokeColor = isDarkMode ? '#444' : '#ccc';

  // 字符串标准化：转小写并去除特殊音标 (例如：Hokkaidō -> hokkaido)
  const normalizeStr = (str) => {
    return str ? str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toLowerCase() : "";
  };

  const elem = document.getElementById('globeViz');
  const globe = Globe()(elem)
    .backgroundColor('rgba(0,0,0,0)')
    .showGlobe(false) 
    .polygonAltitude(0.01);

  // 同时获取三个数据文件
  Promise.all([
    fetch('/assets/json/countries.geojson').then(r => r.json()),
    fetch('/assets/json/provinces.geojson').then(r => r.json()),
    fetch('/assets/json/visited.json').then(r => r.json())
  ]).then(([countries, provinces, visited]) => {
    
    // 预处理你读取的 visited.json
    const normalizedFootprints = visited.map(normalizeStr);
    
    // 自动修正部分非标准英文拼写以防无法匹配 (对应你的 JSON 文件)
    normalizedFootprints.push("missouri", "dubai");

    // 合并地图边界
    const allFeatures = [...countries.features, ...provinces.features];

    globe
      .polygonsData(allFeatures)
      .polygonCapColor(feat => {
        // 读取地图板块的名称
        const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
        
        // 匹配算法：互相包含或精准匹配
        const isVisited = normalizedFootprints.some(place => 
          featureName === place || 
          (place.length > 3 && featureName.includes(place)) || 
          (featureName.length > 3 && place.includes(featureName))
        );

        if (isVisited) {
          return 'rgba(220, 53, 69, 0.8)'; // 点亮为红色
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

  // 窗口自适应
  window.addEventListener('resize', () => {
    globe.width(elem.clientWidth);
  });
</script>

<div class="mt-4">
  <p>The globe above highlights the provinces and regions I have visited so far. Feel free to drag to rotate and scroll to zoom in!</p>
</div>

<!-- prettier-ignore-end -->