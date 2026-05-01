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
    🌍 Downloading Map Data (approx. 15MB), please wait...
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
  ]).then(([countries, provinces, visited]) => {
    
    document.getElementById('loading').style.display = 'none';

    const normalizedFootprints = visited.map(normalizeStr);

    // 【精准匹配逻辑封装】：彻底解决墨西哥、Mary等躺枪问题
    const isPlaceVisited = (featureName) => {
      if (!featureName) return false;
      return normalizedFootprints.some(place => {
        // 1. 完全精准一致
        if (featureName === place) return true;
        
        // 2. 特殊缩写处理
        if (place === 'dc' && featureName.includes('columbia')) return true;
        if (place === 'inner mongol' && (featureName.includes('mongol') || featureName.includes('nei'))) return true;

        // 3. 单词边界匹配 (解决 Maryland 点亮 Mary 的问题)
        if (place.length > 3) {
          try {
            // 使用 \b 确保匹配的是完整单词
            const regex = new RegExp(`\\b${place}\\b`, 'i');
            if (regex.test(featureName)) return true;
          } catch (e) {
            // 如果正则特殊字符报错，退回基础 includes (但已剔除反向包含)
            if (featureName.includes(place)) return true;
          }
        }
        return false;
      });
    };

    const visitedProvinces = provinces.features.filter(feat => {
      const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
      if (isPlaceVisited(featureName)) {
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
        const featureName = normalizeStr(feat.properties.name || feat.properties.NAME || feat.properties.NAME_1 || '');
        if (isPlaceVisited(featureName) || visitedProvinces.includes(feat)) {
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

    globe.controls().autoRotate = true;
    globe.controls().autoRotateSpeed = 1.0;
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