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
  // 【微调1】：把多边形底色的透明度从 0.05 提高到 0.15，让陆地轮廓更明显
  const baseColor = isDarkMode ? 'rgba(255, 255, 255, 0.15)' : 'rgba(0, 0, 0, 0.15)';
  const strokeColor = isDarkMode ? '#444' : '#ccc';
  // 【新增】：定义一个与 al-folio 背景相融的“磨砂玻璃”地球内芯颜色，透明度设为 0.7
  const globeCoreColor = isDarkMode ? 'rgba(30, 30, 30, 0.7)' : 'rgba(250, 250, 250, 0.7)';

  const normalizeStr = (str) => {
    return str ? str.normalize("NFD").replace(/[\u0300-\u036f]/g, "").toLowerCase() : "";
  };

  const elem = document.getElementById('globeViz');
  
  const globe = Globe()(elem)
    .width(elem.clientWidth)
    .height(elem.clientHeight)
    .backgroundColor('rgba(0,0,0,0)')
    .showGlobe(true) // 【微调2】：把 false 改成 true，召唤地球本体
    .globeColor(globeCoreColor) // 【微调3】：注入磨砂玻璃颜色
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

    // 【极简精准匹配】：因为用了 name_en 和国家代码，不再需要任何复杂的正则和防碰瓷！
    const isPlaceVisited = (feat) => {
      const countryCode = feat.properties.iso_a2 || feat.properties.ISO_A2 || '';
      
      // 如果国家没去过，直接 false
      if (!normalizedFootprints[countryCode]) return false;

      // 提取地图的纯英文名
      const featureName = normalizeStr(feat.properties.name_en || feat.properties.name || '');
      const validPlaces = normalizedFootprints[countryCode];

      // 极速判断：精准相等即点亮
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
          return 'rgba(220, 53, 69, 0.8)';
        }
        return baseColor;
      })
      .polygonAltitude(feat => feat.properties.isVisited ? 0.005 : 0)
      .polygonStrokeColor(() => strokeColor)
      .polygonLabel(feat => {
        // Label 里依然显示它最初的 name，如果想全显示英文，可以把这里也改成 name_en
        const name = feat.properties.name_en || feat.properties.name || 'Unknown';
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