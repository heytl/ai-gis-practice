<template>
  <div class="layer-manager">
    <div class="layer-header">
      <h3>图层管理</h3>
      <el-dropdown>
        <el-button type="primary" :icon="Plus">图层操作</el-button>
        <template #dropdown>
          <el-dropdown-menu>
            <el-dropdown-item @click="createEmptyLayer"
              ><el-icon><Edit /></el-icon>新建图层</el-dropdown-item
            >
            <el-upload
              class="upload-btn"
              action=""
              :auto-upload="false"
              :show-file-list="false"
              accept=".geojson,.json"
              :on-change="handleFileChange"
            >
              <el-dropdown-item
                ><el-icon><Upload /></el-icon>导入GeoJSON</el-dropdown-item
              >
            </el-upload>
            <el-dropdown-item @click="openExcelDialog"
              ><el-icon><Upload /></el-icon>导入表图层</el-dropdown-item
            >
          </el-dropdown-menu>
        </template>
      </el-dropdown>
    </div>

    <div class="layer-list">
      <el-empty v-if="layers.length === 0" description="暂无图层" />
      <el-card
        v-for="(layer, index) in layers"
        :key="index"
        class="layer-item"
        :class="{ 'editing-layer': layer.isEditing }"
        @contextmenu.prevent="showContextMenu($event, layer, index)"
      >
        <div class="layer-item-content">
          <div class="layer-info">
            <el-switch v-model="layer.visible" @change="toggleLayerVisibility(layer)" />
            <span class="layer-name">{{ layer.name }}</span>
          </div>
          <div class="layer-actions">
            <el-button
              type="info"
              :icon="MoreFilled"
              circle
              size="small"
              title="更多操作"
              @click="showContextMenu($event, layer, index, true)"
            ></el-button>
          </div>
        </div>
      </el-card>
    </div>

    <!-- 缓冲区分析弹窗 -->
    <buffer-analysis ref="bufferAnalysisRef" @buffer-complete="handleBufferComplete" />

    <!-- Excel导入表图层弹窗 -->
    <excel-to-geo-json-dialog v-model:visible="excelDialogVisible" @import="handleExcelImport" />

    <!-- 统一的上下文菜单 -->
    <div
      ref="contextMenuRef"
      v-if="contextMenuVisible"
      class="context-menu"
      :style="contextMenuStyle"
      @click.stop
    >
      <div class="context-menu-item" @click="handleContextMenuCommand('zoom')">
        <el-icon><ZoomIn /></el-icon>
        <span>缩放至图层</span>
      </div>
      <div class="context-menu-item" @click="handleContextMenuCommand('export')">
        <el-icon><Download /></el-icon>
        <span>导出图层</span>
      </div>
      <div class="context-menu-item" @click="handleContextMenuCommand('remove')">
        <el-icon><Delete /></el-icon>
        <span>删除图层</span>
      </div>
      <div class="context-menu-item" @click="handleContextMenuCommand('buffer')">
        <el-icon><Opportunity /></el-icon>
        <span>缓冲区分析</span>
      </div>
      <div
        v-if="activeLayer?.layer.isEditable"
        class="context-menu-item"
        @click="handleContextMenuCommand('edit')"
      >
        <el-icon><Edit /></el-icon>
        <span>{{ activeLayer?.layer.isEditing ? '停止编辑' : '开始编辑' }}</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick } from 'vue' // 导入 nextTick
import {
  Plus,
  Download,
  Delete,
  MoreFilled,
  ZoomIn,
  Opportunity,
  Edit,
} from '@element-plus/icons-vue'
import BufferAnalysis from './BufferAnalysis.vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import GeoJSON from 'ol/format/GeoJSON'
import ExcelToGeoJsonDialog from './ExcelToGeoJsonDialog.vue'

// 定义图层类型
export interface LayerInfo {
  id: string
  name: string
  visible: boolean
  data: any
  olLayer: any
  isEditable: boolean
  isEditing: boolean
}

const layers = ref<LayerInfo[]>([])
const currentEditingLayer = ref<LayerInfo | null>(null)
const excelDialogVisible = ref(false)
const openExcelDialog = () => {
  excelDialogVisible.value = true
}
const handleExcelImport = (geojson: any) => {
  addGeoJsonLayer('表图层_' + (layers.value.length + 1), geojson, true)
  ElMessage.success('表图层导入成功')
}

// 定义事件
const emit = defineEmits([
  'add-layer',
  'remove-layer',
  'toggle-layer',
  'zoom-to-layer',
  'buffer-complete',
  'edit-layer',
])

const currentLayer = ref<LayerInfo>()
const bufferAnalysisRef = ref(null)

