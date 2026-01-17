<template>
  <el-container style="min-height: 100vh; display: flex; justify-content: center; align-items: center; background-color: var(--el-fill-color-lighter); padding: 40px;">
    
    <el-card shadow="always" style="width: 100%; max-width: 900px;">
      
      <div style="text-align: center; padding: 30px 0; border-bottom: 1px solid #ebeef5;">
        <el-icon :size="80" :color="hasErrors ? '#E6A23C' : '#67C23A'">
          <component :is="hasErrors ? 'WarningFilled' : 'SuccessFilled'" />
        </el-icon>
        <h1 style="margin: 20px 0 10px; font-size: 28px; color: #303133;">
          {{ hasErrors ? '迁移完成 (有异常)' : '迁移完美完成' }}
        </h1>
        <p style="color: #909399; margin: 0;">
          {{ hasErrors ? `发现 ${reportData.errors.length} 个文件传输失败，请检查下方日志` : '所有数据已安全传输至 ZimaOS' }}
        </p>
      </div>

      <div style="padding: 30px;">
        <el-row :gutter="20">
          <el-col :span="8">
            <div class="stat-card">
              <div class="label">总数据量 (实际传输)</div>
              <div class="value" style="color: #409EFF">{{ formattedSize }}</div>
            </div>
          </el-col>
          <el-col :span="8">
            <div class="stat-card">
              <div class="label">总耗时</div>
              <div class="value">{{ formattedDuration }}</div>
            </div>
          </el-col>
          <el-col :span="8">
            <div class="stat-card">
              <div class="label">平均速度</div>
              <div class="value" style="color: #67C23A">{{ avgSpeed }}</div>
            </div>
          </el-col>
        </el-row>
      </div>

      <div v-if="hasErrors" style="padding: 0 30px 30px;">
        <el-divider content-position="left"><span style="color: #F56C6C; font-weight: bold;">❌ 失败文件列表</span></el-divider>
        <el-table :data="errorTableData" style="width: 100%" height="250" border stripe size="small">
          <el-table-column prop="msg" label="错误详情" />
        </el-table>
      </div>

      <div style="background: #f5f7fa; padding: 20px; text-align: right; display: flex; justify-content: flex-end; gap: 10px;">
        <el-button @click="$emit('back')">返回首页</el-button>
        </div>

    </el-card>
  </el-container>
</template>

<script setup>
import { computed } from 'vue'
import { SuccessFilled, WarningFilled } from '@element-plus/icons-vue'

// 1. 接收原始数据 (确保 key 与 MigrationMonitor 传出来的一致)
const props = defineProps({
  reportData: {
    type: Object,
    required: true,
    default: () => ({
      totalSize: 0,   // 接收 Int64 (Bytes)
      uploadTime: 0,  // 接收 Int64 (Seconds)
      errors: []      // 接收 Array
    })
  }
})

const emit = defineEmits(['back', 'close'])

// 2. 辅助逻辑：是否有错误
const hasErrors = computed(() => {
  return props.reportData.errors && props.reportData.errors.length > 0
})

// 3. 辅助逻辑：错误列表转表格格式
const errorTableData = computed(() => {
  return (props.reportData.errors || []).map(err => ({ msg: err }))
})

// ==========================================
// ⬇️ 核心：在这里进行格式化计算 (Frontend Logic) ⬇️
// ==========================================

// 格式化文件大小 (Bytes -> GB/MB)
const formattedSize = computed(() => {
  // 注意：这里读取的是 props.reportData.totalSize
  // 如果你在 Monitor 里存的是 uploadFilesTotal，请确保传进来时名字对上了，或者这里改成 uploadFilesTotal
  const bytes = props.reportData.totalSize
  if (!bytes || bytes === 0) return '0 B'
  
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB', 'TB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
})

// 格式化时间 (Seconds -> HHh MMm SSs)
const formattedDuration = computed(() => {
  const seconds = props.reportData.uploadTime
  if (!seconds || seconds === 0) return '0s'
  
  const h = Math.floor(seconds / 3600)
  const m = Math.floor((seconds % 3600) / 60)
  const s = Math.floor(seconds % 60)
  
  let result = ''
  if (h > 0) result += `${h}h `
  if (m > 0) result += `${m}m `
  result += `${s}s`
  return result
})

// 计算平均速度 (Size / Time)
const avgSpeed = computed(() => {
  const bytes = props.reportData.totalSize
  const seconds = props.reportData.uploadTime
  
  if (!bytes || !seconds || seconds <= 0) return '0 MB/s'
  
  const mb = bytes / 1024 / 1024
  const speed = mb / seconds
  
  return speed.toFixed(2) + ' MB/s'
})

</script>

<style scoped>
.stat-card {
  text-align: center;
  padding: 20px;
  background: #fff;
  border-radius: 8px;
  border: 1px solid #ebeef5;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.05);
  transition: all 0.3s;
}
.stat-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px 0 rgba(0, 0, 0, 0.1);
}
.label {
  font-size: 14px;
  color: #909399;
  margin-bottom: 10px;
}
.value {
  font-size: 24px;
  font-weight: bold;
  color: #303133;
}
</style>