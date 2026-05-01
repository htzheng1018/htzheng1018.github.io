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
    .polygonAltitude(0); // <--- 注意，这里以分号结尾，删掉了下一行的 polygonResolution

  // 【终极升级】：严苛的 Fetch 拦截器，一旦找不到文件立刻在屏幕上报错！
  const loadData = (url) => {
    return fetch(url).then(r => {
      if (!r.ok) throw new Error(`找不到文件 (HTTP ${r.status}): ${url}`);
      return r.json();
    });
  };

  // 使用 Jekyll 的相对路径，防止 GitHub Pages 路径迷失
  Promise.all([
    loadData('{{ "/assets/json/countries.geojson" | relative_url }}'),
    loadData('{{ "/assets/json/provinces.geojson" | relative_url }}'),
    loadData('{{ "/assets/json/visited.json" | relative_url }}')
  ]).then(([countries, provinces, visited]) => {
    
    document.getElementById('loading').style.display = 'none';

    const normalizedFootprints = visited.map(normalizeStr);

    // 【优化 1】：只在加载时计算一次，过滤并给去过的省份打上 isVisited 标记
    const visitedProvinces = provinces.features.filter(feat => {
      const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
      const isVisited = normalizedFootprints.some(place => 
        featureName === place || 
        (place.length > 3 && featureName.includes(place)) || 
        (featureName.length > 3 && place.includes(featureName))
      );

      if (isVisited) {
        feat.properties.isVisited = true; // 打上专属红色通行证
        return true;
      }
      return false;
    });

    const allFeatures = [...countries.features, ...visitedProvinces];

    // 【终极解药】：强制将 Mapshaper 导出的顺时针坐标反转为逆时针
    allFeatures.forEach(feat => {
      if (!feat.geometry) return;
      if (feat.geometry.type === 'Polygon') {
        feat.geometry.coordinates.forEach(ring => ring.reverse());
      } else if (feat.geometry.type === 'MultiPolygon') {
        feat.geometry.coordinates.forEach(poly => poly.forEach(ring => ring.reverse()));
      }
    });

    globe
      .polygonsData(allFeatures)
      // 【优化 2】：极速读取标记，瞬间变红
      .polygonCapColor(feat => feat.properties.isVisited ? 'rgba(220, 53, 69, 0.8)' : baseColor)
      // 【优化 3】：解决黑红闪烁！去过的省份稍微凸起(0.005)，国家底图贴地(0)
      .polygonAltitude(feat => feat.properties.isVisited ? 0.005 : 0)
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
    // 拦截到错误，直接在网页中央显示红字！
    document.getElementById('loading').innerHTML = `<span style="color: #dc3545;">⚠️ 数据加载失败: <br>${error.message}<br>请检查 GitHub 仓库中文件的扩展名是 .json 还是 .geojson</span>`;
    console.error("Fetch Data Error:", error);
  });

  window.addEventListener('resize', () => {
    globe.width(elem.clientWidth);
    globe.height(elem.clientHeight);
  });
</script>

<div class="mt-4">
  <p>The globe above highlights the provinces and regions I have visited so far. Feel free to drag to rotate and scroll to zoom in!</p>
</div>

<!-- prettier-ignore-end -->