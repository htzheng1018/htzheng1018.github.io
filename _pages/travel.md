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
  const baseColor = isDarkMode ? 'rgba(255, 255, 255, 0.05)' : 'rgba(0, 0, 0, 0.05)';
  const strokeColor = isDarkMode ? '#444' : '#ccc';

  const normalizeStr = (str) => {
    return str ? str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toLowerCase() : "";
  };

  const elem = document.getElementById('globeViz');
  
  const globe = Globe()(elem)
    .width(elem.clientWidth)
    .height(elem.clientHeight)
    .backgroundColor('rgba(0,0,0,0)')
    .showGlobe(false)
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

    // 预处理字典：将所有省份名称转小写并去音标
    const normalizedFootprints = {};
    for (const [countryCode, places] of Object.entries(visitedDict)) {
      normalizedFootprints[countryCode] = places.map(normalizeStr);
    }

    // 【结构化匹配】：先卡国家代码，再查省份
    const isPlaceVisited = (feat) => {
      const countryCode = feat.properties.iso_a2 || feat.properties.ISO_A2 || '';
      
      // 如果这个国家你根本没去过，直接一刀切判定为 false，极速过滤！
      if (!normalizedFootprints[countryCode]) return false;

      const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
      const validPlaces = normalizedFootprints[countryCode];

      return validPlaces.some(place => {
        // 1. 完全精准一致
        if (featureName === place) return true;
        
        // 2. 特殊缩写处理
        if (place === 'dc' && (featureName === 'district of columbia' || featureName === 'washington dc')) return true;
        if (place === 'inner mongol' && (featureName.includes('mongol') || featureName.includes('nei'))) return true;

        // 3. 单词边界匹配
        if (place.length > 3) {
          try {
            const regex = new RegExp(`\\b${place}\\b`, 'i');
            if (regex.test(featureName)) return true;
          } catch (e) {
            return false;
          }
        }
        return false;
      });
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
          return 'rgba(220, 53, 69, 0.8)';
        }
        return baseColor;
      })
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

    globe.controls().autoRotate = false;
    globe.controls().autoRotateSpeed = 0.1;
    globe.controls().enableZoom = true;

  }).catch(error => {
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