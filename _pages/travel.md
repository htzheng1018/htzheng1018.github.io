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
  
  // 【配色大升级】：引入海洋颜色，并让陆地和海洋在亮/暗模式下完美适配
  const oceanColor = isDarkMode ? '#041122' : '#c6e0ff'; // 海洋：深藏青 / 浅海蓝
  const baseColor = isDarkMode ? '#2b2b2b' : '#f8f9fa';  // 没去过的陆地：深灰 / 极浅灰（近白）
  const strokeColor = isDarkMode ? '#444444' : '#cccccc'; // 边界线
  
  const normalizeStr = (str) => {
    return str ? str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toLowerCase() : "";
  };

  const elem = document.getElementById('globeViz');
  
  const globe = Globe()(elem)
    .width(elem.clientWidth)
    .height(elem.clientHeight)
    .backgroundColor('rgba(0,0,0,0)') 
    // 【修改点】：召唤地球本体作为海洋，并涂上我们调配的蓝色
    .showGlobe(true) 
    .globeColor(oceanColor); 

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

    const normalizedFootprints = {};
    for (const [countryCode, places] of Object.entries(visitedDict)) {
      normalizedFootprints[countryCode] = places.map(normalizeStr);
    }

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
          return '#dc3545'; // 100% 不透明的红色
        }
        return baseColor;
      })
      // 【修改点】：给陆地增加一点点海拔，防止和蓝色的海洋表面打架闪烁
      .polygonAltitude(feat => feat.properties.isVisited ? 0.01 : 0.005)
      .polygonStrokeColor(() => strokeColor)
      .polygonLabel(feat => {
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