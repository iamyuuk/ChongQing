<template>
  <div class="map-wrapper">
    <!-- 顶部搜索栏 -->
    <div class="search-bar">
      <div class="search-input-wrap">
        <span class="search-icon">🔍</span>
        <input
          v-model="searchKeyword"
          type="search"
          placeholder="搜索重庆景点或地点..."
          enterkeyhint="search"
          @keyup.enter="onSearch"
          @focus="isSearchFocused = true"
        />
        <button
          v-if="searchKeyword"
          class="search-clear"
          @click="clearSearch"
        >
          ✕
        </button>
      </div>
      <button
        v-if="isSearchFocused"
        class="search-cancel"
        @click="cancelSearch"
      >
        取消
      </button>
    </div>

    <!-- 搜索结果浮层 -->
    <div
      v-if="isSearchFocused && searchResults.length"
      class="search-results"
    >
      <div
        v-for="item in searchResults"
        :key="item.id"
        class="search-result-item"
        @click="goToPlace(item)"
      >
        <div class="result-name">{{ item.name }}</div>
        <div class="result-addr">{{ item.address }}</div>
      </div>
    </div>

    <!-- 点击搜索结果外部关闭 -->
    <div
      v-if="isSearchFocused && searchResults.length"
      class="search-overlay"
      @click="cancelSearch"
    ></div>

    <div id="map-container"></div>

    <!-- iOS 风格底部卡片 -->
    <div
      class="spot-card"
      :class="{ show: activeSpot }"
      @click.self="closeCard"
    >
      <div class="card-handle"></div>
      <div v-if="activeSpot" class="card-content">
        <h2 class="spot-name">{{ activeSpot.name }}</h2>
        <div class="spot-tags">
          <span v-for="tag in activeSpot.tags" :key="tag" class="tag">{{ tag }}</span>
        </div>
        <p class="spot-desc">{{ activeSpot.description }}</p>
        <div class="spot-tips">
          <h3>📝 攻略 & 注意事项</h3>
          <ul>
            <li v-for="(tip, idx) in activeSpot.tips" :key="idx">{{ tip }}</li>
          </ul>
        </div>

        <button
          v-if="activeSpot.video"
          class="video-btn"
          @click="openDouyin"
        >
          🎵 查看抖音攻略
        </button>

        <button class="copy-btn" @click="copyPosition">
          {{ copyLabel }}
        </button>

        <a
          class="nav-btn"
          :href="navLink"
          target="_blank"
        >
          🧭 导航去这里
        </a>
      </div>
    </div>

    <!-- 点击卡片外部关闭的遮罩 -->
    <div
      v-show="activeSpot"
      class="card-overlay"
      @click="closeCard"
    ></div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref, computed } from 'vue'
import AMapLoader from '@amap/amap-jsapi-loader'
import { spots } from '../data/spots.js'

const map = ref(null)
const activeSpot = ref(null)

const searchKeyword = ref('')
const searchResults = ref([])
const isSearchFocused = ref(false)
const copyLabel = ref('📋 复制坐标')
let placeSearch = null
let tempMarker = null

const navLink = computed(() => {
  if (!activeSpot.value) return '#'
  const [lng, lat] = activeSpot.value.position
  return `http://maps.apple.com/?daddr=${lat},${lng}&dirflg=r`
})

function closeCard() {
  activeSpot.value = null
}

function createMarkerContent(name) {
  const div = document.createElement('div')
  div.className = 'custom-marker'
  div.innerHTML = `
    <div class="marker-pin">
      <div class="marker-inner"></div>
    </div>
    <div class="marker-label">${name}</div>
  `
  return div
}

function onSearch() {
  if (!placeSearch || !searchKeyword.value.trim()) return
  placeSearch.search(searchKeyword.value, (status, result) => {
    if (status === 'complete' && result.info === 'OK') {
      searchResults.value = result.poiList.pois
    } else {
      searchResults.value = []
    }
  })
}

function goToPlace(poi) {
  if (tempMarker) {
    map.value.remove(tempMarker)
    tempMarker = null
  }

  const pos = [poi.location.lng, poi.location.lat]

  tempMarker = new window.AMap.Marker({
    position: pos,
    content: createMarkerContent(poi.name),
    offset: new window.AMap.Pixel(-20, -40),
    anchor: 'bottom-center',
  })
  tempMarker.on('click', () => {
    activeSpot.value = {
      name: poi.name,
      position: pos,
      tags: [],
      description: poi.address || '',
      tips: [],
      video: '',
    }
  })
  map.value.add(tempMarker)

  map.value.setCenter(pos)
  map.value.setZoom(16)

  searchKeyword.value = poi.name
  searchResults.value = []
  isSearchFocused.value = false
}

function clearSearch() {
  searchKeyword.value = ''
  searchResults.value = []
  if (tempMarker) {
    map.value.remove(tempMarker)
    tempMarker = null
  }
}

function cancelSearch() {
  isSearchFocused.value = false
  searchResults.value = []
}

