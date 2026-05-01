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
  
  // 初始化地球仪（基础配置）
  const globe = Globe()(elem)
    .width(elem.clientWidth)
    .height(elem.clientHeight)
    .backgroundColor('rgba(0,0,0,0)')
    .showGlobe(false)
    .polygonResolution(2); 

  // 严苛的 Fetch 拦截器
  const loadData = (url) => {
    return fetch(url).then(r => {
      if (!r.ok) throw new Error(`找不到文件 (HTTP ${r.status}): ${url}`);
      return r.json();
    });
  };

  // 【终极匹配算法】：严谨判断，绝不乱杀无辜
  const checkVisited = (featureName, footprints) => {
    if (!featureName) return false;
    
    return footprints.some(place => {
      // 1. 完全精准匹配
      if (featureName === place) return true;
      
      // 2. 允许带行政后缀的匹配 (如：地图叫 "sichuan province"，你写的是 "sichuan")
      // 限制条件：地图名字必须包含你的足迹，且长度不能相差超过 12 个字符
      if (place.length > 3 && featureName.includes(place) && (featureName.length - place.length <= 12)) {
        return true;
      }

      // 3. 特殊别名处理
      if (place === 'inner mongol' && (featureName.includes('mongol') || featureName.includes('nei'))) return true;
      if (place === 'dc' && featureName.includes('columbia')) return true;

      return false;
    });
  };

  Promise.all([
    loadData('{{ "/assets/json/countries.geojson" | relative_url }}'),
    loadData('{{ "/assets/json/provinces.geojson" | relative_url }}'),
    loadData('{{ "/assets/json/visited.json" | relative_url }}')
  ]).then(([countries, provinces, visited]) => {
    
    document.getElementById('loading').style.display = 'none';

    // 预处理你的 visited.json
    const normalizedFootprints = visited.map(normalizeStr);
    
    // 过滤出你去过的省份，打上专属标记
    const visitedProvinces = provinces.features.filter(feat => {
      const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
      if (checkVisited(featureName, normalizedFootprints)) {
        feat.properties.isVisited = true; // 贴上标签，提升后续渲染速度
        return true;
      }
      return false;
    });

    const allFeatures = [...countries.features, ...visitedProvinces];

    // 修复 Mapshaper 导出的 3D 拓扑翻转问题 (将顺时针改为逆时针)
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
      // 【修复】：根据我们刚才打的 isVisited 标签，瞬间变红
      .polygonCapColor(feat => feat.properties.isVisited ? 'rgba(220, 53, 69, 0.8)' : baseColor)
      // 【修复】：解决闪烁！去过的省份微微凸起 (0.005)，国家底图完全贴地 (0)
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
    document.getElementById('loading').innerHTML = `<span style="color: #dc3545;">⚠️ 数据加载失败: <br>${error.message}<br>请检查文件路径和后缀名</span>`;
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