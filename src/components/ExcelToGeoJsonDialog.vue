<template>
  <el-dialog :model-value="visible" title="导入表图层" width="500px" @close="handleClose">
    <el-upload
      class="excel-upload"
      drag
      action=""
      :auto-upload="false"
      :show-file-list="false"
      accept=".xlsx,.xls"
      :on-change="handleFileChange"
    >
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">将Excel文件拖到此处，或<em>点击上传</em></div>
    </el-upload>
    <div v-if="fields.length > 0" style="margin-top: 20px;">
      <el-form :model="form">
        <el-form-item label="图层类型">
          <el-select v-model="form.layerType" placeholder="请选择图层类型">
            <el-option label="点图层" value="point" />
            <el-option label="圆图层" value="circle" />
            <el-option label="扇形面图层" value="sector" />
          </el-select>
        </el-form-item>
        <template v-if="form.layerType === 'point'">
          <el-form-item label="经度字段">
            <el-select v-model="form.lonField" placeholder="请选择经度字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
          <el-form-item label="纬度字段">
            <el-select v-model="form.latField" placeholder="请选择纬度字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
        </template>
        <template v-if="form.layerType === 'circle'">
          <el-form-item label="经度字段">
            <el-select v-model="form.lonField" placeholder="请选择经度字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
          <el-form-item label="纬度字段">
            <el-select v-model="form.latField" placeholder="请选择纬度字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
          <el-form-item label="半径字段">
            <el-select v-model="form.radiusField" placeholder="请选择半径字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
        </template>
        <template v-if="form.layerType === 'sector'">
          <el-form-item label="经度字段">
            <el-select v-model="form.lonField" placeholder="请选择经度字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
          <el-form-item label="纬度字段">
            <el-select v-model="form.latField" placeholder="请选择纬度字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
          <el-form-item label="半径字段">
            <el-select v-model="form.radiusField" placeholder="请选择半径字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
          <el-form-item label="起始角度字段">
            <el-select v-model="form.startAngleField" placeholder="请选择起始角度字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
          <el-form-item label="结束角度字段">
            <el-select v-model="form.endAngleField" placeholder="请选择结束角度字段">
              <el-option v-for="f in fields" :key="f" :label="f" :value="f" />
            </el-select>
          </el-form-item>
        </template>
      </el-form>
    </div>
    <template #footer>
      <el-button @click="handleClose">取消</el-button>
      <el-button type="primary" :disabled="!canImport" @click="handleImport">导入</el-button>
    </template>
  </el-dialog>
</template>

<script setup lang="ts">
import { ref, computed, reactive, watch, defineEmits, nextTick } from 'vue'
import * as XLSX from 'xlsx'
import * as turf from '@turf/turf'
import { ElMessage } from 'element-plus'

const props = defineProps<{ visible: boolean }>()
const emit = defineEmits(['close', 'import'])

const fields = ref<string[]>([])
const excelData = ref<any[]>([])
const form = reactive({
  layerType: '',
  lonField: '',
  latField: '',
  radiusField: '',
  startAngleField: '',
  endAngleField: '',
})

const canImport = computed(() => {
  if (!excelData.value.length) return false
  if (form.layerType === 'point') {
    return form.lonField && form.latField
  } else if (form.layerType === 'sector') {
    return (
      form.lonField &&
      form.latField &&
      form.radiusField &&
      form.startAngleField &&
      form.endAngleField
    )
  } else if (form.layerType === 'circle') {
    return form.lonField && form.latField && form.radiusField
  }
  return false
})

const handleFileChange = (file: any) => {
  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const data = new Uint8Array(e.target?.result as ArrayBuffer)
      const workbook = XLSX.read(data, { type: 'array' })
      const sheetName = workbook.SheetNames[0]
      const worksheet = workbook.Sheets[sheetName]
      const json = XLSX.utils.sheet_to_json(worksheet)
      excelData.value = json
      fields.value = Object.keys(json[0] || {})
      Object.assign(form, {
        layerType: '',
        lonField: '',
        latField: '',
        radiusField: '',
        startAngleField: '',
        endAngleField: '',
      })
    } catch (err) {
      ElMessage.error('Excel文件解析失败')
    }
  }
  reader.readAsArrayBuffer(file.raw)
}

const handleImport = () => {
  let geojson: any = null
  if (form.layerType === 'point') {
    geojson = {
      type: 'FeatureCollection',
      features: excelData.value.map((row: any) => ({
        type: 'Feature',
        geometry: {
          type: 'Point',
          coordinates: [Number(row[form.lonField]), Number(row[form.latField])],
        },
        properties: { ...row },
      })),
    }
  } else if (form.layerType === 'sector') {
    geojson = {
      type: 'FeatureCollection',
      features: excelData.value.map((row: any) => {
        const center = [Number(row[form.lonField]), Number(row[form.latField])]
        const radius = Number(row[form.radiusField])
        const startAngle = Number(row[form.startAngleField])
        const endAngle = Number(row[form.endAngleField])
        const sector = turf.sector(center, radius, startAngle, endAngle, { steps: 64, units: 'kilometers' })
        return {
          type: 'Feature',
          geometry: sector.geometry,
          properties: { ...row },
        }
      }),
    }
  } else if (form.layerType === 'circle') {
    geojson = {
      type: 'FeatureCollection',
      features: excelData.value.map((row: any) => {
        const center = [Number(row[form.lonField]), Number(row[form.latField])]
        const radius = Number(row[form.radiusField])
        const circle = turf.circle(center, radius, {
          steps: 64,
          units: 'kilometers',
          properties: { ...row }
        })
        return {
          type: 'Feature',
          geometry: circle.geometry,
          properties: { ...row },
        }
      })
    }
  }
  if (geojson) {
    emit('import', geojson)
    handleClose()
  }
}

const handleClose = () => {
  emit('close')
  emit('update:visible', false)
  nextTick(() => {
    fields.value = []
    excelData.value = []
    Object.assign(form, {
      layerType: '',
      lonField: '',
      latField: '',
      radiusField: '',
      startAngleField: '',
      endAngleField: '',
    })
  })
}
</script>

<style scoped>
.excel-upload {
  width: 100%;
  margin-bottom: 10px;
}
</style>