// 处理文件上传
const handleFileChange = (file: any) => {
  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const geoJson = JSON.parse(e.target?.result as string)
      addGeoJsonLayer(file.name.replace(/\.(geojson|json)$/i, ''), geoJson)
    } catch (error) {
      ElMessage.error('GeoJSON文件解析失败')
      console.error(error)
    }
  }
  reader.readAsText(file.raw)
}

// 创建空白图层
const createEmptyLayer = () => {
  const defaultName = `新建图层_${layers.value.length + 1}`
  ElMessageBox.prompt('请输入图层名称', '新建图层', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    inputValue: defaultName,
    inputPattern: /^[^\\/:*?"<>|]+$/,
    inputErrorMessage: '名称不能包含特殊字符',
  })
    .then(({ value }) => {
      if (!value) return
      const emptyGeoJson = {
        type: 'FeatureCollection',
        features: [],
      }
      addGeoJsonLayer(value, emptyGeoJson, true)
    })
    .catch(() => {})
}

// 处理图层名称确认
const handleLayerNameConfirm = (layerName: string) => {
  const emptyGeoJson = {
    type: 'FeatureCollection',
    features: [],
  }
  addGeoJsonLayer(layerName, emptyGeoJson, true)
}

// 添加GeoJSON图层
const addGeoJsonLayer = (name: string, geoJson: any, isEditable: boolean = false) => {
  // 生成唯一ID
  const layerId = 'layer_' + Date.now()

  // 创建图层对象
  const layerInfo: LayerInfo = {
    id: layerId,
    name: name,
    visible: true,
    data: geoJson,
    olLayer: null,
    isEditable: isEditable,
    isEditing: false,
  }

  // 添加到图层列表
  layers.value.push(layerInfo)

  // 通知父组件添加图层
  emit('add-layer', layerInfo)

  ElMessage.success(`成功导入图层: ${layerInfo.name}`)
}

// 切换图层可见性
const toggleLayerVisibility = (layer: LayerInfo) => {
  emit('toggle-layer', layer)
}