function copyPosition() {
  if (!activeSpot.value) return
  const [lng, lat] = activeSpot.value.position
  const text = `position: [${lng}, ${lat}],`

  const onSuccess = () => {
    copyLabel.value = '✅ 已复制'
    setTimeout(() => {
      copyLabel.value = '📋 复制坐标'
    }, 2000)
  }

  if (navigator.clipboard) {
    navigator.clipboard.writeText(text).then(onSuccess).catch(() => {
      fallbackCopy(text, onSuccess)
    })
  } else {
    fallbackCopy(text, onSuccess)
  }
}

function openDouyin() {
  if (!activeSpot.value || !activeSpot.value.video) return
  const url = activeSpot.value.video

  // 1. 复制到剪贴板
  if (navigator.clipboard) {
    navigator.clipboard.writeText(url).catch(() => {
      fallbackCopy(url, () => {})
    })
  } else {
    fallbackCopy(url, () => {})
  }

  // 2. 询问是否打开抖音
  if (confirm('链接已复制到剪贴板，是否打开抖音查看视频？')) {
    window.location.href = url
  }
}

function fallbackCopy(text, callback) {
  const input = document.createElement('input')
  input.value = text
  document.body.appendChild(input)
  input.select()
  document.execCommand('copy')
  document.body.removeChild(input)
  callback()
}

onMounted(() => {
  window._AMapSecurityConfig = {
    securityJsCode: 'eae2c7526d782d00c802c818566eb40a',
  }

  AMapLoader.load({
    key: '6f224192f718a8a6b136ba463d778bc8',
    version: '2.0',
    plugins: ['AMap.Scale', 'AMap.ToolBar', 'AMap.Geolocation', 'AMap.PlaceSearch', 'AMap.Geocoder'],
  })
    .then((AMap) => {
      map.value = new AMap.Map('map-container', {
        zoom: 11,
        center: [106.5516, 29.563],
        viewMode: '2D',
      })

      map.value.addControl(new AMap.Scale())
      map.value.addControl(new AMap.ToolBar())

      const geolocation = new AMap.Geolocation({
        enableHighAccuracy: true,
        timeout: 10000,
        buttonPosition: 'LB',
        buttonOffset: new AMap.Pixel(10, 20),
        zoomToAccuracy: true,
      })
      map.value.addControl(geolocation)

      geolocation.getCurrentPosition((status, result) => {
        if (status === 'complete') {
          console.log('定位成功', result)
        } else {
          console.warn('定位失败，使用默认中心', result)
        }
      })

      placeSearch = new AMap.PlaceSearch({
        city: '重庆',
        citylimit: true,
        pageSize: 8,
      })

      // 点击地图任意位置，逆地理编码识别地标
      const geocoder = new AMap.Geocoder({
        radius: 50,
        extensions: 'all',
      })
      map.value.on('click', (e) => {
        geocoder.getAddress(e.lnglat, (status, result) => {
          if (status === 'complete' && result.info === 'OK') {
            const pois = result.regeocode.pois
            if (pois && pois.length > 0) {
              const poi = pois[0]
              activeSpot.value = {
                name: poi.name,
                position: [poi.location.lng, poi.location.lat],
                tags: poi.type ? [poi.type] : [],
                description: result.regeocode.formattedAddress || '',
                tips: [],
                video: '',
              }
              map.value.setCenter([poi.location.lng, poi.location.lat])
              map.value.setZoom(16)
            }
          }
        })
      })

      // 添加景点标记
      spots.forEach((spot) => {
        const marker = new AMap.Marker({
          position: spot.position,
          content: createMarkerContent(spot.name),
          offset: new AMap.Pixel(-20, -40),
          anchor: 'bottom-center',
        })

        marker.on('click', () => {
          activeSpot.value = spot
          map.value.setCenter(spot.position)
          map.value.setZoom(14)
        })

        map.value.add(marker)
      })
    })
    .catch((err) => {
      console.error('地图加载失败:', err)
    })
})

onUnmounted(() => {
  if (map.value) {
    map.value.destroy()
    map.value = null
  }
})
</script>

<style scoped>
.map-wrapper {
  position: relative;
  width: 100%;
  height: 100vh;
  height: 100dvh;
  overflow: hidden;
}

#map-container {
  width: 100%;
  height: 100%;
  touch-action: none;
}

/* 搜索栏 */
.search-bar {
  position: absolute;
  top: env(safe-area-inset-top, 12px);
  left: 0;
  right: 0;
  z-index: 30;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 16px;
  pointer-events: none;
}

.search-bar > * {
  pointer-events: auto;
}

