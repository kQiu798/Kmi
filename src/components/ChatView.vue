<template>
  <div class="chat-container">
    <!-- ====== 顶部导航栏 ====== -->
    <header class="chat-header">
      <div class="header-inner">
        <div class="logo-area">
          <span class="logo-text"><img :src="logoSrc" alt="Kmi" class="header-logo" /></span>
          <!-- 人设切换：点击展开下拉，选择结果持久化到本地缓存 -->
          <el-dropdown trigger="click" popper-class="persona-dropdown" @command="handlePersonaCommand">
            <button type="button" class="persona-btn" :title="`当前人设：${personaLabel}`">
              <span class="persona-btn-name">{{ personaLabel }}</span>
              <el-icon class="persona-caret">
                <ArrowDown />
              </el-icon>
            </button>
            <template #dropdown>
              <el-dropdown-menu>
                <el-dropdown-item v-for="opt in PERSONA_OPTIONS" :key="opt.value" :command="opt.value"
                  :class="{ 'persona-item-active': persona === opt.value }">
                  <span class="persona-item">
                    <span class="persona-item-text">
                      <span class="persona-item-name">{{ opt.label }}</span>
                      <span class="persona-item-desc">{{ opt.desc }}</span>
                    </span>
                    <el-icon v-if="persona === opt.value" class="persona-item-check">
                      <Check />
                    </el-icon>
                  </span>
                </el-dropdown-item>
              </el-dropdown-menu>
            </template>
          </el-dropdown>
        </div>
        <div class="header-actions">
          <!-- <el-tooltip content="查询余额" placement="bottom">
            <el-button :icon="Coin" text @click="fetchBalance" :loading="balanceLoading" />
          </el-tooltip> -->
          <el-tooltip content="清空对话" placement="bottom">
            <el-button :icon="Delete" text @click="clearConversation"
              :disabled="messages.length === 0 && !streamingContent && !streamingReasoning" />
          </el-tooltip>
          <el-tooltip content="设置" placement="bottom">
            <el-button :icon="Setting" text @click="showSettings = true" />
          </el-tooltip>
        </div>
      </div>
    </header>

    <!-- ====== 消息列表区域 ====== -->
    <div class="messages-area" ref="messagesRef">
      <div class="messages-inner">
        <!-- 欢迎页（空状态） -->
        <div v-if="messages.length === 0 && !streamingContent && !streamingReasoning && !loading" class="welcome">
          <div class="welcome-icon">
            <img :src="logoSrc" alt="DeepSeek-Kmi" class="welcome-logo" />
          </div>
          <h2 class="welcome-title">你好！有什么我可以帮助你的？</h2>
          <p class="welcome-subtitle">
            当前使用 <strong>{{ modelLabel }}</strong> 模型 · 支持上传文件
          </p>
          <div class="suggestions">
            <div class="suggestion-chip" v-for="(s, i) in suggestions" :key="i"
              @click="inputText = s.short; sendMessage()">
              <span>{{ s.short }}</span>
              <el-icon>
                <TopRight />
              </el-icon>
            </div>
          </div>
        </div>

        <!-- 消息列表 -->
        <div v-for="msg in messages" :key="msg.id" class="message-wrapper" :class="msg.role">
          <div class="message-row">
            <div class="avatar" v-if="msg.role === 'assistant'">
              <span class="avatar-ai">AI</span>
            </div>
            <div class="bubble-wrapper">
              <!-- 用户消息：文字 + 附件 -->
              <template v-if="msg.role === 'user'">
                <div class="bubble user-bubble">
                  <div v-if="msg.content" class="msg-text">{{ msg.content }}</div>
                  <div v-if="msg.files && msg.files.length" class="msg-files">
                    <div v-for="(f, fi) in msg.files" :key="fi" class="msg-file-item">
                      <img v-if="f.isImage && f.dataUrl" :src="f.dataUrl" :alt="f.name" class="uploaded-image"
                        @click="previewImage(f.dataUrl)" />
                      <div v-else class="file-info">
                        <span class="file-info-icon">📄</span>
                        <span class="file-info-name">{{ f.name }}</span>
                      </div>
                    </div>
                  </div>
                </div>
                <!-- 用户消息操作按钮 -->
                <div v-if="msg.content" class="msg-actions user-actions">
                  <button class="msg-action-btn" @click="copyToClipboard(msg.content)" title="复制消息">
                    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <rect x="9" y="9" width="13" height="13" rx="2" ry="2" />
                      <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1" />
                    </svg>
                  </button>
                </div>
              </template>
              <!-- AI 消息：思考过程 + Markdown -->
              <template v-else>
                <!-- 思考过程折叠块，仅 Pro 模式有 reasoning 时显示 -->
                <div v-if="msg.reasoning" class="reasoning-block">
                  <div class="reasoning-toggle" @click="toggleReasoning(msg.id)">
                    <span class="reasoning-label">已深度思考</span>
                    <span class="reasoning-arrow" :class="{ expanded: expandedReasonings[msg.id] }">▼</span>
                  </div>
                  <transition name="reasoning-slide">
                    <div v-show="expandedReasonings[msg.id]" class="reasoning-content">
                      <pre class="reasoning-text">{{ msg.reasoning }}</pre>
                    </div>
                  </transition>
                </div>
                <!-- 主要回复 -->
                <div class="bubble ai-bubble markdown-body" v-html="renderMarkdown(msg.content)" />
                <!-- AI 操作按钮 -->
                <div v-if="msg.content" class="msg-actions ai-actions">
                  <button class="msg-action-btn" @click="copyToClipboard(msg.content)" title="复制消息">
                    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <rect x="9" y="9" width="13" height="13" rx="2" ry="2" />
                      <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1" />
                    </svg>
                  </button>
                  <button class="msg-action-btn" @click="regenerate(msg.id)" title="重新生成">
                    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <path d="M23 4v6h-6" />
                      <path d="M1 20v-6h6" />
                      <path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15" />
                    </svg>
                  </button>
                </div>
                <!-- 版本切换（多个版本时显示左右箭头） -->
                <div v-if="msg.versions && msg.versions.length > 1" class="version-nav">
                  <button class="version-arrow" :disabled="msg.currentVersion <= 0" @click="switchVersion(msg.id, -1)">
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <path d="M15 18l-6-6 6-6" />
                    </svg>
                  </button>
                  <span class="version-indicator">{{ msg.currentVersion + 1 }} / {{ msg.versions.length }}</span>
                  <button class="version-arrow" :disabled="msg.currentVersion >= msg.versions.length - 1"
                    @click="switchVersion(msg.id, 1)">
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <path d="M9 18l6-6-6-6" />
                    </svg>
                  </button>
                </div>
              </template>
            </div>
            <div class="avatar" v-if="msg.role === 'user'">
              <span class="avatar-user">我</span>
            </div>
          </div>
        </div>

        <!-- 正在生成的 AI 回复（流式） -->
        <div v-if="streamingContent || streamingReasoning" class="message-wrapper assistant">
          <div class="message-row">
            <div class="avatar">
              <span class="avatar-ai">AI</span>
            </div>
            <div class="bubble-wrapper">
              <!-- 流式思考内容实时展示 -->
              <div v-if="streamingReasoning" class="reasoning-block">
                <div class="reasoning-toggle active">
                  <span class="reasoning-label">
                    深度思考中<span class="thinking-ellipsis">...</span>
                  </span>
                </div>
                <div class="reasoning-content">
                  <pre class="reasoning-text">{{ streamingReasoning }}</pre>
                </div>
              </div>
              <!-- 流式主内容 -->
              <div v-if="streamingContent" class="bubble ai-bubble markdown-body"
                v-html="renderMarkdown(streamingContent)" />
            </div>
          </div>
        </div>

        <!-- 加载中（等待首次响应） -->
        <div v-if="loading && !streamingContent && !streamingReasoning" class="message-wrapper assistant">
          <div class="message-row">
            <div class="avatar">
              <span class="avatar-ai">AI</span>
            </div>
            <div class="bubble-wrapper">
              <div class="bubble ai-bubble thinking-wrapper">
                <div class="thinking-dots">
                  <span></span><span></span><span></span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- ====== 底部输入区 ====== -->
    <footer class="input-area">
      <div class="input-inner">
        <!-- 待发送的附件预览 -->
        <div v-if="pendingFiles.length > 0" class="pending-files">
          <div v-for="(f, i) in pendingFiles" :key="i" class="pending-file-chip">
            <img v-if="f.isImage" :src="f.dataUrl" class="pending-thumb" />
            <span v-else class="pending-file-icon">📄</span>
            <span class="pending-file-name">{{ f.name }}</span>
            <el-button :icon="Close" text size="small" @click="removePendingFile(i)" class="file-remove-btn" />
          </div>
        </div>

        <div class="input-box">
          <el-input v-model="inputText" type="textarea" :rows="3" :autosize="{ minRows: 1, maxRows: 8 }"
            placeholder="输入消息，Enter 发送，Shift+Enter 换行" :disabled="loading" @keydown="handleKeydown"
            class="chat-input" />
          <div class="input-actions">
            <div class="input-left-actions">
              <!-- 上传按钮 -->
              <el-tooltip content="上传文件" placement="top">
                <el-button :icon="Paperclip" text size="default" @click="triggerFileInput" :disabled="loading"
                  class="upload-btn" />
              </el-tooltip>
              <input ref="fileInputRef" type="file" multiple
                accept=".*"
                @change="handleFileSelect" style="display:none" />
              <!-- Model Toggle: Flash / Pro -->
              <div class="model-toggle">
                <button class="toggle-btn" :class="{ active: modelType === 'flash' }"
                  @click="setModel('flash')">快速</button>
                <button class="toggle-btn" :class="{ active: modelType === 'pro' }" @click="setModel('pro')">深度</button>
              </div>
              <!-- Deep Thinking Toggle + Reasoning Effort（联网搜索开启时隐藏） -->
              <div v-if="!webSearchEnabled" class="dt-controls">
                <button class="dt-toggle" :class="{ active: deepThinkingEnabled }" @click="toggleDeepThinking">
                  深度思考
                </button>
                <div v-if="deepThinkingEnabled" class="re-toggle">
                  <button class="re-btn" :class="{ active: reasoningEffort === 'low' }"
                    @click="setReasoningEffort('low')">Low</button>
                  <button class="re-btn" :class="{ active: reasoningEffort === 'high' }"
                    @click="setReasoningEffort('high')">High</button>
                  <button class="re-btn" :class="{ active: reasoningEffort === 'max' }"
                    @click="setReasoningEffort('max')">Max</button>
                </div>
              </div>
              <!-- 联网搜索 Toggle -->
              <div class="websearch-controls">
                <button class="websearch-toggle" :class="{ active: webSearchEnabled }" @click="toggleWebSearch">
                  联网搜索
                </button>
              </div>
              <!-- Prefix Continuation Toggle：仅配置了前缀的人设显示（联网搜索开启时隐藏） -->
              <div class="prefix-controls" v-if="PERSONA_PREFIX_MAP[persona] && !webSearchEnabled">
                <button class="prefix-toggle" :class="{ active: prefixContinuation }" @click="togglePrefixContinuation">
                  前缀续写
                </button>
              </div>
            </div>
            <div class="input-right-actions">
              <span class="input-tip">Kmi 可能会产生不准确的信息，请注意甄别</span>
              <el-button type="primary" :icon="Promotion" :loading="loading && !streamingContent && !streamingReasoning"
                :disabled="(!inputText.trim() && pendingFiles.length === 0) || loading" circle size="large"
                @click="sendMessage" class="send-btn" />
            </div>
          </div>
        </div>
      </div>
    </footer>

    <!-- ====== 图片预览弹窗 ====== -->
    <el-dialog v-model="previewVisible" title="图片预览" width="auto" class="image-preview-dialog">
      <img :src="previewUrl" style="max-width:80vw;max-height:80vh;border-radius:8px" />
    </el-dialog>

    <!-- ====== 余额弹窗 ====== -->
    <el-dialog v-model="showBalanceDialog" title="账户余额" width="30%" class="balance-dialog">
      <div v-if="balanceInfo" class="balance-body">
        <div class="balance-item" v-for="(val, key) in balanceInfo.balance_infos?.[0] || balanceInfo" :key="key">
          <span class="balance-label">{{ key === 'currency' ? '币种' : key === 'total_balance' ? '总余额' : key ===
            'granted_balance' ? '赠送余额' : key === 'topped_up_balance' ? '充值余额' : key }}</span>
          <span class="balance-value">{{ val }}</span>
        </div>
      </div>
      <div v-else class="balance-loading">加载中...</div>
    </el-dialog>

    <!-- ====== 设置弹窗 ====== -->
    <SettingsDialog v-model:visible="showSettings" v-model:api-key="apiKey" v-model:persona="persona" />
  </div>
