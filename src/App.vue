<template>
  <el-config-provider>
    
    <Welcome 
      v-if="currentStep === 'welcome'" 
      @start="goToConnection" 
    />
    
    <ConnectionConfig 
      v-else-if="currentStep === 'connection'"
      @back="goBackToWelcome"
      @next="handleStorageDataFetched"
    />

    <VolumeMapping 
      v-else-if="currentStep === 'mapping'"
      :storageData="storageData"
      @back="goToConnection"
      @start="handleStartMigration"
    />

    <MigrationMonitor 
      v-else-if="currentStep === 'monitor'" 
      :task-data="finalMapping"
      @back="goBackToWelcome"
      @viewReport="handleViewReport"
    />

    <MigrationReport 
      v-else-if="currentStep === 'report'"
      :reportData="reportData"
      @back="goBackToWelcome"
      @close="goBackToWelcome"
    />

  </el-config-provider>
</template>

<script setup>
import { ref } from 'vue'
import { ElMessage } from 'element-plus'
import Welcome from './components/Welcome.vue'
import ConnectionConfig from './components/ConnectionConfig.vue'
import VolumeMapping from './components/VolumeMapping.vue' 
import MigrationMonitor from './components/MigrationMonitor.vue'
import MigrationReport from './components/MigrationReport.vue' // ✅ [新增] 引入报告组件

// 状态管理
// 流程：'welcome' -> 'connection' -> 'mapping' -> 'monitor' -> 'report'
const currentStep = ref('welcome')
const storageData = ref({})  
const finalMapping = ref({}) 
const reportData = ref({}) // ✅ [新增] 用于存储报告数据

// 切换到连接页
const goToConnection = () => {
  currentStep.value = 'connection'
}

// 返回欢迎页 (重置)
const goBackToWelcome = () => {
  currentStep.value = 'welcome'
  // 可选：如果希望返回首页时清空数据，可以在这里重置 reportData 或 finalMapping
}

// Step 2 -> Step 3: 接收存储数据 -> 跳转映射页
const handleStorageDataFetched = (data) => {
  storageData.value = data
  currentStep.value = 'mapping' 
  console.log('已获取存储列表:', data)
}

// Step 3 -> Step 4: 向后端发送指令并跳转
const handleStartMigration = async (mappingResult) => {
  finalMapping.value = mappingResult
  console.log('最终映射配置:', mappingResult)

  const loadingMsg = ElMessage({
    message: '正在向后端下发任务...',
    type: 'info',    
    duration: 0,     
    grouping: true,
  })

  try {
    const currentIp = window.location.hostname
    const baseUrl = `http://${currentIp}:5175`

    const response = await fetch(`${baseUrl}/migrate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        mappings: mappingResult
      })
    })

    loadingMsg.close()

    if (response.ok) {
      ElMessage.success('任务启动成功！')
      currentStep.value = 'monitor'
    } else {
      const errText = await response.text()
      ElMessage.error(`启动失败: ${errText}`)
    }

  } catch (error) {
    loadingMsg.close()
    console.error(error)
    ElMessage.error('无法连接后端服务，请检查网络')
  }
}

// ✅ [新增] Step 4 -> Step 5: 接收监控页传来的报告数据并跳转
const handleViewReport = (data) => {
  console.log("收到报告数据:", data)
  reportData.value = data // 保存数据传递给 MigrationReport 组件
  currentStep.value = 'report' // 切换视图
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
  font-family: var(--el-font-family);
}
</style>