.search-input-wrap {
  flex: 1;
  display: flex;
  align-items: center;
  height: 40px;
  padding: 0 12px;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(12px);
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.search-icon {
  font-size: 14px;
  margin-right: 6px;
  color: #8e8e93;
}

.search-input-wrap input {
  flex: 1;
  border: none;
  outline: none;
  background: transparent;
  font-size: 15px;
  color: #1c1c1e;
}

.search-input-wrap input::placeholder {
  color: #8e8e93;
}

.search-clear {
  width: 20px;
  height: 20px;
  border: none;
  border-radius: 50%;
  background: #c7c7cc;
  color: #fff;
  font-size: 11px;
  display: flex;
  align-items: center;
  justify-content: center;
  -webkit-tap-highlight-color: transparent;
}

.search-cancel {
  border: none;
  background: transparent;
  font-size: 15px;
  color: #007aff;
  padding: 0 4px;
  -webkit-tap-highlight-color: transparent;
}

/* 搜索结果 */
.search-results {
  position: absolute;
  top: calc(env(safe-area-inset-top, 12px) + 56px);
  left: 16px;
  right: 16px;
  z-index: 30;
  max-height: 45vh;
  background: rgba(255, 255, 255, 0.96);
  backdrop-filter: blur(16px);
  border-radius: 12px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.1);
  overflow-y: auto;
  overscroll-behavior: contain;
}

.search-result-item {
  padding: 12px 16px;
  border-bottom: 1px solid #f2f2f7;
}

.search-result-item:last-child {
  border-bottom: none;
}

.result-name {
  font-size: 15px;
  font-weight: 500;
  color: #1c1c1e;
  margin-bottom: 2px;
}

.result-addr {
  font-size: 13px;
  color: #8e8e93;
}

.search-overlay {
  position: absolute;
  inset: 0;
  z-index: 25;
}

/* 自定义标记点样式 */
:global(.custom-marker) {
  display: flex;
  flex-direction: column;
  align-items: center;
  cursor: pointer;
}

:global(.marker-pin) {
  width: 32px;
  height: 32px;
  border-radius: 50% 50% 50% 0;
  background: #ff3b30;
  transform: rotate(-45deg);
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.3);
}

:global(.marker-inner) {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: #fff;
  transform: rotate(45deg);
}

:global(.marker-label) {
  margin-top: 4px;
  padding: 2px 8px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
  color: #1c1c1e;
  white-space: nowrap;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
}

/* 底部卡片遮罩 */
.card-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0);
  z-index: 10;
  pointer-events: auto;
}

/* iOS 风格底部卡片 */
.spot-card {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 20;
  max-height: 55vh;
  background: rgba(255, 255, 255, 0.92);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-radius: 20px 20px 0 0;
  box-shadow: 0 -4px 20px rgba(0, 0, 0, 0.08);
  transform: translateY(110%);
  transition: transform 0.35s cubic-bezier(0.32, 0.72, 0, 1);
  overflow-y: auto;
  overscroll-behavior: contain;
}

.spot-card.show {
  transform: translateY(0);
}

.card-handle {
  width: 36px;
  height: 4px;
  border-radius: 2px;
  background: rgba(60, 60, 67, 0.3);
  margin: 8px auto 0;
}

.card-content {
  padding: 16px 20px 32px;
}

.spot-name {
  font-size: 22px;
  font-weight: 700;
  color: #1c1c1e;
  margin: 0 0 10px;
}

.spot-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 12px;
}

.tag {
  padding: 3px 10px;
  border-radius: 12px;
  background: #f2f2f7;
  font-size: 12px;
  color: #636366;
  font-weight: 500;
}

.spot-desc {
  font-size: 15px;
  line-height: 1.6;
  color: #3a3a3c;
  margin: 0 0 16px;
}

.spot-tips h3 {
  font-size: 15px;
  font-weight: 600;
  color: #1c1c1e;
  margin: 0 0 10px;
}

.spot-tips ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.spot-tips li {
  position: relative;
  padding-left: 16px;
  font-size: 14px;
  line-height: 1.7;
  color: #48484a;
  margin-bottom: 6px;
}

.spot-tips li::before {
  content: '';
  position: absolute;
  left: 0;
  top: 9px;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #ff3b30;
}

/* 复制坐标按钮 */
.copy-btn {
  display: block;
  width: 100%;
  margin-top: 16px;
  padding: 14px 0;
  border-radius: 12px;
  background: #f2f2f7;
  color: #007aff;
  font-size: 16px;
  font-weight: 600;
  text-align: center;
  border: none;
  -webkit-tap-highlight-color: transparent;
  transition: background 0.2s;
}

.copy-btn:active {
  background: #e5e5ea;
}

/* 导航按钮 */
.nav-btn {
  display: block;
  width: 100%;
  margin-top: 12px;
  padding: 14px 0;
  border-radius: 12px;
  background: #007aff;
  color: #fff;
  font-size: 16px;
  font-weight: 600;
  text-align: center;
  text-decoration: none;
  -webkit-tap-highlight-color: transparent;
  transition: background 0.2s;
}

.nav-btn:active {
  background: #005ecb;
}

/* 抖音视频按钮 */
.video-btn {
  display: block;
  width: 100%;
  margin-top: 12px;
  padding: 14px 0;
  border: none;
  border-radius: 12px;
  background: #000;
  color: #fff;
  font-size: 16px;
  font-weight: 600;
  text-align: center;
  text-decoration: none;
  -webkit-tap-highlight-color: transparent;
  transition: background 0.2s;
}

.video-btn:active {
  background: #333;
}
</style>