</template>

<script setup>
import { ref, nextTick, watch, computed, onMounted } from 'vue'
import { marked } from 'marked'
import hljs from 'highlight.js'
import 'highlight.js/styles/atom-one-dark.css'
import { ElMessage } from 'element-plus'
import {
  Setting, Delete, Promotion, TopRight,
  Paperclip, Close, Coin, ArrowDown, Check
} from '@element-plus/icons-vue'
import SettingsDialog from './SettingsDialog.vue'
import logoSrc from '../assets/logo.png'

// ====== API 地址 ======
const API_BASE = 'https://api.deepseek.com/chat/completions'

// ====== 本地缓存：对话历史 ======
const HISTORY_KEY = 'deepseek_chat_history'
// localStorage 通常上限 5MB，留出余量给其余配置键，超出即逐级降级
const HISTORY_BUDGET = 3.5 * 1024 * 1024
let historyWarned = false

// 反序列化：逐条校验，缓存损坏或结构不符时安全丢弃，避免渲染期崩溃
function loadHistory() {
  let raw = null
  try {
    raw = localStorage.getItem(HISTORY_KEY)
  } catch {
    return [] // 隐私模式下 localStorage 可能直接抛错
  }
  if (!raw) return []

  try {
    const parsed = JSON.parse(raw)
    if (!Array.isArray(parsed)) return []
    return parsed.filter(isValidMessage).map(m => ({
      ...m,
      versions: Array.isArray(m.versions) ? m.versions : undefined,
      currentVersion: Number.isInteger(m.currentVersion) ? m.currentVersion : 0
    }))
  } catch {
    return []
  }
}

function isValidMessage(m) {
  return !!m
    && typeof m === 'object'
    && typeof m.content === 'string'
    && (m.role === 'user' || m.role === 'assistant')
}

// 序列化：withFilePayload 为 false 时剥离体积不可控的图片/文本文件内容，只留元信息
function serializeHistory(msgs, withFilePayload) {
  return JSON.stringify(msgs.map(m => {
    if (!Array.isArray(m.files) || m.files.length === 0) return m
    return {
      ...m,
      files: m.files.map(f => {
        const meta = { name: f.name, type: f.type, size: f.size, isImage: !!f.isImage }
        return withFilePayload ? { ...meta, dataUrl: f.dataUrl, textContent: f.textContent } : meta
      })
    }
  }))
}

function writeHistory(withFilePayload) {
  let payload
  try {
    payload = serializeHistory(messages.value, withFilePayload)
  } catch {
    return false // 存在循环引用等异常，放弃本档
  }
  if (payload.length > HISTORY_BUDGET) return false
  try {
    localStorage.setItem(HISTORY_KEY, payload)
    return true
  } catch {
    return false // 配额超限或存储被禁用
  }
}

// 写入失败不影响对话本身，仅提示一次，避免每次保存都打扰用户
function warnHistoryOnce(text) {
  if (historyWarned) return
  historyWarned = true
  ElMessage.warning(text)
}

// 先完整保存；超限则剥离文件负载重试；仍失败则放弃本次持久化
function persistHistory() {
  if (writeHistory(true)) return
  if (writeHistory(false)) {
    warnHistoryOnce('对话历史较大，图片与文件内容未写入本地缓存（刷新后仅保留文件名）')
    return
  }
  warnHistoryOnce('本地缓存空间不足，本次对话未能保存')
}

// ====== 状态 ======
const apiKey = ref(localStorage.getItem('deepseek_api_key') || '')
const messages = ref(loadHistory())
const inputText = ref('')
const loading = ref(false)
const streamingContent = ref('')
const streamingReasoning = ref('')
const expandedReasonings = ref({})
const showSettings = ref(!apiKey.value)
const messagesRef = ref(null)
const fileInputRef = ref(null)
const pendingFiles = ref([])

// 对话变化即写入本地缓存。messages 只在离散时机变动（发送、回复完成、切换版本、清空），
// 流式过程走 streamingContent，不会逐 token 触发这里
watch(messages, persistHistory, { deep: true })

// 图片预览
const previewVisible = ref(false)
const previewUrl = ref('')

// ====== 模型管理 ======
const modelType = ref(localStorage.getItem('deepseek_model') || 'flash')
const MODEL_MAP = {
  flash: 'deepseek-v4-flash',
  pro: 'deepseek-v4-pro'
}

const modelLabel = computed(() => modelType.value === 'flash' ? 'Flash 快速' : 'Pro 深度')

// 深度思考独立控制
const deepThinkingEnabled = ref(localStorage.getItem('deepseek_thinking_enabled') === 'true')
const reasoningEffort = ref(localStorage.getItem('deepseek_reasoning_effort') || 'high')

// ====== 前缀续写 & 人设 ======
const prefixContinuation = ref(localStorage.getItem('deepseek_prefix_continuation') === 'true')
const persona = ref(localStorage.getItem('deepseek_persona') || 'default')

// 人设选项：标签、说明与取值集中在此，供顶部切换下拉复用
const PERSONA_OPTIONS = [
  { label: '默认助手', desc: '通用对话助手，回答详细准确', value: 'default' },
  { label: '代码助手', desc: '资深软件工程师，只输出代码', value: 'code_assistant' }
]

const personaLabel = computed(() =>
  PERSONA_OPTIONS.find(o => o.value === persona.value)?.label || PERSONA_OPTIONS[0].label
)

// 顶部下拉切换人设：只负责赋值与持久化，副作用由 watch(persona) 统一处理
function handlePersonaCommand(command) {
  if (command === persona.value) return
  persona.value = command
  localStorage.setItem('deepseek_persona', command)
}

// ====== 联网搜索 ======
const webSearchEnabled = ref(localStorage.getItem('deepseek_web_search') === 'true')
// Tavily API Key - 上线前请写死在代码里
const TAVILY_API_KEY = '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' // ← TODO: 请在这里填写 Tavily API Key

const SEARCH_TOOLS = [
  {
    type: 'function',
    function: {
      name: 'web_search',
      description: '搜索互联网获取最新信息，当用户问及时事、新闻、实时数据时使用',
      parameters: {
        type: 'object',
        properties: {
          query: {
            type: 'string',
            description: '搜索关键词'
          }
        },
        required: ['query']
      }
    }
  }
]

function getTavilyKey() {
  return TAVILY_API_KEY || localStorage.getItem('tavily_api_key') || ''
}

