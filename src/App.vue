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
    />

  </el-config-provider>
</template>

<script setup>
import { ref } from 'vue'
import { ElMessage } from 'element-plus' // 记得引入这个用于提示
import Welcome from './components/Welcome.vue'
import ConnectionConfig from './components/ConnectionConfig.vue'
import VolumeMapping from './components/VolumeMapping.vue' 
import MigrationMonitor from './components/MigrationMonitor.vue' // [新增] 引入新组件

// 状态管理
// 流程：'welcome' -> 'connection' -> 'mapping' -> 'monitor'
const currentStep = ref('welcome')
const storageData = ref({})  
const finalMapping = ref({}) 

// 切换到连接页
const goToConnection = () => {
  currentStep.value = 'connection'
}

// 返回欢迎页
const goBackToWelcome = () => {
  currentStep.value = 'welcome'
}

// Step 2 -> Step 3: 接收存储数据 -> 跳转映射页
const handleStorageDataFetched = (data) => {
  storageData.value = data
  currentStep.value = 'mapping' 
  console.log('已获取存储列表:', data)
}

// [修改] Step 3 -> Step 4: 向后端发送指令并跳转
const handleStartMigration = async (mappingResult) => {
  // 1. 保存映射结果
  finalMapping.value = mappingResult
  console.log('最终映射配置:', mappingResult)

  // 2. 显示 Loading 提示
  // ❌ 错误写法 (Element Plus 没有这个方法): 
  // const loadingMsg = ElMessage.loading({ ... })

  // ✅ 正确写法: 直接调用 ElMessage，设置 duration: 0 (不自动关闭)
  const loadingMsg = ElMessage({
    message: '正在向后端下发任务...',
    type: 'info',    // 使用 info 类型
    duration: 0,     // 0 表示不会自动消失，直到调用 .close()
    grouping: true,
  })

  try {
    const currentIp = window.location.hostname
    const baseUrl = `http://${currentIp}:5175`

    // 3. 发送 POST 请求给后端
    const response = await fetch(`${baseUrl}/migrate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        mappings: mappingResult
      })
    })

    // 手动关闭 Loading 提示
    loadingMsg.close()

    if (response.ok) {
      // 4. 成功！跳转到监控页
      ElMessage.success('任务启动成功！')
      currentStep.value = 'monitor'
    } else {
      // 失败：读取错误信息
      const errText = await response.text()
      ElMessage.error(`启动失败: ${errText}`)
    }

  } catch (error) {
    // 发生异常也要关闭 Loading
    loadingMsg.close()
    console.error(error)
    ElMessage.error('无法连接后端服务，请检查网络')
  }
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
  font-family: var(--el-font-family);
}
</style>