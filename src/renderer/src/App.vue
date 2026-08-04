<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { API_BASE_URL } from './config'

type MessageRole = 'user' | 'assistant'

type ChatMessage = {
  id: string
  role: MessageRole
  content: string
}

type Conversation = {
  id: string
  title: string
  messages: ChatMessage[]
}

type AuthUser = {
  id: string
  username: string
  createdAt: string
  updatedAt: string
}

type AuthResponse = {
  user: AuthUser
  token: string
}

type SessionResponse = {
  user: AuthUser
}

const TOKEN_KEY = 'spa.auth.token'
const DEFAULT_GREETING = '你好，先输入一个问题开始吧。'

const conversationsKey = (userId: string) => `spa.conversations.${userId}`
const activeConversationKey = (userId: string) => `spa.activeConversation.${userId}`
const createId = () => crypto.randomUUID()
const createAssistantMessage = (content = DEFAULT_GREETING): ChatMessage => ({
  id: createId(),
  role: 'assistant',
  content
})
const createConversation = (title = '新对话', greeting = DEFAULT_GREETING): Conversation => ({
  id: createId(),
  title,
  messages: [createAssistantMessage(greeting)]
})
const readJson = <T,>(key: string, fallback: T): T => {
  const raw = localStorage.getItem(key)

  if (!raw) {
    return fallback
  }

  try {
    return JSON.parse(raw) as T
  } catch {
    return fallback
  }
}
const summarizeTitle = (content: string) => {
  const cleaned = content.trim().replace(/\s+/g, ' ')
  return cleaned.length > 18 ? `${cleaned.slice(0, 18)}…` : cleaned
}

const loginName = ref('')
const password = ref('')
const authMode = ref<'login' | 'register'>('login')
const authError = ref('')
const isSessionChecking = ref(true)
const isAuthenticating = ref(false)
const authToken = ref('')
const authUser = ref<AuthUser | null>(null)
const conversations = ref<Conversation[]>([])
const activeConversationId = ref('')
const question = ref('')

const requestJson = async <T,>(path: string, options: { method?: string; body?: unknown; token?: string } = {}) => {
  try {
    const response = await fetch(`${API_BASE_URL}${path}`, {
      method: options.method ?? 'GET',
      headers: {
        ...(options.body ? { 'Content-Type': 'application/json' } : {}),
        ...(options.token ? { Authorization: `Bearer ${options.token}` } : {})
      },
      body: options.body ? JSON.stringify(options.body) : undefined
    })

    const payload = await response.json().catch(() => null)

    if (!response.ok) {
      throw new Error(payload?.message || '请求失败')
    }

    return payload as T
  } catch (error) {
    if (error instanceof TypeError) {
      throw new Error('无法连接后台服务，请先启动 backend')
    }

    throw error
  }
}

const loadUserState = (userId: string) => {
  const storedConversations = readJson<unknown>(conversationsKey(userId), [])
  const loadedConversations = Array.isArray(storedConversations) ? (storedConversations as Conversation[]) : []

  conversations.value = loadedConversations.length > 0 ? loadedConversations : [createConversation()]

  const storedActiveId = localStorage.getItem(activeConversationKey(userId)) ?? ''
  const activeExists = storedActiveId && conversations.value.some((conversation) => conversation.id === storedActiveId)

  activeConversationId.value = activeExists ? storedActiveId : conversations.value[0].id
  saveUserState()
}

const saveUserState = () => {
  const userId = authUser.value?.id

  if (!userId) {
    return
  }

  localStorage.setItem(conversationsKey(userId), JSON.stringify(conversations.value))
  localStorage.setItem(activeConversationKey(userId), activeConversationId.value)
}

const clearAuthState = () => {
  authToken.value = ''
  authUser.value = null
  conversations.value = []
  activeConversationId.value = ''
  loginName.value = ''
  password.value = ''
  authError.value = ''
  question.value = ''
}

const restoreSession = async () => {
  const storedToken = localStorage.getItem(TOKEN_KEY)

  if (!storedToken) {
    isSessionChecking.value = false
    return
  }

  authToken.value = storedToken

  try {
    const session = await requestJson<SessionResponse>('/auth/session', {
      token: storedToken
    })

    authUser.value = session.user
    loadUserState(session.user.id)
  } catch {
    localStorage.removeItem(TOKEN_KEY)
    clearAuthState()
  } finally {
    isSessionChecking.value = false
  }
}

const submitAuth = async () => {
  const username = loginName.value.trim()
  const secret = password.value.trim()

  if (!username || !secret) {
    authError.value = '请输入用户名和密码'
    return
  }

  isAuthenticating.value = true
  authError.value = ''

  try {
    const path = authMode.value === 'register' ? '/auth/register' : '/auth/login'
    const result = await requestJson<AuthResponse>(path, {
      method: 'POST',
      body: {
        username,
        password: secret
      }
    })

    authToken.value = result.token
    authUser.value = result.user
    localStorage.setItem(TOKEN_KEY, result.token)
    loadUserState(result.user.id)
    password.value = ''
  } catch (error) {
    authError.value = error instanceof Error ? error.message : '认证失败'
  } finally {
    isAuthenticating.value = false
  }
}

