---
layout: page
title: Travel
permalink: /travel/
nav: true
nav_order: 5 
---

<!-- prettier-ignore-start -->

<!-- 增加了一个相对定位的外层容器，用来放置 Loading 提示和地球仪 -->
<div style="position: relative; width: 100%; height: 600px; margin-top: 20px;">
  
  <!-- Loading 提示框 -->
  <div id="loading" style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); font-weight: bold; font-size: 1.2em; color: #888; z-index: 10;">
    🌍 Downloading Map Data (approx. 15MB), please wait...
  </div>

  <!-- 3D 地球仪容器 -->
  <div id="globeViz" style="width: 100%; height: 100%; cursor: grab; border-radius: 10px; overflow: hidden;"></div>

</div>

<!-- 引入库 -->
<script src="https://unpkg.com/globe.gl"></script>

<script>
  const isDarkMode = document.documentElement.getAttribute('data-theme') === 'dark';
  const baseColor = isDarkMode ? 'rgba(255, 255, 255, 0.05)' : 'rgba(0, 0, 0, 0.05)';
  const strokeColor = isDarkMode ? '#444' : '#ccc';

  const normalizeStr = (str) => {
    return str ? str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toLowerCase() : "";
  };

  const elem = document.getElementById('globeViz');
  
  // 【修复 1】：强制指定初始化时的宽度和高度，解决地球偏移问题
  const globe = Globe()(elem)
    .width(elem.clientWidth)
    .height(elem.clientHeight)
    .backgroundColor('rgba(0,0,0,0)')
    .showGlobe(false)
    .polygonAltitude(0)
    .polygonResolution(2); // 进一步降低内置分辨率，提升渲染速度

  // 【修复 2】：使用绝对路径，并加入 catch 捕捉错误
  Promise.all([
    fetch('/assets/json/countries.geojson').then(r => r.json()),
    fetch('/assets/json/provinces.geojson').then(r => r.json()),
    fetch('/assets/json/visited.json').then(r => r.json())
  ]).then(([countries, provinces, visited]) => {
    
    // 数据加载成功，隐藏 Loading 提示
    document.getElementById('loading').style.display = 'none';

    const normalizedFootprints = visited.map(normalizeStr);
    // 这里如果 json 里没有 missouri，系统也不会报错，不影响你的动态更新逻辑
    normalizedFootprints.push("missouri", "dubai");

    // 过滤省份数据
    const visitedProvinces = provinces.features.filter(feat => {
      const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
      return normalizedFootprints.some(place => 
        featureName === place || 
        (place.length > 3 && featureName.includes(place)) || 
        (featureName.length > 3 && place.includes(featureName))
      );
    });

    const allFeatures = [...countries.features, ...visitedProvinces];

    globe
      .polygonsData(allFeatures)
      .polygonCapColor(feat => {
        const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
        const isVisited = normalizedFootprints.some(place => 
          featureName === place || 
          (place.length > 3 && featureName.includes(place)) || 
          (featureName.length > 3 && place.includes(featureName))
        );

        if (isVisited || visitedProvinces.includes(feat)) {
          return 'rgba(220, 53, 69, 0.8)';
        }
        return baseColor;
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

  }).catch(error => {
    // 如果加载失败，在页面上显示报错信息，方便排查
    document.getElementById('loading').innerText = '⚠️ Error loading map data. Please check the browser console.';
    console.error("Fetch Data Error:", error);
  });

  // 窗口大小改变时，重新计算宽高
  window.addEventListener('resize', () => {
    globe.width(elem.clientWidth);
    globe.height(elem.clientHeight);
  });
</script>

<div class="mt-4">
  <p>The globe above highlights the provinces and regions I have visited so far. Feel free to drag to rotate and scroll to zoom in!</p>
</div>

<!-- prettier-ignore-end -->