async function callTavily(query) {
  const key = getTavilyKey()
  if (!key) throw new Error('Tavily API Key 未配置，请在代码中设置 TAVILY_API_KEY')
  const res = await fetch('https://api.tavily.com/search', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      api_key: key,
      query,
      max_results: 5,
      search_depth: 'basic'
    })
  })
  if (!res.ok) throw new Error(`Tavily 搜索失败 (${res.status})`)
  const data = await res.json()
  return data.results.map((r, i) =>
    `[${i + 1}] ${r.title}\n${r.url}\n${r.content}`
  ).join('\n\n')
}

// ====== 人设 Prompt ======
const PERSONA_PROMPTS = {
  default: '你是Kmi调用DeepSeek V4的大模型, 请用中文回答用户的问题，回答要详细、准确、有帮助。回答中可以使用 Markdown 格式来增强可读性。对于用户上传的文件和图片，请仔细分析并给出专业的见解。若用户输入无明确问题，可友好回应并引导提问。',
  code_assistant: `你是一名资深软件工程师。请严格遵循以下要求生成或修改代码：

1. **逻辑严谨**：明确需求、执行流程、数据关系及边界条件，确保逻辑正确。
2. **代码简洁**：遵循 KISS、DRY 原则，避免冗余代码、重复逻辑和过度设计。
3. **遵循规范**：沿用项目现有技术栈、代码风格、命名规范及架构设计。
4. **最小改动**：仅修改必要内容，不擅自重构无关代码，不破坏现有功能。
5. **完整可用**：输出完整、可直接运行的代码，不省略关键逻辑，不使用伪代码或无意义占位符。
6. **异常处理**：考虑空值、边界条件、异常场景及必要的安全校验。
7. **结果优先**：先分析需求，再实施修改；存在不确定性时，优先检查现有代码及上下文，禁止凭空假设项目结构或接口。
8. **输出要求**：只输出代码，不解释实现过程，不添加客套话、总结或无关内容。确实需要说明时，仅说明影响实现的关键问题。

以正确性、可维护性和最小改动为优先级。生成前检查代码完整性、语法、依赖及潜在错误。`
}

// ====== 人设：前缀续写对应的强制开头 ======
// key 与 PERSONA_PROMPTS 的 key 对应，value 是前缀续写时强制注入的 AI 回复起始文字
// 留空即不注入前缀；当前所有人设均为空，前缀续写功能处于停用状态
const PERSONA_PREFIX_MAP = {
  default: '',
  code_assistant: ''
}

// 会话启动时确定当天日期，用于注入 system prompt
const todayStr = new Date().toLocaleString('zh-CN', { year: 'numeric', month: 'long', day: 'numeric', weekday: 'long', hour: '2-digit', minute: '2-digit' })

const systemPrompt = computed(() => {
  const base = PERSONA_PROMPTS[persona.value] || PERSONA_PROMPTS.default
  return `${base}\n\n当前日期：${todayStr}。请基于此日期回答与时间相关的问题。`
})

// ====== 人设切换时的上下文提示消息 ======
const PERSONA_SWITCH_MESSAGES = {
  default: '（系统通知：人设已恢复为默认助手）',
  code_assistant: '（系统通知：人设已切换为代码助手）请以资深软件工程师的身份协助我。'
}

// 当人设切换时，如果新人设不支持前缀续写则自动关闭
watch(persona, (val, oldVal) => {
  if (!PERSONA_PREFIX_MAP[val]) {
    prefixContinuation.value = false
    localStorage.setItem('deepseek_prefix_continuation', 'false')
  }
  // 人设切换时插入上下文消息，让 AI 知道角色变了
  if (oldVal && val !== oldVal && messages.value.length > 0) {
    const switchMsg = PERSONA_SWITCH_MESSAGES[val]
    if (switchMsg) {
      messages.value.push({
        id: Date.now() + '_' + Math.random().toString(36).slice(2, 8),
        role: 'user',
        content: switchMsg
      })
    }
  }
})

function setModel(type) {
  modelType.value = type
  localStorage.setItem('deepseek_model', type)
  ElMessage.info(`已切换到 ${type === 'flash' ? 'Flash 快速' : 'Pro 深度'} 模型`)
}

function toggleDeepThinking() {
  deepThinkingEnabled.value = !deepThinkingEnabled.value
  localStorage.setItem('deepseek_thinking_enabled', deepThinkingEnabled.value ? 'true' : 'false')

  // 深度思考与前缀续写互斥
  if (deepThinkingEnabled.value && prefixContinuation.value) {
    prefixContinuation.value = false
    localStorage.setItem('deepseek_prefix_continuation', 'false')
    ElMessage.info('深度思考已开启')
  } else {
    ElMessage.info(deepThinkingEnabled.value ? '深度思考已开启' : '深度思考已关闭')
  }
}

function setReasoningEffort(effort) {
  reasoningEffort.value = effort
  localStorage.setItem('deepseek_reasoning_effort', effort)
}

// ====== 前缀续写：仅配置了前缀的人设可用 ======
function togglePrefixContinuation() {
  prefixContinuation.value = !prefixContinuation.value
  localStorage.setItem('deepseek_prefix_continuation', prefixContinuation.value ? 'true' : 'false')

  // 前缀续写与深度思考互斥
  if (prefixContinuation.value && deepThinkingEnabled.value) {
    deepThinkingEnabled.value = false
    localStorage.setItem('deepseek_thinking_enabled', 'false')
    ElMessage.info('前缀续写已开启')
  } else {
    if (prefixContinuation.value) {
      const prefix = PERSONA_PREFIX_MAP[persona.value]
      ElMessage.info(`前缀续写已开启`)
    } else {
      ElMessage.info('前缀续写已关闭')
    }
  }
}

// ====== 联网搜索开关（与深度思考、前缀续写互斥） ======
function toggleWebSearch() {
  webSearchEnabled.value = !webSearchEnabled.value
  localStorage.setItem('deepseek_web_search', webSearchEnabled.value ? 'true' : 'false')

  if (webSearchEnabled.value) {
    if (deepThinkingEnabled.value) {
      deepThinkingEnabled.value = false
      localStorage.setItem('deepseek_thinking_enabled', 'false')
    }
    if (prefixContinuation.value) {
      prefixContinuation.value = false
      localStorage.setItem('deepseek_prefix_continuation', 'false')
    }
    ElMessage.info('联网搜索已开启')
  } else {
    ElMessage.info('联网搜索已关闭')
  }
}

// ====== 思考过程展开/折叠 ======
function toggleReasoning(msgId) {
  expandedReasonings.value = {
    ...expandedReasonings.value,
    [msgId]: !expandedReasonings.value[msgId]
  }
}

// ====== 查询余额 ======
const balanceLoading = ref(false)
const showBalanceDialog = ref(false)
const balanceInfo = ref(null)

async function fetchBalance() {
  if (!apiKey.value) {
    showSettings.value = true
    return
  }
  balanceLoading.value = true
  balanceInfo.value = null
  try {
    const res = await fetch('https://api.deepseek.com/user/balance', {
      headers: { Authorization: `Bearer ${apiKey.value}` }
    })
    if (!res.ok) throw new Error(`请求失败 (${res.status})`)
    const data = await res.json()
    balanceInfo.value = data
    showBalanceDialog.value = true
  } catch (err) {
    ElMessage.error(err.message || '余额查询失败')
  } finally {
    balanceLoading.value = false
  }
}

// ====== 快捷问题 ======
const suggestions = ref([
  { short: '英伟达最新 GPU 架构突破', full: '英伟达最新 GPU 架构有什么突破' },
  { short: '2026 值得关注的 AI 开源项目', full: '2026 年最值得关注的 AI 开源项目' },
  { short: '量子计算最新进展', full: '前沿科技：量子计算的最新进展' },
  { short: 'AI 编程助手深度对比', full: 'AI 编程助手深度对比评测' },
  { short: '全球半导体供应链格局', full: '全球半导体供应链最新格局' }
])
const suggestionsLoading = ref(false)

