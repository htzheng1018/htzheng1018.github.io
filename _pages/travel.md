---
layout: page
title: Travel
permalink: /travel/
nav: true
nav_order: 5 
---

<!-- prettier-ignore-start -->

<div style="position: relative; width: 100%; height: 600px; margin-top: 20px;">
  <div id="loading" style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); font-weight: bold; font-size: 1.2em; color: #888; z-index: 10;">
    🌍 Downloading Map Data, please wait...
  </div>
  <div id="globeViz" style="width: 100%; height: 100%; cursor: grab; border-radius: 10px; overflow: hidden;"></div>
</div>

<script src="https://unpkg.com/globe.gl"></script>

<script>
  const isDarkMode = document.documentElement.getAttribute('data-theme') === 'dark';
  
  // 【优化 1：取消透视】直接使用 100% 不透明的实体颜色 (Hex)
  const baseColor = isDarkMode ? '#2b2b2b' : '#e0e0e0'; // 未去过的陆地：深灰 / 浅灰
  const strokeColor = isDarkMode ? '#444444' : '#cccccc'; // 边界线颜色
  
  const normalizeStr = (str) => {
    return str ? str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toLowerCase() : "";
  };

  const elem = document.getElementById('globeViz');
  
  const globe = Globe()(elem)
    .width(elem.clientWidth)
    .height(elem.clientHeight)
    .backgroundColor('rgba(0,0,0,0)') // 保持外部背景透明
    .showGlobe(false) // 保持内核隐藏（海洋全透明）
    .polygonAltitude(0); 

  const loadData = (url) => {
    return fetch(url).then(r => {
      if (!r.ok) throw new Error(`找不到文件 (HTTP ${r.status}): ${url}`);
      return r.json();
    });
  };

  Promise.all([
    loadData('{{ "/assets/json/countries.geojson" | relative_url }}'),
    loadData('{{ "/assets/json/provinces.geojson" | relative_url }}'),
    loadData('{{ "/assets/json/visited.json" | relative_url }}')
  ]).then(([countries, provinces, visitedDict]) => {
    
    document.getElementById('loading').style.display = 'none';

    // 预处理字典
    const normalizedFootprints = {};
    for (const [countryCode, places] of Object.entries(visitedDict)) {
      normalizedFootprints[countryCode] = places.map(normalizeStr);
    }

    // 极简精准匹配
    const isPlaceVisited = (feat) => {
      const countryCode = feat.properties.iso_a2 || feat.properties.ISO_A2 || '';
      if (!normalizedFootprints[countryCode]) return false;

      const featureName = normalizeStr(feat.properties.name_en || feat.properties.name || '');
      const validPlaces = normalizedFootprints[countryCode];

      return validPlaces.includes(featureName);
    };

    const visitedProvinces = provinces.features.filter(feat => {
      if (isPlaceVisited(feat)) {
        feat.properties.isVisited = true; 
        return true;
      }
      return false;
    });

    const allFeatures = [...countries.features, ...visitedProvinces];

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
      .polygonCapColor(feat => {
        if (feat.properties.isVisited || isPlaceVisited(feat)) {
          // 【优化 1：取消透视】使用 100% 不透明的红色
          return '#dc3545'; 
        }
        return baseColor;
      })
      .polygonAltitude(feat => feat.properties.isVisited ? 0.005 : 0)
      .polygonStrokeColor(() => strokeColor)
      .polygonLabel(feat => {
        // 【优化 2：修复 Unknown】火力覆盖所有可能的命名属性！
        const name = feat.properties.name_en || 
                     feat.properties.NAME_EN || 
                     feat.properties.name || 
                     feat.properties.NAME || 
                     feat.properties.ADMIN || 
                     feat.properties.SOVEREIGNT || 
                     'Unknown';
                     
        return `
          <div style="background: rgba(0, 0, 0, 0.75); color: white; padding: 6px 10px; border-radius: 4px; font-size: 14px;">
            <b>${name}</b>
          </div>
        `;
      });

    globe.controls().autoRotate = false;
    globe.controls().autoRotateSpeed = 0.1;
    globe.controls().enableZoom = true;

  }).catch(error => {
    document.getElementById('loading').innerHTML = `<span style="color: #dc3545;">⚠️ 数据加载失败: <br>${error.message}</span>`;
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