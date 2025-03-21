<template>
  <div class="map-container" ref="mapContainer"></div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watchEffect } from 'vue'
import { Map, View } from 'ol'
import TileLayer from 'ol/layer/Tile'
import XYZ from 'ol/source/XYZ'
import OSM from 'ol/source/OSM' // 添加这行
import { fromLonLat } from 'ol/proj'
import 'ol/ol.css' // 确保这行存在

const mapContainer = ref<HTMLElement | null>(null)
const map = ref<Map | null>(null)

// 定义props接收图层数据
const props = defineProps({
  geoJsonLayers: {
    type: Array,
    default: () => []
  }
})

// 定义事件
const emit = defineEmits(['map-ready'])

onMounted(() => {
  if (mapContainer.value) {
    console.log('Map container size:', mapContainer.value.offsetWidth, mapContainer.value.offsetHeight)
    
    // 创建ArcGIS卫星影像底图
    const arcGISLayer = new TileLayer({
      source: new XYZ({
        url: 'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
        maxZoom: 19
      })
    })
    
    // 创建OpenStreetMap底图作为备选
    const osmLayer = new TileLayer({
      source: new OSM()
    })

    // 创建地图，使用两个底图，如果ArcGIS不可用，会显示OSM
    map.value = new Map({
      target: mapContainer.value,
      layers: [osmLayer, arcGISLayer], // 添加两个底图
      view: new View({
        center: fromLonLat([116.3833, 39.9]), // 默认中心点（北京）
        zoom: 5
      })
    })

    // 通知父组件地图已准备好
    emit('map-ready', map.value)
    
    // 确保地图渲染
    setTimeout(() => {
      if (map.value) {
        map.value.updateSize();
      }
    }, 200);
    
    // 添加窗口大小变化监听器
    window.addEventListener('resize', handleResize)
  }
})

onUnmounted(() => {
  if (map.value) {
    map.value.setTarget(undefined)
    map.value = null
  }
  window.removeEventListener('resize', handleResize)
})

// 处理窗口大小变化
const handleResize = () => {
  if (map.value) {
    console.log('Resizing map to:', mapContainer.value?.offsetWidth, mapContainer.value?.offsetHeight)
    map.value.updateSize()
  }
}

// 暴露地图实例给父组件
defineExpose({
  map
})
</script>

<style scoped>
.map-container {
  width: 100%;
  height: 100%;
  position: relative; /* 添加这行 */
}
</style>