async function fetchHotSuggestions() {
  const tavilyKey = getTavilyKey()
  const deepseekKey = apiKey.value
  if (!tavilyKey || !deepseekKey) return

  const cached = localStorage.getItem('hot_suggestions_cache')
  if (cached) {
    try {
      const { data, time } = JSON.parse(cached)
      // 一天调用一次（24小时缓存）
      if (Date.now() - time < 86400000 && Array.isArray(data) && data.length === 5) {
        suggestions.value = data
        return
      }
    } catch { /* 缓存失效，重新获取 */ }
  }

  suggestionsLoading.value = true
  try {
    // Step 1: Tavily 搜索热点话题
    const now = new Date()
    const searchRes = await fetch('https://api.tavily.com/search', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        api_key: tavilyKey,
        query: `${now.getFullYear()}年${now.getMonth() + 1}月${now.getDate()}日${now.getHours()}时${now.getMinutes()}分 人工智能 AI 科技 编程 芯片 热门话题`,
        max_results: 8,
        search_depth: 'basic'
      })
    })
    if (!searchRes.ok) return

    const searchData = await searchRes.json()
    const rawTitles = (searchData.results || [])
      .map(r => r.title?.replace(/^[\d\s.、\-—:：]+/, '').trim())
      .filter(Boolean)
      .slice(0, 5)
    if (rawTitles.length < 5) return

    // Step 2: Flash 模型将每个标题精简到 ≤25 字
    const summarizeRes = await fetch(API_BASE, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${deepseekKey}`
      },
      body: JSON.stringify({
        model: 'deepseek-v4-flash',
        messages: [
          {
            role: 'system',
            content: '你是一个标题精简助手。我会给你5个热点话题标题，请将每个标题精简到25个字以内，保留核心意思。直接返回精简后的5个标题，每行一个，不要序号，不要额外说明。'
          },
          {
            role: 'user',
            content: `请精简以下5个热点话题标题：\n${rawTitles.map((t, i) => `${i + 1}. ${t}`).join('\n')}`
          }
        ],
        stream: false,
        max_tokens: 512,
        thinking: { type: 'disabled' },
        temperature: 0.3
      })
    })
    if (!summarizeRes.ok) return

    const json = await summarizeRes.json()
    const text = json.choices?.[0]?.message?.content || ''
    const shortTitles = text.split('\n')
      .map(l => l.trim().replace(/^\d+[.、]\s*/, ''))
      .filter(Boolean)

    if (shortTitles.length >= 5) {
      const paired = shortTitles.slice(0, 5).map((short, i) => ({
        short,
        full: rawTitles[i] || short
      }))
      suggestions.value = paired
      localStorage.setItem('hot_suggestions_cache', JSON.stringify({ data: paired, time: Date.now() }))
    }
  } catch {
    // 获取失败则保留默认推荐
  } finally {
    suggestionsLoading.value = false
  }
}

watch(apiKey, (val) => {
  if (val) fetchHotSuggestions()
})

onMounted(() => {
  if (apiKey.value) fetchHotSuggestions()
})

// ====== Markdown 渲染 ======
marked.use({
  breaks: true,
  gfm: true
})

function renderMarkdown(content) {
  if (!content) return ''
  try {
    return marked.parse(content)
  } catch {
    return content
  }
}

// 代码高亮 & 滚动 + 复制按钮
watch([messages, streamingContent, streamingReasoning, loading], () => {
  nextTick(() => {
    document.querySelectorAll('.markdown-body pre code').forEach((block) => {
      try {
        if (!block.classList.contains('hljs')) {
          hljs.highlightElement(block)
        }
      } catch {
        // 流式传输中代码块可能不完整
      }
    })
    injectCopyButtons()
    scrollToBottom()
  })
}, { deep: true, immediate: true })

function injectCopyButtons() {
  document.querySelectorAll('.markdown-body pre').forEach((pre) => {
    if (pre.querySelector('.copy-code-btn')) return
    const code = pre.querySelector('code')
    if (!code || !code.textContent) return

    pre.style.position = 'relative'

    const btn = document.createElement('button')
    btn.className = 'copy-code-btn'
    btn.innerHTML = '<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="9" y="9" width="13" height="13" rx="2" ry="2"/><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/></svg>'
    btn.title = '复制代码'

    btn.addEventListener('click', async (e) => {
      e.stopPropagation()
      try {
        await navigator.clipboard.writeText(code.textContent)
        btn.innerHTML = '<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#67c23a" stroke-width="2"><polyline points="20 6 9 17 4 12"/></svg>'
        btn.title = '已复制'
        setTimeout(() => {
          btn.innerHTML = '<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="9" y="9" width="13" height="13" rx="2" ry="2"/><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/></svg>'
          btn.title = '复制代码'
        }, 2000)
      } catch {
        btn.title = '复制失败'
        setTimeout(() => { btn.title = '复制代码' }, 2000)
      }
    })

    pre.appendChild(btn)
  })
}

function scrollToBottom() {
  nextTick(() => {
    if (messagesRef.value) {
      messagesRef.value.scrollTop = messagesRef.value.scrollHeight
    }
  })
}

// ====== 消息操作：复制 & 重新生成 ======
function copyToClipboard(text) {
  if (!text) return
  navigator.clipboard.writeText(text).then(() => {
    ElMessage.success('已复制')
  }).catch(() => {
    ElMessage.error('复制失败')
  })
}

async function regenerate(msgId) {
  if (loading.value) return
  const msgIndex = messages.value.findIndex(m => m.id === msgId)
  if (msgIndex === -1) return

  // 找到最后一条用户消息
  let lastUserIdx = -1
  for (let i = msgIndex - 1; i >= 0; i--) {
    if (messages.value[i].role === 'user') {
      lastUserIdx = i
      break
    }
  }
  if (lastUserIdx === -1) return

  const msg = messages.value[msgIndex]
  const lastUserMsg = messages.value[lastUserIdx]

  // 初始化 versions（将当前内容作为第 1 版）
  if (!msg.versions) {
    msg.versions = [{ content: msg.content, reasoning: msg.reasoning, truncated: msg.truncated }]
    msg.currentVersion = 0
  }

  // 删掉这条 AI 消息之后的所有消息
  messages.value = messages.value.slice(0, msgIndex + 1)

  // 自行构建 API 请求（不绕 sendMessage）
  const msgs = [{ role: 'system', content: systemPrompt.value }]
  for (const m of messages.value.slice(0, -1)) {
    msgs.push({ role: m.role, content: m.content || '' })
  }
  msgs.push({ role: 'user', content: lastUserMsg.content || '' })
  // 如果有文件
  if (lastUserMsg.files && lastUserMsg.files.length > 0) {
    const built = buildContentTextOnly(lastUserMsg.content || '', lastUserMsg.files)
    msgs[msgs.length - 1] = { role: 'user', content: built }
  }

  // 如果联网搜索已开启，执行搜索并注入结果
  if (webSearchEnabled.value) {
    const searchQuery = (lastUserMsg.content || '').trim() + ` ${new Date().toISOString().split('T')[0] + ' ' + new Date().toTimeString().slice(0, 5)}`
    streamingContent.value = `🔍 正在搜索：${lastUserMsg.content || ''}...`
    try {
      const searchResult = await callTavily(searchQuery)
      const searchContext = `[以下是为回答用户问题而从互联网搜索到的公开信息]\n\n${searchResult}`
      // 在 system prompt 之后、history 之前插入搜索上下文
      msgs.splice(1, 0, { role: 'user', content: searchContext })
    } catch (e) {
      // 搜索失败不影响继续生成
      console.error('重新生成时搜索失败:', e)
    }
    streamingContent.value = ''
  }

  loading.value = true
  streamingContent.value = ''
  streamingReasoning.value = ''

  try {
    const modelName = MODEL_MAP[modelType.value] || 'deepseek-v4-flash'
    const isProMode = modelType.value === 'pro'

    // 根据当前设置构造请求（模型、深度思考、思考深度等）
    const requestBody = {
      model: modelName,
      messages: msgs,
      stream: true,
      max_tokens: 65536,
      ...(deepThinkingEnabled.value && !webSearchEnabled.value
        ? {
          thinking: { type: 'enabled' },
          reasoning_effort: reasoningEffort.value
        }
        : {
          thinking: { type: 'disabled' },
          temperature: 0.7
        }
      )
    }

    const response = await fetch(API_BASE, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey.value}`
      },
      body: JSON.stringify(requestBody)
    })

    if (!response.ok) {
      let e = `重新生成失败 (${response.status})`
      try { const err = await response.json(); e = err.error?.message || e } catch { /* ignore */ }
      throw new Error(e)
    }

    // 流式读取
    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    let buffer = ''
    let fullContent = ''
    let fullReasoning = ''
    let finishReason = null

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      buffer += decoder.decode(value, { stream: true })
      const lines = buffer.split('\n')
      buffer = lines.pop() || ''
      for (const line of lines) {
        const trimmed = line.trim()
        if (!trimmed || !trimmed.startsWith('data:')) continue
        const data = trimmed.slice(5).trim()
        if (data === '[DONE]') continue
        try {
          const json = JSON.parse(data)
          const delta = json.choices?.[0]?.delta
          if (delta?.reasoning_content) {
            fullReasoning += delta.reasoning_content
            streamingReasoning.value = fullReasoning
          }
          if (delta?.content) {
            fullContent += delta.content
            streamingContent.value = fullContent
          }
          if (json.choices?.[0]?.finish_reason) {
            finishReason = json.choices[0].finish_reason
          }
        } catch { /* skip */ }
      }
    }

    // 添加为新版本
    msg.versions.push({
      content: fullContent,
      reasoning: fullReasoning || undefined,
      truncated: finishReason === 'length'
    })
    msg.currentVersion = msg.versions.length - 1
    msg.content = fullContent
    msg.reasoning = fullReasoning || undefined
    msg.truncated = finishReason === 'length'
  } catch (error) {
    console.error('重新生成失败:', error)
    messages.value.push({
      id: Date.now() + '_' + Math.random().toString(36).slice(2, 8),
      role: 'assistant',
      content: `> ⚠️ **${error.message || '重新生成失败'}**`
    })
  } finally {
    loading.value = false
    streamingContent.value = ''
    streamingReasoning.value = ''
  }
}

function switchVersion(msgId, direction) {
  const msg = messages.value.find(m => m.id === msgId)
  if (!msg || !msg.versions || msg.versions.length < 2) return
  const newIdx = msg.currentVersion + direction
  if (newIdx < 0 || newIdx >= msg.versions.length) return
  msg.currentVersion = newIdx
  const v = msg.versions[newIdx]
  msg.content = v.content
  msg.reasoning = v.reasoning
  msg.truncated = v.truncated
}

// ====== 文件上传 ======
const ALLOWED_IMAGE_TYPES = ['image/png', 'image/jpeg', 'image/gif', 'image/webp', 'image/svg+xml']
const TEXT_EXTENSIONS = new Set([
  'txt', 'md', 'json', 'csv', 'py', 'js', 'ts', 'jsx', 'tsx',
  'html', 'css', 'xml', 'yaml', 'yml', 'sh', 'bat', 'sql',
  'log', 'env', 'ini', 'cfg', 'conf', 'gradle', 'kt', 'java',
  'c', 'cpp', 'h', 'hpp', 'rs', 'go', 'rb', 'php', 'swift',
  'toml', 'lock', 'gitignore', 'dockerfile', 'vue', 'svelte',
  'tex', 'r', 'm', 'scala', 'clj', 'ex', 'exs', 'erl', 'hs'
])

