<template>
  <el-container style="min-height: 100vh; background-color: var(--el-fill-color-lighter);">
    <el-main style="display: flex; flex-direction: column; align-items: center; padding-top: 40px;">
      
      <div style="text-align: center; margin-bottom: 30px;">
        <h2 style="color: var(--el-text-color-primary);">存储映射配置</h2>
        <p style="color: var(--el-text-color-secondary);">请为每个 Synology 存储卷选择一个 ZimaOS 目标硬盘</p>
      </div>

      <el-card shadow="never" style="width: 100%; max-width: 1000px;">
        
        <el-table :data="sourceList" stripe style="width: 100%">
          
          <el-table-column label="Synology 源存储卷" min-width="240">
            <template #default="scope">
              <div style="display: flex; flex-direction: column; gap: 5px;">
                <div style="display: flex; align-items: center; gap: 8px;">
                  <el-icon color="#409EFF"><FolderOpened /></el-icon>
                  <span style="font-weight: bold;">{{ scope.row.id }}</span>
                </div>
                <div style="font-size: 12px; color: #666;">
                  已用 {{ formatSize(scope.row.used) }} / 总 {{ formatSize(scope.row.size) }}
                </div>
                <el-progress 
                  :percentage="calcPercent(scope.row.used, scope.row.size)" 
                  :stroke-width="6"
                  :color="customColors"
                />
              </div>
            </template>
          </el-table-column>

          <el-table-column width="60" align="center">
            <el-icon size="20" color="#909399"><Right /></el-icon>
          </el-table-column>

          <el-table-column label="目标硬盘 (ZimaOS)" min-width="300">
            <template #default="scope">
              <el-form-item :error="validateSpace(scope.row, mappings[scope.row.id])" style="margin-bottom: 0;">
                <el-select 
                  v-model="mappings[scope.row.id]" 
                  placeholder="请选择目标硬盘" 
                  size="large"
                  style="width: 100%;"
                >
                  <el-option
                    v-for="disk in targetList"
                    :key="disk.path"
                    :label="disk.name"
                    :value="disk.path"
                  >
                    <div style="display: flex; justify-content: space-between; align-items: center; width: 100%;">
                      <span>{{ disk.name }}</span>
                      <span style="color: #8492a6; font-size: 12px;">
                        剩余 {{ getFreeSpace(disk) }}
                      </span>
                    </div>
                  </el-option>
                </el-select>
              </el-form-item>
              
              <div v-if="mappings[scope.row.id]" style="font-size: 12px; color: #888; margin-top: 5px;">
                目标路径: {{ mappings[scope.row.id] }}
              </div>
            </template>
          </el-table-column>
        </el-table>

        <div style="margin-top: 30px; display: flex; justify-content: space-between; align-items: center;">
          <el-button @click="$emit('back')">返回重连</el-button>
          
          <el-space>
            <div style="text-align: right; margin-right: 10px; font-size: 13px; color: #666;">
               <div v-if="!isAllSelected">请完成所有映射</div>
               <div v-else style="color: #67C23A;">已准备就绪</div>
            </div>
            <el-button type="primary" size="large" @click="handleStart" :disabled="!isAllSelected">
              确认并开始迁移
            </el-button>
          </el-space>
        </div>

      </el-card>
    </el-main>
  </el-container>
</template>

<script setup>
import { ref, computed } from 'vue'
import { FolderOpened, Right } from '@element-plus/icons-vue'
import { ElMessageBox } from 'element-plus'

const props = defineProps({
  storageData: {
    type: Object,
    required: true
  }
})

const emit = defineEmits(['back', 'start'])

// 数据源
const sourceList = props.storageData.source_volumes || []
const targetList = props.storageData.target_disks || []

// 存储映射关系: key=群晖路径, value=Zima路径
const mappings = ref({})

// --- 工具函数: 字节转换 ---
const formatSize = (bytes) => {
  if (bytes === 0) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB', 'TB', 'PB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

// --- 工具函数: 计算百分比 ---
const calcPercent = (used, total) => {
  if (!total) return 0
  return Math.min(Math.round((used / total) * 100), 100)
}

// --- 工具函数: 获取 Zima 盘剩余空间 ---
const getFreeSpace = (disk) => {
  // 剩余 = 总大小 - 已用
  const free = disk.extensions.size - disk.extensions.used
  return formatSize(free)
}

// --- 逻辑: 简单验证 (仅提示，不强制阻止) ---
const validateSpace = (sourceVol, targetPath) => {
  if (!targetPath) return null
  
  // 找到目标盘对象
  const targetDisk = targetList.find(d => d.path === targetPath)
  if (!targetDisk) return null

  const freeBytes = targetDisk.extensions.size - targetDisk.extensions.used
  // 如果源卷已用空间 > 目标盘剩余空间，返回警告文字
  if (sourceVol.used > freeBytes) {
    return `⚠️ 空间可能不足 (需 ${formatSize(sourceVol.used)}, 剩 ${formatSize(freeBytes)})`
  }
  return null
}

// 进度条颜色
const customColors = [
  { color: '#67C23A', percentage: 60 },
  { color: '#E6A23C', percentage: 85 },
  { color: '#F56C6C', percentage: 100 },
]

// 是否全部选完了
const isAllSelected = computed(() => {
  return sourceList.every(vol => mappings.value[vol.id])
})

const handleStart = () => {
    // console.log(mappings.value)
  ElMessageBox.confirm(
    '确定映射关系无误吗？任务开始后将进行数据同步。',
    '确认迁移',
    { confirmButtonText: '开始', cancelButtonText: '取消', type: 'info' }
  ).then(() => {
    emit('start', mappings.value)
  })
}
</script>