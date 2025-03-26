<template>
  <div class="home-container">
    <div class="map-container">
      <MapView ref="mapViewRef" @map-ready="onMapReady" />
      <DrawToolbar
        :visible="currentEditingLayer !== null"
        @tool-change="onDrawToolChange"
        @stop-drawing="stopDrawing"
      />
    </div>
    <div class="layer-manager-container">
      <LayerManager
        @add-layer="addLayer"
        @remove-layer="removeLayer"
        @toggle-layer="toggleLayerVisibility"
        @zoom-to-layer="zoomToLayer"
        @edit-layer="onEditLayer"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import MapView from '@/components/MapView.vue'
import LayerManager from '@/components/LayerManager.vue'
import DrawToolbar from '@/components/DrawToolbar.vue'
import { Map } from 'ol'
import VectorLayer from 'ol/layer/Vector'
import VectorSource from 'ol/source/Vector'
import GeoJSON from 'ol/format/GeoJSON'
import { ElMessage } from 'element-plus'
import Draw from 'ol/interaction/Draw'
import { Feature } from 'ol'
import { Geometry } from 'ol/geom'

const mapViewRef = ref<any>(null)
const olMap = ref<Map | null>(null)
const currentEditingLayer = ref<LayerInfo | null>(null)
const currentDraw = ref<Draw | null>(null)

// 地图准备好后的回调
const onMapReady = (map: Map) => {
  olMap.value = map
}

// 处理图层编辑状态变更
const onEditLayer = (layer: LayerInfo) => {
  if (layer.isEditing) {
    currentEditingLayer.value = layer
  } else {
    stopDrawing()
    currentEditingLayer.value = null
  }
}

// 处理绘制工具变更
const onDrawToolChange = (geometryType: string) => {
  if (!olMap.value || !currentEditingLayer.value?.olLayer) return

  // 移除现有的绘制交互
  stopDrawing()

  // 创建新的绘制交互
  const draw = new Draw({
    source: currentEditingLayer.value.olLayer.getSource(),
    type: geometryType,
  })

  // 监听绘制完成事件
  draw.on('drawend', (event) => {
    const feature = event.feature as Feature<Geometry>
    // 移除实时更新GeoJSON逻辑
  })

  // 添加绘制交互到地图
  olMap.value.addInteraction(draw)
  currentDraw.value = draw
}

// 停止绘制
const stopDrawing = () => {
  if (olMap.value && currentDraw.value) {
    olMap.value.removeInteraction(currentDraw.value)
    currentDraw.value = null
  }
}

// 添加图层
const addLayer = (layerInfo: LayerInfo) => {
  if (!olMap.value) {
    ElMessage.error('地图尚未初始化')
    return
  }

  try {
    // 创建OpenLayers矢量图层
    const vectorSource = new VectorSource({
      features: new GeoJSON().readFeatures(layerInfo.data, {
        featureProjection: 'EPSG:3857', // Web墨卡托投影
      }),
    })

    const vectorLayer = new VectorLayer({
      source: vectorSource,
      visible: layerInfo.visible,
    })

    // 将OpenLayers图层添加到地图
    olMap.value.addLayer(vectorLayer)

    // 更新图层信息中的OpenLayers图层引用
    layerInfo.olLayer = vectorLayer

    // 如果有要素，缩放到图层范围
    if (vectorSource.getFeatures().length > 0) {
      const extent = vectorSource.getExtent()
      olMap.value.getView().fit(extent, {
        padding: [50, 50, 50, 50],
        duration: 1000,
      })
    }
  } catch (error) {
    ElMessage.error('添加图层失败')
    console.error(error)
  }
}

// 移除图层
const removeLayer = (layerInfo: LayerInfo) => {
  if (!olMap.value || !layerInfo.olLayer) return

  olMap.value.removeLayer(layerInfo.olLayer)
}

// 切换图层可见性
const toggleLayerVisibility = (layerInfo: LayerInfo) => {
  if (layerInfo.olLayer) {
    layerInfo.olLayer.setVisible(layerInfo.visible)
  }
}

// 缩放至图层
const zoomToLayer = (layerInfo: LayerInfo) => {
  if (!olMap.value || !layerInfo.olLayer) return

  const vectorSource = layerInfo.olLayer.getSource()
  if (vectorSource && vectorSource.getFeatures().length > 0) {
    const extent = vectorSource.getExtent()
    olMap.value.getView().fit(extent, {
      padding: [50, 50, 50, 50],
      duration: 1000,
    })
  }
}
</script>

<style scoped>
.home-container {
  height: 100vh;
  width: 100vw;
  position: relative;
  overflow: hidden;
}

.map-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 0;
}

.layer-manager-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 300px;
  height: 100%;
  background-color: rgba(245, 247, 250, 0.95);
  border-right: 1px solid #e6e6e6;
  overflow-y: auto;
  z-index: 1;
  box-shadow: 2px 0 10px rgba(0, 0, 0, 0.1);
  padding: 0;
  margin: 0;
}
</style>
