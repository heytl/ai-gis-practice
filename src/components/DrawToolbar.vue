<template>
  <div class="draw-toolbar" v-if="visible">
    <el-button-group>
      <el-button
        :type="currentTool === 'Point' ? 'primary' : 'default'"
        :icon="Location"
        @click="selectTool('Point')"
        title="绘制点"
      />
      <el-button
        :type="currentTool === 'LineString' ? 'primary' : 'default'"
        :icon="Connection"
        @click="selectTool('LineString')"
        title="绘制线"
      />
      <el-button
        :type="currentTool === 'Polygon' ? 'primary' : 'default'"
        :icon="Crop"
        @click="selectTool('Polygon')"
        title="绘制面"
      />
    </el-button-group>
    <el-button type="danger" :icon="Close" @click="stopDrawing" title="停止绘制" />
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue'
import { Location, Connection, Crop, Close } from '@element-plus/icons-vue'

const props = defineProps<{
  visible: boolean
}>()

const emit = defineEmits(['tool-change', 'stop-drawing'])

const currentTool = ref<string>('')

// 选择绘制工具
const selectTool = (tool: string) => {
  currentTool.value = tool
  emit('tool-change', tool)
}

// 停止绘制
const stopDrawing = () => {
  currentTool.value = ''
  emit('stop-drawing')
}

// 监听visible变化，当隐藏时重置当前工具
watch(
  () => props.visible,
  (newValue) => {
    if (!newValue) {
      currentTool.value = ''
    }
  },
)
</script>

<style scoped>
.draw-toolbar {
  position: absolute;
  top: 20px;
  right: 20px;
  background-color: white;
  padding: 5px;
  border-radius: 4px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  display: flex;
  gap: 5px;
  z-index: 1;
}
</style>
