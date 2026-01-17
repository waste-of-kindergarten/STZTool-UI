<template>
  <el-container style="min-height: 100vh; display: flex; justify-content: center; align-items: center; background-color: var(--el-fill-color-lighter); padding: 40px;">
    
    <el-card shadow="hover" style="width: 100%; max-width: 1200px; text-align: center;">
      
      <template #header>
        <div style="display: flex; justify-content: space-between; align-items: center; padding: 0 10px;">
          <div style="display: flex; align-items: center; gap: 10px;">
            <span style="font-weight: bold; font-size: 20px;">数据迁移任务监控</span>
            <el-tag v-if="status === 'running'" type="primary" effect="dark">运行中</el-tag>
            <el-tag v-else-if="status === 'finished'" type="success" effect="dark">已完成</el-tag>
          </div>
          <div style="font-size: 13px; color: #909399;">
             <span v-if="status === 'running'"><el-icon class="is-loading"><Loading /></el-icon> 实时同步中...</span>
          </div>
        </div>
      </template>

      <el-collapse accordion style="margin-bottom: 40px; text-align: left;">
        <el-collapse-item name="1">
          <template #title>
             <span style="font-size: 14px; font-weight: bold; color: #606266;">📋 点击查看详细映射列表</span>
          </template>
          <div v-for="(target, source) in taskData" :key="source" style="padding: 12px 20px; border-bottom: 1px dashed #eee; display: flex; align-items: center; justify-content: space-between;">
            <div style="display: flex; align-items: center; width: 45%;">
              <el-tag size="default" type="info">源 (Synology)</el-tag>
              <span style="margin-left: 15px; font-size: 14px; color: #303133; font-weight: 500;">{{ source }}</span>
            </div>
            <el-icon style="color: #C0C4CC; font-size: 20px;"><Right /></el-icon>
            <div style="display: flex; align-items: center; justify-content: flex-end; width: 45%;">
              <span style="margin-right: 15px; font-size: 14px; color: #303133; font-weight: 500;">{{ target }}</span>
              <el-tag size="default" type="success">目标 (ZimaOS)</el-tag>
            </div>
          </div>
        </el-collapse-item>
      </el-collapse>

      <div style="display: flex; justify-content: center; padding: 30px 0; background: linear-gradient(to bottom, #fff, #fdfdfd);">
        <el-progress 
          type="dashboard" 
          :percentage="percentage" 
          :color="colors"
          :width="280" 
          :stroke-width="15"
        >
          <template #default="{ percentage }">
            <span class="percentage-value">{{ percentage }}%</span>
            <div class="percentage-label">{{ statusText }}</div>
          </template>
        </el-progress>
      </div>

      <div style="background: #2b2b2b; color: #eee; padding: 25px; border-radius: 8px; text-align: left; margin: 30px 0; box-shadow: 0 4px 12px rgba(0,0,0,0.1);">
        <div style="display: flex; justify-content: space-between; margin-bottom: 15px; border-bottom: 1px solid #444; padding-bottom: 10px;">
          <span style="font-size: 14px; font-weight: bold; color: #fff;">🚀 传输控制台</span>
          <span style="font-size: 13px; color: #67C23A;" v-if="status === 'running'">
             {{ message }}
          </span>
          <span style="font-size: 13px; color: #67C23A;" v-else-if="status === 'finished'">
            ✅ 任务结束
          </span>
        </div>

        <div style="font-size: 12px; color: #aaa; margin-bottom: 8px;">> current_file_processing:</div>
        <div style="font-family: 'Consolas', 'Monaco', monospace; color: #409EFF; font-size: 14px; line-height: 1.6; word-break: break-all;">
          {{ currentFile || 'waiting for stream...' }}<span class="blinking-cursor" v-if="status === 'running'">_</span>
        </div>
      </div>

      <div v-if="status === 'finished'" style="padding: 20px 0;">
        <el-button type="primary" size="large" style="width: 250px; height: 50px; font-size: 16px;" @click="$emit('back')">完成并返回</el-button>
      </div>
      <div v-else style="padding: 20px 0;">
        <el-button type="info" plain disabled size="default">正在执行后台迁移任务，请保持页面开启</el-button>
      </div>

    </el-card>

  </el-container>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import { Right, Loading } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'

const props = defineProps({
  taskData: {
    type: Object,
    default: () => ({})
  }
})

const emit = defineEmits(['back'])

const status = ref('idle')        
const percentage = ref(0)
const currentFile = ref('')
const message = ref('Initializing...')
let timer = null 

const colors = [
  { color: '#f56c6c', percentage: 20 },
  { color: '#e6a23c', percentage: 40 },
  { color: '#5cb87a', percentage: 100 },
]

const statusText = computed(() => {
  if (status.value === 'finished') return 'Done'
  if (status.value === 'error') return 'Error'
  return 'Syncing'
})

const fetchProgress = async () => {
  try {
    const currentIp = window.location.hostname
    const res = await fetch(`http://${currentIp}:5175/progress`)
    
    if (!res.ok) throw new Error("Connection failed")
    
    const data = await res.json()
    
    status.value = data.status
    percentage.value = data.percentage
    currentFile.value = data.current_file
    message.value = data.message

    if (data.status === 'finished' || data.status === 'error') {
      clearInterval(timer)
      if (data.status === 'finished') {
        ElMessage.success("All tasks completed!")
      }
    }
  } catch (error) {
    console.warn("Polling failed:", error)
  }
}

onMounted(() => {
  fetchProgress()
  timer = setInterval(fetchProgress, 500)
})

onUnmounted(() => {
  if (timer) clearInterval(timer)
})
</script>

<style scoped>
.percentage-value {
  display: block;
  font-size: 42px; /* 超大字体 */
  font-weight: 800;
  color: #303133;
}
.percentage-label {
  display: block;
  font-size: 16px;
  margin-top: 8px;
  color: #909399;
  text-transform: uppercase;
  letter-spacing: 1px;
}
/* 光标闪烁动画 */
.blinking-cursor {
  animation: blink 1s step-end infinite;
}
@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}
</style>