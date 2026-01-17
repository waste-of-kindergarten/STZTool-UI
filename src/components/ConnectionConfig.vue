<template>
  <el-container style="min-height: 100vh; background-color: var(--el-fill-color-lighter);">
    <el-main style="display: flex; flex-direction: column; justify-content: center; align-items: center;">
      
      <div style="text-align: center; margin-bottom: 40px;">
        <h2 style="color: var(--el-text-color-primary);">建立连接</h2>
        <p style="color: var(--el-text-color-secondary); font-size: 14px;">请输入源服务器(Synology)和目标服务器(ZimaOS)的凭据</p>
      </div>

      <div style="width: 100%; max-width: 1000px;">
        <el-form 
          ref="formRef"
          :model="formData" 
          :rules="rules" 
          label-position="top"
          size="large"
        >
          <el-row :gutter="40">
            <el-col :xs="24" :md="12" style="margin-bottom: 20px;">
              <el-card shadow="hover">
                <template #header>
                  <div style="display: flex; align-items: center; gap: 10px;">
                    <el-icon size="20" color="#409EFF"><Platform /></el-icon>
                    <span style="font-weight: bold;">Synology (源)</span>
                  </div>
                </template>

                <el-form-item label="SSH Host (IP地址)" prop="synology.host">
                  <el-input v-model="formData.synology.host" placeholder="例如: 192.168.1.100" clearable />
                </el-form-item>

                <el-row :gutter="15">
                  <el-col :span="16">
                    <el-form-item label="用户名" prop="synology.username">
                      <el-input v-model="formData.synology.username" placeholder="root / admin" />
                    </el-form-item>
                  </el-col>
                  
                  <el-col :span="8">
                    <el-form-item label="SSH 端口" prop="synology.port">
                      <el-input-number 
                        v-model="formData.synology.port" 
                        :min="1" 
                        :max="65535" 
                        :controls="false" 
                        placeholder="22"
                        style="width: 100%; text-align: left;" 
                      />
                    </el-form-item>
                  </el-col>
                  </el-row>

                <el-form-item label="密码" prop="synology.password">
                  <el-input 
                    v-model="formData.synology.password" 
                    type="password" 
                    placeholder="请输入密码" 
                    show-password 
                  />
                </el-form-item>
              </el-card>
            </el-col>

            <el-col :xs="24" :md="12" style="margin-bottom: 20px;">
              <el-card shadow="hover">
                <template #header>
                  <div style="display: flex; align-items: center; gap: 10px;">
                    <el-icon size="20" color="#67C23A"><Monitor /></el-icon>
                    <span style="font-weight: bold;">ZimaOS (目标)</span>
                  </div>
                </template>

                <el-form-item label="Host (IP地址)" prop="zima.host">
                  <el-input v-model="formData.zima.host" placeholder="例如: 10.0.0.5" clearable />
                </el-form-item>

                <el-form-item label="用户名" prop="zima.username">
                  <el-input v-model="formData.zima.username" placeholder="admin" />
                </el-form-item>

                <el-form-item label="密码" prop="zima.password">
                  <el-input 
                    v-model="formData.zima.password" 
                    type="password" 
                    placeholder="请输入密码" 
                    show-password 
                  />
                </el-form-item>
                
                <el-alert title="将使用默认 API 端口进行连接" type="info" :closable="false" style="margin-top: 50px;" />

              </el-card>
            </el-col>
          </el-row>

          <div style="margin-top: 30px; text-align: center;">
            <el-space :size="20">
              <el-button @click="$emit('back')" icon="ArrowLeft">上一步</el-button>
              <el-button type="primary" :loading="loading" @click="handleConnect(formRef)" icon="Connection">
                迁移配置
              </el-button>
            </el-space>
          </div>

        </el-form>
      </div>

    </el-main>
  </el-container>
</template>

<script setup>
import { reactive, ref } from 'vue'
import { Platform, Monitor, Connection, ArrowLeft } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'

const emit = defineEmits(['next', 'back'])
const formRef = ref()
const loading = ref(false)

const formData = reactive({
  synology: {
    host: '',
    port: 22,
    username: '',
    password: ''
  },
  zima: {
    host: '',
    username: '',
    password: ''
  }
})

const rules = {
  'synology.host': [{ required: true, message: '请输入群晖 IP 地址', trigger: 'blur' }],
  'synology.username': [{ required: true, message: '请输入群晖用户名', trigger: 'blur' }],
  'synology.password': [{ required: true, message: '请输入群晖密码', trigger: 'blur' }],
  'zima.host': [{ required: true, message: '请输入 ZimaOS IP 地址', trigger: 'blur' }],
  'zima.username': [{ required: true, message: '请输入 ZimaOS 用户名', trigger: 'blur' }],
  'zima.password': [{ required: true, message: '请输入 ZimaOS 密码', trigger: 'blur' }]
}

const handleConnect = async (formEl) => {
  if (!formEl) return
  
  await formEl.validate(async (valid) => {
    if (valid) {
      loading.value = true
      
      try {
        // 1. 获取当前浏览器地址栏 IP (适配 Docker Host 网络)
        const currentIp = window.location.hostname
        const baseUrl = `http://${currentIp}:5175`

        console.log(`正在连接后端: ${baseUrl}`)

        // === 阶段一: 发送配置 ===
        const configRes = await fetch(`${baseUrl}/config`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(formData)
        })

        if (!configRes.ok) throw new Error(`配置验证失败: ${configRes.status}`)

        ElMessage.success('配置验证通过，正在扫描存储设备...')

        // === 阶段二: 并行获取两端存储信息 ===
        // 使用 Promise.all 同时发起两个请求，速度更快
        const [synoRes, zimaRes] = await Promise.all([
          fetch(`${baseUrl}/storage/synology`),
          fetch(`${baseUrl}/storage/zima`)
        ])

        if (!synoRes.ok) throw new Error("无法获取群晖存储列表")
        if (!zimaRes.ok) throw new Error("无法获取 ZimaOS 硬盘列表")

        const synoData = await synoRes.json()
        const zimaData = await zimaRes.json()

        // 组装数据
        const storageInfo = {
          source_volumes: synoData, // 结构: [{id, size, used}, ...]
          target_disks: zimaData    // 结构: [{name, path, extensions: {size, used...}}, ...]
        }

        console.log("存储扫描完成:", storageInfo)
        
        // 触发下一步，把这一大包数据传给父组件
        emit('next', storageInfo)

      } catch (error) {
        console.error(error)
        ElMessage.error(error.message || '连接过程中发生错误')
      } finally {
        loading.value = false
      }
    } else {
      ElMessage.error('请填写完整所有必填项')
    }
  })
}
</script>

<style scoped>
/* 强制让 input-number 内部的文字左对齐，防止数字因为 padding 太大而看不见 */
:deep(.el-input-number .el-input__inner) {
  text-align: left;
}
</style>