const handleLogout = async () => {
  const token = authToken.value

  try {
    if (token) {
      await requestJson('/auth/logout', {
        method: 'POST',
        token
      })
    }
  } finally {
    localStorage.removeItem(TOKEN_KEY)
    clearAuthState()
  }
}

const currentConversation = computed(
  () => conversations.value.find((conversation) => conversation.id === activeConversationId.value) ?? null
)

const currentMessages = computed(() => currentConversation.value?.messages ?? [])

const selectConversation = (id: string) => {
  activeConversationId.value = id
  saveUserState()
}

const startNewConversation = () => {
  const nextConversation = createConversation()
  conversations.value = [nextConversation, ...conversations.value]
  activeConversationId.value = nextConversation.id
  question.value = ''
  saveUserState()
}

const submitQuestion = () => {
  const content = question.value.trim()
  const conversation = currentConversation.value

  if (!content || !conversation) {
    return
  }

  conversation.messages.push(
    {
      id: createId(),
      role: 'user',
      content
    },
    {
      id: createId(),
      role: 'assistant',
      content: `你刚才问的是：${content}`
    }
  )

  if (conversation.title === '新对话') {
    conversation.title = summarizeTitle(content)
  }

  question.value = ''
  saveUserState()
}

onMounted(() => {
  void restoreSession()
})
</script>

<template>
  <div v-if="isSessionChecking" class="login-screen">
    <section class="login-card">
      <p class="eyebrow">Simple Plan Agent</p>
      <h1>正在连接后台</h1>
      <p class="description">正在检查会话状态，请稍候。</p>
    </section>
  </div>

  <div v-else-if="!authUser" class="login-screen">
    <section class="login-card">
      <p class="eyebrow">Simple Plan Agent</p>
      <h1>{{ authMode === 'register' ? '注册' : '登录' }}</h1>
      <p class="description">
        {{ authMode === 'register' ? '创建新账号并进入后台登录。' : '使用后台账号继续你的对话。' }}
      </p>

      <div class="auth-switch" role="tablist" aria-label="登录方式">
        <button type="button" :class="{ active: authMode === 'login' }" @click="authMode = 'login'">登录</button>
        <button
          type="button"
          :class="{ active: authMode === 'register' }"
          @click="authMode = 'register'"
        >
          注册
        </button>
      </div>

      <form class="login-form" @submit.prevent="submitAuth">
        <label class="field">
          <span>用户名</span>
          <input v-model="loginName" type="text" placeholder="输入用户名" autocomplete="username" />
        </label>

        <label class="field">
          <span>密码</span>
          <input v-model="password" type="password" placeholder="输入密码" autocomplete="current-password" />
        </label>

        <p v-if="authError" class="error">{{ authError }}</p>

        <button type="submit" :disabled="isAuthenticating">
          {{ isAuthenticating ? (authMode === 'register' ? '注册中…' : '登录中…') : authMode === 'register' ? '注册' : '登录' }}
        </button>
      </form>
    </section>
  </div>

  <div v-else class="chat-app">
    <aside class="sidebar">
      <div class="sidebar-header">
        <div class="account">
          <span class="account-name">{{ authUser.username }}</span>
          <span class="account-note">后台登录</span>
        </div>
        <button class="ghost-button" type="button" @click="handleLogout">退出</button>
      </div>

      <button class="new-chat" type="button" @click="startNewConversation">+ 新对话</button>

      <nav class="conversation-list" aria-label="对话列表">
        <button
          v-for="conversation in conversations"
          :key="conversation.id"
          class="conversation-item"
          :class="{ active: activeConversationId === conversation.id }"
          type="button"
          @click="selectConversation(conversation.id)"
        >
          {{ conversation.title }}
        </button>
      </nav>
    </aside>

    <main class="chat-main">
      <header class="chat-header">
        <div>
          <p class="eyebrow">Simple Plan Agent</p>
          <h1>{{ currentConversation?.title ?? '新对话' }}</h1>
        </div>
      </header>

      <section class="messages" aria-live="polite">
        <article
          v-for="message in currentMessages"
          :key="message.id"
          class="message"
          :class="message.role"
        >
          <div class="avatar">{{ message.role === 'user' ? '你' : 'AI' }}</div>
          <div class="bubble">{{ message.content }}</div>
        </article>
      </section>

      <form class="composer" @submit.prevent="submitQuestion">
        <textarea
          v-model="question"
          rows="1"
          placeholder="输入你的问题..."
          @keydown.enter.exact.prevent="submitQuestion"
        />
        <button type="submit">发送</button>
      </form>
    </main>
  </div>
</template>