function triggerFileInput() {
  fileInputRef.value?.click()
}

function getFileExtension(filename) {
  return filename.split('.').pop()?.toLowerCase() || ''
}

async function handleFileSelect(e) {
  const files = Array.from(e.target.files || [])
  e.target.value = ''

  if (pendingFiles.value.length + files.length > 5) {
    ElMessage.warning('最多同时上传 5 个文件')
    return
  }

  for (const file of files) {
    if (file.size > 15 * 1024 * 1024) {
      ElMessage.warning(`"${file.name}" 超过 15MB 限制，已跳过`)
      continue
    }
    try {
      const processed = await processFile(file)
      if (processed) pendingFiles.value.push(processed)
    } catch (err) {
      ElMessage.error(`"${file.name}" 处理失败: ${err.message}`)
    }
  }
}

async function processFile(file) {
  const ext = getFileExtension(file.name)
  const isImage = ALLOWED_IMAGE_TYPES.includes(file.type) ||
    ['png', 'jpg', 'jpeg', 'gif', 'webp', 'svg'].includes(ext)

  if (isImage) {
    const dataUrl = await readFileAsDataURL(file)
    const compressed = await compressImageIfNeeded(dataUrl, 2048, 0.85)
    return { name: file.name, type: file.type, size: file.size, isImage: true, dataUrl: compressed }
  }

  const isText = TEXT_EXTENSIONS.has(ext) ||
    file.type.startsWith('text/') ||
    ['application/json', 'application/xml', 'application/yaml'].includes(file.type)

  if (isText) {
    const textContent = await readFileAsText(file)
    return { name: file.name, type: file.type, size: file.size, isImage: false, textContent }
  }

  ElMessage.warning(`"${file.name}" 格式不支持，已跳过（支持图片和常见文本文件）`)
  return null
}

function readFileAsDataURL(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = () => resolve(reader.result)
    reader.onerror = () => reject(new Error('读取失败'))
    reader.readAsDataURL(file)
  })
}

function readFileAsText(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = () => resolve(reader.result)
    reader.onerror = () => reject(new Error('读取失败'))
    reader.readAsText(file, 'utf-8')
  })
}

function compressImageIfNeeded(dataUrl, maxSize, quality) {
  return new Promise((resolve) => {
    const img = new Image()
    img.onload = () => {
      let { width, height } = img
      if (width <= maxSize && height <= maxSize) { resolve(dataUrl); return }
      const ratio = Math.min(maxSize / width, maxSize / height)
      width = Math.round(width * ratio)
      height = Math.round(height * ratio)
      const canvas = document.createElement('canvas')
      canvas.width = width
      canvas.height = height
      canvas.getContext('2d').drawImage(img, 0, 0, width, height)
      resolve(canvas.toDataURL('image/jpeg', quality))
    }
    img.onerror = () => resolve(dataUrl)
    img.src = dataUrl
  })
}

function removePendingFile(index) {
  pendingFiles.value.splice(index, 1)
}

function previewImage(url) {
  previewUrl.value = url
  previewVisible.value = true
}

// ====== 构造 API 消息内容 ======
function buildContentWithImages(text, files) {
  if (!files || files.length === 0) return text || '（空消息）'
  const parts = []
  let textContent = text || ''
  const imageParts = []
  const textFileContents = []

  for (const f of files) {
    if (f.isImage && f.dataUrl) {
      imageParts.push({ type: 'image_url', image_url: { url: f.dataUrl } })
    } else if (f.isImage) {
      // 从本地缓存恢复的历史消息：图片负载未保留，退化为文字说明，
      // 避免发出 url 为 undefined 的非法 image_url 导致接口报错
      textFileContents.push(`--- 图片: ${f.name}（历史记录，原图未保留）---`)
    } else if (f.textContent) {
      textFileContents.push(`--- 文件: ${f.name} ---\n${f.textContent}\n--- 文件结束 ---`)
    }
  }

  if (imageParts.length > 0 && !text && textFileContents.length === 0) textContent = '请描述这张图片。'
  if (textFileContents.length > 0) {
    const prefix = textContent ? '\n\n' : ''
    textContent += `${prefix}以下是我上传的文件内容：\n\n${textFileContents.join('\n\n')}`
  }
  if (textContent) parts.push({ type: 'text', text: textContent })
  parts.push(...imageParts)
  return parts.length ? parts : [{ type: 'text', text: '（空消息）' }]
}

function buildContentTextOnly(text, files) {
  if (!files || files.length === 0) return text || '（空消息）'
  const imageNames = []
  const textContents = []

  for (const f of files) {
    if (f.isImage) {
      imageNames.push(`${f.name} (${Math.round(f.size / 1024)}KB)`)
    } else if (f.textContent) {
      textContents.push(`--- 文件: ${f.name} ---\n${f.textContent}\n--- 文件结束 ---`)
    }
  }

  let finalText = text || ''
  if (imageNames.length > 0) {
    const prefix = finalText ? '\n\n' : ''
    finalText += `${prefix}[用户上传了 ${imageNames.length} 张图片]\n${imageNames.map(n => `- ${n}`).join('\n')}`
  }
  if (textContents.length > 0) {
    const prefix = finalText ? '\n\n' : ''
    finalText += `${prefix}以下是我上传的文件内容：\n\n${textContents.join('\n\n')}`
  }
  if (imageNames.length > 0 && !text && textContents.length === 0) {
    finalText = `我上传了以下图片，请查看：\n${imageNames.map(n => `- ${n}`).join('\n')}`
  }
  return [{ type: 'text', text: finalText || '（空消息）' }]
}

