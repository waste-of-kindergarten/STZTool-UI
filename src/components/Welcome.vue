<template>
  <div style="height: 100vh; display: flex; justify-content: center; align-items: center; background-color: var(--el-fill-color-lighter);">
    
    <el-card shadow="hover" style="width: 600px; max-width: 90%;">
      
      <el-result
        icon="info"
        title="NAS 数据迁移助手"
        sub-title="基于 Vue 3 + ZimaOS 的可视化迁移工具"
      >
        <template #extra>
          
          <el-space direction="vertical" fill :size="30" style="width: 100%;">
            
            <el-alert
              title="准备工作"
              type="warning"
              :closable="false"
              description="请确保您已知晓源服务器(Synology)的管理员账号与密码。"
              show-icon
            />

            <el-steps :active="0" align-center finish-status="success">
              <el-step title="连接" />
              <el-step title="选择" />
              <el-step title="迁移" />
            </el-steps>

            <el-divider border-style="dashed">
              <el-icon><Connection /></el-icon>
            </el-divider>

            <el-button type="primary" size="large" round @click="handleStart">
              开始引导向导 <el-icon class="el-icon--right"><ArrowRight /></el-icon>
            </el-button>

          </el-space>
        </template>
      </el-result>

    </el-card>
  </div>
</template>

<script setup>
// 只要引入图标，其他样式全靠组件自带
import { Connection, ArrowRight } from '@element-plus/icons-vue'

const emit = defineEmits(['start'])

const handleStart = () => {
  emit('start')
}
</script>