// 导出图层
const exportLayer = (layer: LayerInfo) => {
  if (layer.isEditing) {
    ElMessage.warning('编辑中的图层无法导出')
    return
  }

  // 使用弹窗让用户确认或修改导出文件名
  ElMessageBox.prompt('请输入导出文件名', '导出图层', {
    confirmButtonText: '导出',
    cancelButtonText: '取消',
    inputValue: layer.name,
    inputPattern: /^[^\\/:*?"<>|]+$/,
    inputErrorMessage: '文件名不能包含特殊字符',
  })
    .then(({ value }) => {
      if (!value) return

      let blob: Blob
      if (layer.isEditable) {
        const format = new GeoJSON()
        const features = layer.olLayer.getSource().getFeatures()
        const geoJson = format.writeFeatures(features, {
          dataProjection: 'EPSG:4326',
          featureProjection: 'EPSG:3857',
        })
        blob = new Blob([geoJson], { type: 'application/json' })
      } else {
        blob = new Blob([JSON.stringify(layer.data)], { type: 'application/json' })
      }
      const url = URL.createObjectURL(blob)
      const link = document.createElement('a')
      link.href = url
      link.download = `${value}.geojson`
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
      URL.revokeObjectURL(url)
      ElMessage.success(`成功导出图层: ${value}`)
    })
    .catch(() => {
      // 用户取消导出
    })
}

// 删除图层
const removeLayer = (index: number) => {
  const layer = layers.value[index]
  layers.value.splice(index, 1)
  emit('remove-layer', layer)
  ElMessage.success(`成功删除图层: ${layer.name}`)
}

// 由于已统一使用上下文菜单，不再需要单独的下拉菜单处理函数

// 统一的上下文菜单相关
const contextMenuVisible = ref(false)
const contextMenuStyle = ref({
  top: '0px',
  left: '0px',
  visibility: 'visible',
})
const activeLayer = ref<{ layer: LayerInfo; index: number } | null>(null)
const contextMenuRef = ref<HTMLElement | null>(null) // 添加 contextMenuRef

// 显示上下文菜单（同时处理右键和按钮点击）
const showContextMenu = async (
  event: MouseEvent,
  layer: LayerInfo,
  index: number,
  isButtonClick: boolean = false,
) => {
  // 阻止事件冒泡
  event.stopPropagation()

  let initialTop = 0
  let initialLeft = 0

  // 根据触发方式设置初始菜单位置
  if (isButtonClick) {
    // 如果是按钮点击，菜单显示在按钮下方
    const buttonRect = (event.currentTarget as HTMLElement).getBoundingClientRect()
    initialTop = buttonRect.bottom
    initialLeft = buttonRect.left
  } else {
    // 如果是右键点击，菜单显示在鼠标位置
    initialTop = event.clientY
    initialLeft = event.clientX
  }

  // 先设置初始位置，但不显示
  contextMenuStyle.value = {
    top: `${initialTop}px`,
    left: `${initialLeft}px`,
    visibility: 'hidden', // 先隐藏，计算完位置再显示
  }

  // 保存当前操作的图层
  activeLayer.value = { layer, index }

  // 显示菜单（此时还是隐藏的）
  contextMenuVisible.value = true

  // 等待DOM更新完成，以便获取菜单尺寸
  await nextTick()

  if (contextMenuRef.value) {
    const menuHeight = contextMenuRef.value.offsetHeight
    const windowHeight = window.innerHeight

    let finalTop = initialTop
    let finalLeft = initialLeft

    // 检查垂直方向是否溢出
    if (initialTop + menuHeight > windowHeight) {
      finalTop = initialTop - menuHeight
      // 如果向上调整后顶部溢出（菜单比视口高），则定位到视口顶部
      if (finalTop < 0) {
        finalTop = 5 // 留一点边距
      }
    }

    // 更新最终样式并显示菜单
    contextMenuStyle.value = {
      top: `${finalTop}px`,
      left: `${finalLeft}px`,
      visibility: 'visible',
    }
  } else {
    // 如果获取不到ref，使用初始位置并显示
    contextMenuStyle.value = {
      top: `${initialTop}px`,
      left: `${initialLeft}px`,
      visibility: 'visible',
    }
  }
}

// 处理上下文菜单命令
const handleContextMenuCommand = (command: string) => {
  if (activeLayer.value) {
    currentLayer.value = activeLayer.value.layer

    if (command === 'export') {
      exportLayer(activeLayer.value.layer)
    } else if (command === 'remove') {
      removeLayer(activeLayer.value.index)
    } else if (command === 'zoom') {
      zoomToLayer(activeLayer.value.layer)
    } else if (command === 'buffer') {
      // 触发缓冲区分析弹窗
      bufferAnalysisRef.value?.open(activeLayer.value.layer)
    } else if (command === 'edit' && activeLayer.value.layer.isEditable) {
      // 切换图层编辑状态
      activeLayer.value.layer.isEditing = !activeLayer.value.layer.isEditing
      // 关闭其他图层的编辑状态
      layers.value.forEach((l) => {
        if (l.id !== activeLayer.value?.layer?.id) {
          l.isEditing = false
        }
      })
      emit('edit-layer', activeLayer.value.layer)
    }

    // 隐藏菜单
    contextMenuVisible.value = false
  }
}

// 处理缓冲区分析完成事件
const handleBufferComplete = (bufferedLayer: LayerInfo) => {
  addGeoJsonLayer(bufferedLayer.name, bufferedLayer.data)
  emit('buffer-complete', bufferedLayer)
}

// 缩放至图层
const zoomToLayer = (layer: LayerInfo) => {
  emit('zoom-to-layer', layer)
  ElMessage.success(`已缩放至图层: ${layer.name}`)
}

// 点击其他地方关闭上下文菜单
const handleClickOutside = (event: MouseEvent) => {
  // 确保点击事件不是来自菜单按钮
  contextMenuVisible.value = false
}

// 挂载和卸载全局点击事件
onMounted(() => {
  document.addEventListener('click', handleClickOutside)
})

onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside)
})
</script>

<style scoped>
.layer-manager {
  height: 100%;
  width: 100%;
  display: flex;
  flex-direction: column;
  padding: 10px;
  box-sizing: border-box;
}

.layer-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.layer-header .el-dropdown {
  margin-left: auto;
}

.layer-header h3 {
  margin: 0;
}

.layer-list {
  flex: 1;
  overflow-y: auto;
  width: 100%;
}

.layer-item {
  margin-bottom: 10px;
  width: 100%;
  transition: background-color 0.3s;
}

.layer-item.editing-layer {
  background-color: #f0f8ff;
  border: 1px solid #409eff;
}

.layer-item.editing-layer .layer-info::before {
  content: '✎';
  margin-right: 8px;
  color: #409eff;
}

.layer-item-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.layer-info {
  display: flex;
  align-items: center;
  gap: 10px;
}

.layer-name {
  font-size: 14px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 150px;
}

.layer-actions {
  display: flex;
  gap: 5px;
}

/* 右键菜单样式 */
.context-menu {
  position: fixed;
  background: white;
  border-radius: 4px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  z-index: 9999;
  min-width: 120px;
}

.context-menu-item {
  padding: 8px 16px;
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.context-menu-item:hover {
  background-color: #f5f7fa;
}
</style>