// ====== 发送消息 ======
async function sendMessage() {
  const text = inputText.value.trim()
  const hasText = !!text
  const hasFiles = pendingFiles.value.length > 0

  if (!hasText && !hasFiles) return
  if (loading.value) return

  if (!apiKey.value) {
    showSettings.value = true
    return
  }

  const userMsg = {
    id: Date.now() + '_' + Math.random().toString(36).slice(2, 8),
    role: 'user',
    content: text || '',
    files: hasFiles ? [...pendingFiles.value] : undefined
  }
  messages.value.push(userMsg)
  inputText.value = ''
  const filesToSend = [...pendingFiles.value]
  pendingFiles.value = []
  loading.value = true
  streamingContent.value = ''
  streamingReasoning.value = ''
  scrollToBottom()

  const modelName = MODEL_MAP[modelType.value] || 'deepseek-v4-flash'
  const isProMode = modelType.value === 'pro'

  let useVision = filesToSend.some(f => f.isImage) && !isProMode
  let retried = false

  // ====== [修复] 构建 API 消息 ======
  function buildApiMessages(useVisionFlag) {
    const msgs = [{
      role: 'system',
      content: systemPrompt.value
    }]

    // 历史消息（不含当前这条）
    for (const m of messages.value.slice(0, -1)) {
      if (m.role === 'user' && m.files && m.files.length > 0) {
        const built = useVisionFlag
          ? buildContentWithImages(m.content, m.files)
          : buildContentTextOnly(m.content, m.files)
        msgs.push({ role: 'user', content: built })
      } else {
        const msg = { role: m.role, content: m.content || '' }
        // 仅当深度思考开启时才回传 reasoning_content（联网搜索时深度思考已关闭）
        if (m.role === 'assistant' && m.reasoning && deepThinkingEnabled.value) {
          msg.reasoning_content = m.reasoning
        }
        msgs.push(msg)
      }
    }

    // ====== [修复核心] 当前用户消息始终正常发送 ======
    // 前缀续写不是替换用户消息，而是在用户消息之后追加一条带 prefix:true 的 assistant 消息
    // 用来强制 AI 的回复从指定文字开始续写
    const current = useVisionFlag
      ? buildContentWithImages(text, filesToSend)
      : buildContentTextOnly(text, filesToSend)
    msgs.push({ role: 'user', content: current })

    // 前缀续写：在用户消息之后，追加 assistant 前缀（与深度思考互斥，双重保险）
    if (prefixContinuation.value && !deepThinkingEnabled.value) {
      const prefixText = PERSONA_PREFIX_MAP[persona.value]
      if (prefixText) {
        // prefix: true 告诉 API：这条 assistant 消息是回复的开头，请从这里续写
        msgs.push({ role: 'assistant', content: prefixText, prefix: true })
      }
    }

    return msgs
  }

  async function doRequest() {
    const apiMessages = buildApiMessages(useVision)

    // 前缀续写（prefix: true）需要走 Beta API
    // 条件与 buildApiMessages 中实际注入前缀的条件保持一致：没有前缀可注入时不切端点
    const apiBase = prefixContinuation.value && !deepThinkingEnabled.value && PERSONA_PREFIX_MAP[persona.value]
      ? 'https://api.deepseek.com/beta/chat/completions'
      : API_BASE

    const requestBody = {
      model: modelName,
      messages: apiMessages,
      stream: true,
      max_tokens: 65536,
      ...(deepThinkingEnabled.value
        ? {
          thinking: { type: 'enabled' },
          reasoning_effort: reasoningEffort.value
        }
        : {
          thinking: { type: 'disabled' },
          temperature: 0.7
        }
      )
    }

    const response = await fetch(apiBase, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey.value}`
      },
      body: JSON.stringify(requestBody)
    })

    if (!response.ok) {
      let errorMsg = `请求失败 (${response.status})`
      try {
        const err = await response.json()
        errorMsg = err.error?.message || errorMsg
      } catch { /* ignore */ }

      if (useVision && errorMsg.includes('unknown variant') && !retried) {
        useVision = false
        retried = true
        ElMessage.info('当前模型暂不支持图片识别，已自动切换为文字描述模式')
        return doRequest()
      }
      throw new Error(errorMsg)
    }

    return response
  }

  // ====== 联网搜索：始终搜索，不依赖 AI 决策 ======
  async function sendWithWebSearch() {
    // 搜索词加上当天日期，提升时效性
    const searchDate = new Date().toISOString().split('T')[0] + ' ' + new Date().toTimeString().slice(0, 5) // 2026-06-09 14:30
    const searchQuery = (text.trim() || (filesToSend.length > 0 ? '分析上传的文件' : '用户查询')) + ` 当前时间: ${searchDate}`

    streamingContent.value = `🔍 正在搜索：${searchQuery}...`

    const searchResult = await callTavily(searchQuery)

    streamingContent.value = ''

    // 构建 API 消息
    const apiMessages = buildApiMessages(false)
    // 联网搜索时深度思考已关闭，确保所有消息不携带 reasoning_content
    for (const msg of apiMessages) {
      if (msg.reasoning_content) delete msg.reasoning_content
    }

    // 搜索结果作为独立的 user 上下文消息注入（而非塞进 system prompt）
    // 避免触发 DeepSeek 内容过滤，同时让 AI 清楚这是外部搜索来的信息
    const searchContext = `[以下是为回答用户问题而从互联网搜索到的公开信息]\n\n${searchResult}`
    // apiMessages = [system, history..., current_user_msg]
    // 在 history 和 current_user_msg 之间插入搜索上下文
    const messages = [
      apiMessages[0], // system prompt（已含日期）
      { role: 'user', content: searchContext },
      ...apiMessages.slice(1) // history + current user message
    ]

    // 流式请求
    const response = await fetch(API_BASE, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey.value}`
      },
      body: JSON.stringify({
        model: modelName,
        messages,
        stream: true,
        max_tokens: 65536,
        thinking: { type: 'disabled' },
        temperature: 0.7
      })
    })
    if (!response.ok) {
      let e = `联网搜索请求失败 (${response.status})`
      try {
        const err = await response.json()
        const msg = err.error?.message || e
        // DeepSeek 内容过滤误报，给用户友好提示
        if (msg.includes('Content Exists Risk')) {
          throw new Error('搜索到的内容被内容安全系统过滤，请尝试更换搜索词或关闭联网搜索后重试。')
        }
        e = msg
        console.error('WebSearch error detail:', err)
      } catch (errMsg) {
        if (errMsg instanceof Error) throw errMsg
        e = errMsg
      }
      throw new Error(e)
    }

    return await readStream(response)
  }

  // ====== 流式读取公共逻辑 ======
  async function readStream(response, initialContent = '') {
    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    let buffer = ''
    let fullContent = initialContent
    let fullReasoning = ''
    let finishReason = null

    if (initialContent) streamingContent.value = initialContent

    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      buffer += decoder.decode(value, { stream: true })
      const lines = buffer.split('\n')
      buffer = lines.pop() || ''

      for (const line of lines) {
        const trimmed = line.trim()
        if (!trimmed || !trimmed.startsWith('data:')) continue
        const data = trimmed.slice(5).trim()
        if (data === '[DONE]') continue

        try {
          const json = JSON.parse(data)
          const delta = json.choices?.[0]?.delta

          if (delta?.reasoning_content) {
            fullReasoning += delta.reasoning_content
            streamingReasoning.value = fullReasoning
          }
          if (delta?.content) {
            fullContent += delta.content
            streamingContent.value = fullContent
          }
          if (json.choices?.[0]?.finish_reason) {
            finishReason = json.choices[0].finish_reason
            if (finishReason === 'stop') break
          }
        } catch {
          // 跳过解析失败的行
        }
      }
    }

    return { fullContent, fullReasoning, finishReason }
  }

  try {
    let result
    if (webSearchEnabled.value) {
      result = await sendWithWebSearch()
    } else {
      const response = await doRequest()
      const initialContent = prefixContinuation.value && PERSONA_PREFIX_MAP[persona.value]
        ? PERSONA_PREFIX_MAP[persona.value]
        : ''
      result = await readStream(response, initialContent)
    }

    let { fullContent, fullReasoning, finishReason } = result

    // 检测是否因 Token 超限被截断
    let truncationWarning = ''
    if (finishReason === 'length') {
      const reasoningLen = fullReasoning ? Math.round(fullReasoning.length / 2) : 0
      const contentLen = fullContent ? Math.round(fullContent.length / 2) : 0
      truncationWarning = '\n\n---\n> ⚠️ **回复被截断**：'
      if (reasoningLen > 4000 && contentLen < 500) {
        truncationWarning += `思考过程消耗了大量 Token（估算约 ${reasoningLen} tokens），导致最终回答空间不足。`
      } else {
        truncationWarning += `输出达到 Token 上限（当前 max_tokens = 65536）被截断。`
      }
      truncationWarning += '\n>\n> 建议处理方式：\n> - **关闭深度思考**后重新提问\n> - 或将问题拆分为多个小问题分别提问\n> - 在设置中确保 API Key 有足够余额'
      fullContent += truncationWarning
      if (streamingContent.value) {
        streamingContent.value = fullContent
      }
    }

    if (fullContent || fullReasoning) {
      const msgId = Date.now() + '_' + Math.random().toString(36).slice(2, 8)
      messages.value.push({
        id: msgId,
        role: 'assistant',
        content: fullContent,
        reasoning: fullReasoning || undefined,
        truncated: finishReason === 'length',
        versions: [{ content: fullContent, reasoning: fullReasoning || undefined, truncated: finishReason === 'length' }],
        currentVersion: 0
      })
      if (fullReasoning) {
        expandedReasonings.value = { ...expandedReasonings.value, [msgId]: false }
      }
    }
  } catch (error) {
    console.error('请求失败:', error)

    if (filesToSend.some(f => f.isImage) && isProMode) {
      ElMessage.warning('Pro 模型暂不支持图片识别，请切换到 Flash 模型或移出图片后重试')
    } else {
      // 错误信息直接显示到聊天区，不弹窗
      const errMsg = error.message || '请求失败，请检查网络连接和 API 密钥'
      messages.value.push({
        id: Date.now() + '_' + Math.random().toString(36).slice(2, 8),
        role: 'assistant',
        content: `> ⚠️ **${errMsg}**`
      })
    }
  } finally {
    loading.value = false
    streamingContent.value = ''
    streamingReasoning.value = ''
    scrollToBottom()
  }
}

// ====== 清空对话 ======
function clearConversation() {
  if (messages.value.length === 0 && !streamingContent.value && !streamingReasoning.value) return
  messages.value = []
  streamingContent.value = ''
  streamingReasoning.value = ''
  expandedReasonings.value = {}
  scrollToBottom()
}

// ====== 键盘事件 ======
function handleKeydown(e) {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault()
    sendMessage()
  }
}
</script>

<style scoped>
/* ========== 整体布局 ========== */
.chat-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background: #f5f5f5;
  position: relative;
}

/* ========== 顶部导航 ========== */
.chat-header {
  flex-shrink: 0;
  background: #fff;
  border-bottom: 1px solid #e8e8e8;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.04);
  position: sticky;
  top: 0;
  z-index: 10;
}

.header-inner {
  max-width: 1200px;
  margin: 0 auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 24px;
}

.logo-area {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
}

.logo-text {
  font-size: 22px;
  font-weight: 700;
  color: #1a1a1a;
  letter-spacing: 0.5px;
  display: flex;
  align-items: center;
  gap: 2px;
}

.header-logo {
  height: 40px;
  width: auto;
  vertical-align: middle;
  margin-top: 4px;
}

/* ===== 人设切换（幽灵按钮：无边框无底色，悬停出浅灰底） ===== */
.persona-btn {
  display: inline-flex;
  align-items: center;
  gap: 2px;
  padding: 5px 8px;
  font-family: inherit;
  font-size: 15px;
  font-weight: 500;
  line-height: 1.4;
  color: #1a1a1a;
  background: transparent;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  white-space: nowrap;
  transition: background 0.2s;
}

.persona-btn:hover {
  background: #f2f3f5;
}

.persona-btn:focus-visible {
  outline: 2px solid rgba(22, 119, 255, 0.4);
  outline-offset: -1px;
}

.persona-caret {
  font-size: 13px;
  color: #8a8f99;
  transition: color 0.2s;
}

.persona-btn:hover .persona-caret {
  color: #1a1a1a;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 6px;
}

/* ========== 消息区域 ========== */
.messages-area {
  flex: 1;
  overflow-y: auto;
  padding: 24px 0;
}

.messages-inner {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 15px;
}

/* ===== 欢迎页 ===== */
.welcome {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px 20px 40px;
  text-align: center;
}

.welcome-icon {
  margin-bottom: 16px;
  animation: float 3s ease-in-out infinite;
}

.welcome-logo {
  width: 180px;
  height: auto;
  display: block;
}

@keyframes float {

  0%,
  100% {
    transform: translateY(0);
  }

  50% {
    transform: translateY(-10px);
  }
}

.welcome-title {
  font-size: 26px;
  font-weight: 700;
  color: #1a1a1a;
  margin-bottom: 8px;
}

.welcome-subtitle {
  font-size: 15px;
  color: #909399;
  margin-bottom: 32px;
}

.suggestions {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  justify-content: center;
  max-width: 600px;
}

.suggestion-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 10px 18px;
  background: #fff;
  border: 1px solid #e4e7ed;
  border-radius: 20px;
  font-size: 14px;
  color: #606266;
  cursor: pointer;
  transition: all 0.2s;
}

