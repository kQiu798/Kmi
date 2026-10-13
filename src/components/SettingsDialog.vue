<template>
  <el-dialog :model-value="visible" @update:model-value="$emit('update:visible', $event)" title="⚙️ 设置" width="90%"
    :close-on-click-modal="false" class="settings-dialog">
    <div class="settings-body">
      <div class="field-label">
        <el-icon>
          <Key />
        </el-icon>
        <span>DeepSeek API Key</span>
      </div>
      <el-input v-model="localKey" type="password" show-password placeholder="sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
        size="large" />
      <p class="tip">
        你的 API Key 仅存储在本地浏览器中，不会上传到任何服务器。
        可在
        <el-link type="primary" href="https://platform.deepseek.com/api_keys" target="_blank">DeepSeek 官网</el-link>
        获取密钥。
      </p>
      <p v-if="testResult !== null" class="test-result" :class="testResult ? 'success' : 'error'">
        {{ testResult ? 'API 密钥验证通过' : 'API 密钥无效，请检查后重试' }}
      </p>

      <div class="persona-section">
        <div class="field-label">
          <el-icon>
            <Avatar />
          </el-icon>
          <span>角色人设</span>
        </div>
        <el-select v-model="localPersona" class="persona-select" size="large">
          <el-option label="默认助手" value="default" />
          <el-option label="傲娇猫娘" value="tsundere_catgirl" />
        </el-select>
        <p class="tip">人设决定了 AI 的回复风格和语气。</p>
      </div>
    </div>
    <template #footer>
      <el-button @click="handleTest" :loading="testing" :icon="Connection">
        测试连接
      </el-button>
      <el-button type="primary" @click="handleSave" :icon="Check">
        保存
      </el-button>
    </template>
  </el-dialog>
</template>

<script setup>
import { ref, watch } from 'vue'
import { ElMessage } from 'element-plus'
import { Key, Connection, Check, Avatar } from '@element-plus/icons-vue'

const props = defineProps({
  visible: Boolean,
  apiKey: String,
  persona: String
})

const emit = defineEmits(['update:visible', 'update:apiKey', 'update:persona'])

const localKey = ref(props.apiKey || '')
const localPersona = ref(props.persona || 'default')
const testing = ref(false)
const testResult = ref(null)

watch(() => props.apiKey, (val) => {
  localKey.value = val || ''
})

watch(() => props.persona, (val) => {
  localPersona.value = val || 'default'
})

// 弹窗打开时，重置本地值为已保存的值（不保存就关闭不会污染）
watch(() => props.visible, (val) => {
  if (val) {
    localKey.value = props.apiKey || ''
    localPersona.value = props.persona || 'default'
    testResult.value = null
  }
})

// 测试 API 连接
async function handleTest() {
  if (!localKey.value.trim()) {
    ElMessage.warning('请输入 API Key')
    return
  }
  testing.value = true
  testResult.value = null
  try {
    const res = await fetch('https://api.deepseek.com/v1/models', {
      headers: { Authorization: `Bearer ${localKey.value.trim()}` }
    })
    testResult.value = res.ok
    if (!res.ok) ElMessage.error('API Key 无效')
  } catch {
    testResult.value = false
    ElMessage.error('网络错误，无法连接')
  } finally {
    testing.value = false
  }
}

function handleSave() {
  const key = localKey.value.trim()
  if (!key) {
    ElMessage.warning('请输入 API Key')
    return
  }
  localStorage.setItem('deepseek_api_key', key)
  emit('update:apiKey', key)

  // 保存人设
  localStorage.setItem('deepseek_persona', localPersona.value)
  emit('update:persona', localPersona.value)

  emit('update:visible', false)
  ElMessage.success('已保存')
}
</script>

<style scoped>
.settings-body {
  padding: 8px 0;
}

.field-label {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 8px;
  font-weight: 600;
  font-size: 14px;
}

.persona-section {
  margin-top: 18px;
}

.persona-select {
  width: 100%;
}

.tip {
  margin-top: 10px;
  font-size: 13px;
  color: #909399;
  line-height: 1.6;
}

.test-result {
  margin-top: 10px;
  font-size: 14px;
  padding: 6px 0;
}

.test-result.success {
  color: #67c23a;
}

.test-result.error {
  color: #f56c6c;
}
</style>

<!-- ====== 全局样式（弹窗 teleport 到 body，必须非 scoped） ====== -->
<style>
/* 父层 flex 居中 */
.el-overlay-dialog {
  display: flex !important;
  align-items: center !important;
  justify-content: center !important;
}

/* 弹窗本身去 margin */
.el-dialog.settings-dialog {
  margin: 0 !important;
  max-width: 420px;
  width: 90%;
}

@media (max-width: 480px) {
  .el-dialog.settings-dialog {
    width: 92%;
  }

  .el-dialog.settings-dialog .el-dialog__body {
    padding: 16px;
  }
}

@media (max-width: 360px) {
  .el-dialog.settings-dialog .field-label {
    font-size: 13px;
  }

  .el-dialog.settings-dialog .tip {
    font-size: 12px;
  }
}
</style>
