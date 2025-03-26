<template>
  <el-dialog v-model="dialogVisible" title="图层名称" width="30%">
    <el-form label-width="100px">
      <el-form-item label="图层名称">
        <el-input v-model="layerName" placeholder="请输入图层名称" />
      </el-form-item>
    </el-form>

    <template #footer>
      <el-button @click="dialogVisible = false">取消</el-button>
      <el-button type="primary" @click="handleConfirm">确定</el-button>
    </template>
  </el-dialog>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { ElMessage } from 'element-plus'

const dialogVisible = ref(false)
const layerName = ref('')
const defaultName = ref('')

// 定义事件
const emit = defineEmits(['confirm'])

// 打开对话框
const open = (defaultLayerName: string = '') => {
  defaultName.value = defaultLayerName
  layerName.value = defaultLayerName
  dialogVisible.value = true
}

// 处理确认按钮点击
const handleConfirm = () => {
  if (!layerName.value.trim()) {
    ElMessage.warning('图层名称不能为空')
    return
  }

  emit('confirm', layerName.value)
  dialogVisible.value = false
}

// 暴露方法给父组件
defineExpose({ open })
</script>

<style scoped>
.el-input {
  width: 100%;
}
</style>