.suggestion-chip:hover {
  border-color: #409eff;
  color: #409eff;
  background: #ecf5ff;
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(64, 158, 255, 0.15);
}

/* ===== 消息行 ===== */
.message-wrapper {
  margin-bottom: 20px;
}

.message-row {
  display: flex;
  align-items: flex-start;
  gap: 12px;
}

.message-wrapper.user .message-row {
  justify-content: flex-end;
}

/* ===== 头像 ===== */
.avatar {
  flex-shrink: 0;
  margin-top: 0px;
}

.avatar-ai {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 34px;
  height: 34px;
  border-radius: 8px;
  background: linear-gradient(135deg, #4fc3f7, #0288d1);
  color: #fff;
  font-size: 12px;
  font-weight: 700;
}

.avatar-user {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 34px;
  height: 34px;
  border-radius: 8px;
  background: linear-gradient(135deg, #a8e6cf, #3b82f6);
  color: #fff;
  font-size: 12px;
  font-weight: 700;
}

/* ===== 气泡 ===== */
.bubble-wrapper {
  flex: 1;
  min-width: 0;
}

.message-wrapper.user .bubble-wrapper {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
}

.bubble {
  padding: 12px 16px;
  border-radius: 12px;
  line-height: 1.7;
  word-wrap: break-word;
  overflow-wrap: break-word;
}

.user-bubble {
  background: linear-gradient(135deg, #409eff, #337ecc);
  color: #fff;
  font-size: 15px;
  border-bottom-right-radius: 4px;
  max-width: 70%;
  display: inline-block;
  text-align: left;
}

.user-bubble .msg-text {
  white-space: pre-wrap;
}

.ai-bubble {
  background: #fff;
  border: 1px solid #e8e8e8;
  border-bottom-left-radius: 4px;
  color: #333;
}

.thinking-wrapper {
  padding: 8px 16px;
}

/* ===== 消息操作按钮（复制/重新生成） ===== */
.msg-actions {
  display: flex;
  align-items: center;
  gap: 2px;
  padding: 2px 0;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.message-wrapper:hover .msg-actions {
  opacity: 1;
}

.msg-action-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border: none;
  background: transparent;
  color: #c0c4cc;
  cursor: pointer;
  border-radius: 6px;
  transition: all 0.2s;
}

.msg-action-btn:hover {
  background: #f0f2f5;
  color: #409eff;
}

.user-actions {
  justify-content: flex-end;
}

/* ===== 版本切换箭头 ===== */
.version-nav {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  margin-top: 6px;
  padding: 2px 6px;
  border: 1px solid #e8e8e8;
  border-radius: 14px;
  background: #fff;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.message-wrapper:hover .version-nav {
  opacity: 1;
}

.version-arrow {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  border: none;
  background: transparent;
  color: #606266;
  cursor: pointer;
  border-radius: 4px;
  transition: all 0.2s;
  padding: 0;
}

.version-arrow:hover:not(:disabled) {
  background: #f0f2f5;
  color: #409eff;
}

.version-arrow:disabled {
  color: #dcdfe6;
  cursor: not-allowed;
}

.version-indicator {
  font-size: 11px;
  color: #909399;
  user-select: none;
  min-width: 28px;
  text-align: center;
}

/* ===== 消息中的附件 ===== */
.msg-files {
  margin-top: 8px;
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.msg-file-item {
  border-radius: 8px;
  overflow: hidden;
  max-width: 200px;
}

.uploaded-image {
  display: block;
  width: 100%;
  max-height: 200px;
  object-fit: cover;
  border-radius: 8px;
  cursor: pointer;
  border: 2px solid rgba(255, 255, 255, 0.3);
  transition: transform 0.2s;
}

.uploaded-image:hover {
  transform: scale(1.02);
}

.file-info {
  display: flex;
  align-items: center;
  gap: 6px;
  background: rgba(255, 255, 255, 0.15);
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 13px;
  white-space: nowrap;
  max-width: 200px;
}

.file-info-icon {
  font-size: 16px;
}

.file-info-name {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* ========== 思考过程块样式 ========== */
.reasoning-block {
  margin-bottom: 8px;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  overflow: hidden;
  background: #f8fafc;
  font-size: 13px;
}

.reasoning-toggle {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 12px;
  cursor: pointer;
  user-select: none;
  background: #f0f4f8;
  transition: background 0.2s;
}

.reasoning-toggle:hover {
  background: #e8eef5;
}

.reasoning-toggle.active {
  background: #eff6ff;
  cursor: default;
}

.reasoning-label {
  flex: 1;
  color: #475569;
  font-weight: 500;
  font-size: 13px;
}

.reasoning-arrow {
  color: #94a3b8;
  font-size: 10px;
  transition: transform 0.25s ease;
  transform: rotate(-90deg);
}

.reasoning-arrow.expanded {
  transform: rotate(0deg);
}

.reasoning-content {
  padding: 10px 14px;
  max-height: 300px;
  overflow-y: auto;
  border-top: 1px solid #e2e8f0;
}

.reasoning-text {
  margin: 0;
  font-family: 'SF Mono', 'Consolas', 'Monaco', monospace;
  font-size: 12px;
  line-height: 1.75;
  color: #64748b;
  white-space: pre-wrap;
  word-wrap: break-word;
}

.thinking-ellipsis {
  display: inline-block;
  animation: ellipsis-blink 1.4s infinite;
}

@keyframes ellipsis-blink {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.2;
  }
}

.reasoning-slide-enter-active,
.reasoning-slide-leave-active {
  transition: opacity 0.2s ease;
}

.reasoning-slide-enter-from,
.reasoning-slide-leave-to {
  opacity: 0;
}

/* ========== 输入区域 ========== */
.input-area {
  flex-shrink: 0;
  background: #fff;
  border-top: 1px solid #e8e8e8;
  padding: 8px 24px 16px;
}

.input-inner {
  max-width: 1100px;
  margin: 0 auto;
}

/* ===== 待发送附件 ===== */
.pending-files {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 8px;
}

.pending-file-chip {
  display: flex;
  align-items: center;
  gap: 4px;
  background: #f0f2f5;
  border: 1px solid #e4e7ed;
  border-radius: 8px;
  padding: 4px 8px;
  font-size: 13px;
  animation: fadeInUp 0.2s ease-out;
}

.pending-thumb {
  width: 28px;
  height: 28px;
  border-radius: 4px;
  object-fit: cover;
}

.pending-file-icon {
  font-size: 20px;
}

.pending-file-name {
  max-width: 120px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: #606266;
}

.file-remove-btn {
  margin-left: 2px;
}

/* ===== 输入框 ===== */
.input-box {
  position: relative;
}

.input-box :deep(.chat-input .el-textarea__inner) {
  border-radius: 12px;
  padding: 12px 16px;
  font-size: 15px;
  line-height: 1.6;
  resize: none;
  border: 1px solid #dcdfe6;
  transition: border-color 0.2s, box-shadow 0.2s;
}

.input-box :deep(.chat-input .el-textarea__inner:focus) {
  border-color: #409eff;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.15);
}

.input-actions {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 6px;
  padding: 0 2px;
}

.input-left-actions {
  display: flex;
  align-items: center;
  gap: 4px;
}

.input-right-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.upload-btn {
  color: #909399;
}

.upload-btn:hover {
  color: #409eff;
}

/* ===== 模型切换 ===== */
.model-toggle {
  display: inline-flex;
  align-items: center;
  border: 1px solid #e0e0e0;
  border-radius: 20px;
  background: #f5f5f5;
  overflow: hidden;
}

.toggle-btn {
  padding: 4px 14px;
  font-size: 12px;
  font-weight: 500;
  border: none;
  background: transparent;
  color: #909399;
  cursor: pointer;
  transition: all 0.25s;
  line-height: 1.4;
  white-space: nowrap;
}

.toggle-btn.active {
  background: #fff;
  color: #1677ff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border-radius: 20px;
}

/* ===== 深度思考控制组 ===== */
.dt-controls {
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.dt-toggle {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 12px;
  border-radius: 20px;
  font-size: 12px;
  font-weight: 500;
  border: 1px solid #e0e0e0;
  background: transparent;
  color: #909399;
  cursor: pointer;
  transition: all 0.25s;
  white-space: nowrap;
  line-height: 1.4;
}

.dt-toggle.active {
  background: rgba(22, 119, 255, 0.05);
  border-color: #1677ff;
  color: #1677ff;
}

/* ===== 联网搜索按钮 ===== */
.websearch-controls {
  display: inline-flex;
  align-items: center;
}

.websearch-toggle {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 12px;
  border-radius: 20px;
  font-size: 12px;
  font-weight: 500;
  border: 1px solid #e0e0e0;
  background: transparent;
  color: #909399;
  cursor: pointer;
  transition: all 0.25s;
  white-space: nowrap;
  line-height: 1.4;
}

.websearch-toggle.active {
  background: rgba(0, 180, 100, 0.05);
  border-color: #00b464;
  color: #00b464;
}

/* ===== 前缀续写按钮 ===== */
.prefix-controls {
  display: inline-flex;
  align-items: center;
}

.prefix-toggle {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 12px;
  border-radius: 20px;
  font-size: 12px;
  font-weight: 500;
  border: 1px solid #e0e0e0;
  background: transparent;
  color: #909399;
  cursor: pointer;
  transition: all 0.25s;
  white-space: nowrap;
  line-height: 1.4;
}

.prefix-toggle.active {
  background: rgba(245, 158, 11, 0.05);
  border-color: #f59e0b;
  color: #f59e0b;
}

/* ===== 推理努力度 ===== */
.re-toggle {
  display: inline-flex;
  align-items: center;
  border: 1px solid #d0d0d0;
  border-radius: 14px;
  background: #f5f5f5;
  overflow: hidden;
}

.re-btn {
  padding: 2px 10px;
  font-size: 11px;
  font-weight: 500;
  border: none;
  background: transparent;
  color: #909399;
  cursor: pointer;
  transition: all 0.25s;
  line-height: 1.4;
  white-space: nowrap;
}

.re-btn.active {
  background: #fff;
  color: #1677ff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border-radius: 14px;
}

.input-tip {
  font-size: 12px;
  color: #c0c4cc;
}

.send-btn {
  flex-shrink: 0;
}

/* ===== 图片预览弹窗 ===== */
.image-preview-dialog :deep(.el-dialog__body) {
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* ===== 余额弹窗 ===== */
.balance-dialog {
  max-width: 300px;
}

.balance-dialog :deep(.el-dialog) {
  margin-top: 0 !important;
}

.balance-body {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.balance-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 14px;
  background: #f5f7fa;
  border-radius: 8px;
}

.balance-label {
  font-size: 13px;
  color: #909399;
}

.balance-value {
  font-size: 16px;
  font-weight: 600;
  color: #303133;
}

.balance-loading {
  text-align: center;
  color: #909399;
  padding: 20px;
}
</style>

<!-- ====== 全局样式 ====== -->
<style>
.balance-dialog .el-dialog {
  margin: 0 !important;
}

.balance-dialog .el-dialog__header {
  padding-bottom: 4px !important;
}

/* ===== 人设切换下拉面板（菜单 teleport 到 body，不参与 scoped） ===== */
.persona-dropdown.el-popper {
  padding: 0;
  border: 1px solid #eceef2;
  border-radius: 12px;
  box-shadow: 0 6px 24px rgba(0, 0, 0, 0.08);
}

.persona-dropdown .el-dropdown-menu {
  padding: 6px;
  min-width: 212px;
  background: transparent;
}

.persona-dropdown .el-dropdown-menu__item {
  padding: 0;
  border-radius: 8px;
  line-height: 1.4;
}

.persona-dropdown .el-dropdown-menu__item:not(.is-disabled):hover,
.persona-dropdown .el-dropdown-menu__item:not(.is-disabled):focus {
  background: #f5f6f8;
  color: inherit;
}

.persona-item {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
  padding: 8px 10px;
}

.persona-item-text {
  display: flex;
  flex: 1;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.persona-item-name {
  font-size: 14px;
  font-weight: 500;
  line-height: 1.4;
  color: #1a1a1a;
}

.persona-item-desc {
  font-size: 12px;
  line-height: 1.4;
  color: #909399;
  white-space: normal;
}

.persona-item-check {
  flex-shrink: 0;
  font-size: 15px;
  color: #1677ff;
}

.persona-dropdown .el-dropdown-menu__item.persona-item-active .persona-item-name {
  color: #1677ff;
  font-weight: 600;
}

/* ===== 代码块复制按钮（全局，JS 注入不参与 scoped） ===== */
.copy-code-btn {
  position: absolute;
  top: 8px;
  right: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  background: transparent;
  border: none;
  color: #fff;
  cursor: pointer;
  opacity: 0;
  transition: opacity 0.2s;
  z-index: 5;
  padding: 0;
}

.markdown-body pre:hover .copy-code-btn {
  opacity: 0.5;
}

.copy-code-btn:hover {
  opacity: 1 !important;
}

@media (hover: none) {
  .copy-code-btn {
    opacity: 0.35;
  }
}

/* ============================================================
   移动端适配
   ============================================================ */

@media (max-width: 600px) {
  .header-inner {
    padding: 10px 12px;
  }

  .logo-text {
    font-size: 18px;
  }

  .header-logo {
    height: 30px;
  }

  .persona-btn {
    font-size: 14px;
    padding: 4px 7px;
  }

  .messages-inner {
    padding: 0 12px;
  }

  .welcome {
    padding: 40px 12px 30px;
  }

  .welcome-logo {
    width: 140px;
  }

  .welcome-title {
    font-size: 20px;
  }

  .avatar {
    display: none;
  }

  .bubble-wrapper {
    max-width: 100%;
  }

  .input-area {
    padding: 8px 12px 12px;
  }
}

@media (max-width: 480px) {
  .header-inner {
    padding: 8px 10px;
  }

  .logo-text {
    font-size: 16px;
  }

  .header-logo {
    height: 28px;
  }

  .persona-btn {
    font-size: 14px;
    padding: 4px 6px;
  }

  .chat-header {
    min-width: 0;
  }

  .messages-area {
    padding: 16px 0;
  }

  .messages-inner {
    padding: 0 8px;
  }

  .welcome {
    padding: 28px 8px 24px;
  }

  .welcome-icon {
    margin-bottom: 12px;
  }

  .welcome-logo {
    width: 120px;
  }

  .welcome-title {
    font-size: 18px;
  }

  .welcome-subtitle {
    font-size: 13px;
    margin-bottom: 20px;
  }

  .suggestion-chip {
    padding: 8px 14px;
    font-size: 13px;
  }

  .message-wrapper {
    margin-bottom: 14px;
  }

  .avatar {
    display: none;
  }

  .bubble {
    padding: 10px 14px;
    font-size: 14px;
  }

  .bubble-wrapper {
    max-width: 100%;
  }

  .user-bubble {
    max-width: 100%;
  }

  .input-area {
    padding: 6px 10px 10px;
  }

  .input-box :deep(.chat-input .el-textarea__inner) {
    padding: 10px 14px;
    font-size: 14px;
  }

  .input-tip {
    display: none;
  }

  .model-toggle .toggle-btn {
    font-size: 11px;
    padding: 3px 10px;
  }

  .dt-toggle {
    font-size: 11px;
    padding: 3px 10px;
  }

  .prefix-toggle {
    font-size: 11px;
    padding: 3px 10px;
  }

  .websearch-toggle {
    font-size: 11px;
    padding: 3px 10px;
  }

  .re-btn {
    font-size: 10px;
    padding: 2px 8px;
  }

  .input-actions {
    flex-wrap: wrap;
    gap: 4px;
  }

  .input-left-actions {
    gap: 2px;
  }

  .input-right-actions {
    gap: 4px;
  }

  .pending-file-chip {
    font-size: 12px;
    padding: 3px 6px;
  }

  .pending-file-name {
    max-width: 80px;
  }

  .pending-thumb {
    width: 22px;
    height: 22px;
  }

  .reasoning-content {
    max-height: 200px;
  }

  .reasoning-text {
    font-size: 11px;
  }

  /* 移动端操作按钮常显（无 hover） */
  .msg-actions {
    opacity: 1;
  }

  .msg-action-btn {
    width: 32px;
    height: 32px;
  }

  .version-nav {
    opacity: 1;
  }
}

@media (max-width: 360px) {
  .header-inner {
    padding: 6px 6px;
  }

  .logo-text {
    font-size: 14px;
  }

  .header-logo {
    height: 24px;
  }

  .persona-btn {
    font-size: 13px;
    padding: 3px 5px;
  }

  .header-actions {
    gap: 2px;
  }

  .messages-area {
    padding: 10px 0;
  }

  .messages-inner {
    padding: 0 4px;
  }

  .welcome {
    padding: 20px 4px 18px;
  }

  .welcome-logo {
    width: 100px;
  }

  .welcome-title {
    font-size: 15px;
  }

  .welcome-subtitle {
    font-size: 12px;
  }

  .suggestion-chip {
    padding: 6px 10px;
    font-size: 12px;
    gap: 4px;
  }

  .suggestions {
    gap: 6px;
  }

  .message-wrapper {
    margin-bottom: 10px;
  }

  .message-row {
    gap: 6px;
  }

  .avatar {
    display: none;
  }

  .bubble {
    padding: 8px 10px;
    font-size: 13px;
    border-radius: 10px;
  }

  .bubble-wrapper {
    max-width: 100%;
  }

  .user-bubble {
    max-width: 100%;
  }

  .input-area {
    padding: 4px 6px 8px;
  }

  .input-box :deep(.chat-input .el-textarea__inner) {
    padding: 8px 10px;
    font-size: 13px;
    border-radius: 10px;
  }

  .model-toggle .toggle-btn {
    font-size: 10px;
    padding: 2px 8px;
  }

  .dt-toggle {
    font-size: 10px;
    padding: 2px 8px;
  }

  .prefix-toggle {
    font-size: 10px;
    padding: 2px 8px;
  }

  .websearch-toggle {
    font-size: 10px;
    padding: 2px 8px;
  }

  .re-btn {
    font-size: 10px;
    padding: 1px 6px;
  }

  .upload-btn {
    transform: scale(0.85);
  }

  .send-btn {
    transform: scale(0.85);
  }

  .pending-file-chip {
    font-size: 11px;
    padding: 2px 5px;
  }

  .pending-file-name {
    max-width: 60px;
  }

  .markdown-body pre {
    padding: 10px;
    font-size: 12px;
  }

  .markdown-body :not(pre)>code {
    font-size: 0.85em;
  }

  .reasoning-block {
    font-size: 12px;
  }

  .reasoning-toggle {
    padding: 6px 10px;
  }

  .reasoning-content {
    max-height: 150px;
    padding: 8px 10px;
  }

  .reasoning-text {
    font-size: 11px;
  }

  /* 移动端操作按钮常显（无 hover） */
  .msg-actions {
    opacity: 1;
  }

  .msg-action-btn {
    width: 32px;
    height: 32px;
  }

  .version-nav {
    opacity: 1;
  }
}
</style>
