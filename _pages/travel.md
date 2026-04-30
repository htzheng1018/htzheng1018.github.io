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
    // .polygonAltitude(0.01);
    .polygonAltitude(0) // 设为 0，变成 2D 贴图，大大降低 GPU 负担
    .polygonResolution(3);

  // 同时获取三个数据文件
  Promise.all([
    fetch('/assets/json/countries.geojson').then(r => r.json()),
    fetch('/assets/json/provinces.geojson').then(r => r.json()),
    fetch('/assets/json/visited.json').then(r => r.json())
  ]).then(([countries, provinces, visited]) => {
    
    // 预处理你读取的 visited.json
    const normalizedFootprints = visited.map(normalizeStr);
    // normalizedFootprints.push("missouri", "dubai");

    // 【提速核心】：过滤省份数据，把没去过的省份直接剔除，不参与 3D 渲染！
    const visitedProvinces = provinces.features.filter(feat => {
      const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
      return normalizedFootprints.some(place => 
        featureName === place || 
        (place.length > 3 && featureName.includes(place)) || 
        (featureName.length > 3 && place.includes(featureName))
      );
    });

    // 现在只合并“国家底图”和“去过的省份”
    const allFeatures = [...countries.features, ...visitedProvinces];

    globe
      .polygonsData(allFeatures)
      .polygonCapColor(feat => {
        // 判断：如果这个 feature 在我们筛选出的 visitedProvinces 里，或者是国家底图中名字能匹配上的（比如国家级的 Dubai）
        const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
        const isVisited = normalizedFootprints.some(place => 
          featureName === place || 
          (place.length > 3 && featureName.includes(place)) || 
          (featureName.length > 3 && place.includes(featureName))
        );

        if (isVisited || visitedProvinces.includes(feat)) {
          return 'rgba(220, 53, 69, 0.8)'; // 点亮为红色
        }
        return baseColor; // 国家底图留白
      })
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