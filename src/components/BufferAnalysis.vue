<template>
  <el-dialog v-model="dialogVisible" title="缓冲区分析" width="30%">
    <el-form label-width="100px">
      <el-form-item label="缓冲半径">
        <el-input-number v-model="bufferRadius" :min="0" :step="1" />
        <span style="margin-left: 8px">公里</span>
      </el-form-item>
      <el-form-item label="融合缓冲区">
        <el-checkbox v-model="mergeBuffers">合并重叠区域</el-checkbox>
      </el-form-item>
    </el-form>

    <template #footer>
      <el-button @click="dialogVisible = false">取消</el-button>
      <el-button type="primary" @click="handleConfirm">开始执行</el-button>
    </template>
  </el-dialog>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import * as turf from '@turf/turf'
import { ElMessage } from 'element-plus'

const dialogVisible = ref(false)
const bufferRadius = ref(10)
const mergeBuffers = ref(true)

const props = defineProps<{
  currentLayer?: LayerInfo
}>()

const currentLayer = ref<LayerInfo | null>(props.currentLayer || null)

const emit = defineEmits(['buffer-complete'])

const open = (layerData: LayerInfo) => {
  currentLayer.value = layerData
  dialogVisible.value = true
}

const mergeBufferFeatures = (layer) => {
  try {
    if (layer) {
      return turf.union(layer)
    }
    return layer
  } catch (error) {
    console.error('Buffer merge error:', error)
    return layer
  }
}

const handleConfirm = async () => {
  try {
    if (!currentLayer.value) return

    let resultGeometry = turf.buffer(currentLayer.value.data, bufferRadius.value, {
      units: 'kilometers',
    })

    if (mergeBuffers.value) {
      resultGeometry = mergeBufferFeatures(resultGeometry)
    }

    emit('buffer-complete', {
      data: resultGeometry,
      name: `${currentLayer.value.name}_缓冲区_${bufferRadius.value}km${mergeBuffers.value ? '_融合' : ''}`,
    })

    ElMessage.success('缓冲区分析完成')
    dialogVisible.value = false
  } catch (error) {
    ElMessage.error('缓冲区分析失败')
    console.error('Buffer analysis error:', error)
  }
}

defineExpose({ open })
</script>

<style scoped>
.el-input-number {
  width: 200px;
}
</style>
