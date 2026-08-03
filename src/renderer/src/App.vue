<script setup lang="ts">
import { computed, ref } from 'vue'

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
  username: string
}

const AUTH_KEY = 'spa.auth.username'
const DEFAULT_GREETING = '你好，先输入一个问题开始吧。'

const conversationsKey = (username: string) => `spa.conversations.${username}`
const activeConversationKey = (username: string) => `spa.activeConversation.${username}`
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
const loginError = ref('')
const authUser = ref<AuthUser | null>(null)
const conversations = ref<Conversation[]>([])
const activeConversationId = ref('')
const question = ref('')

const loadUserState = (username: string) => {
  const storedConversations = readJson<unknown>(conversationsKey(username), [])
  const loadedConversations = Array.isArray(storedConversations) ? (storedConversations as Conversation[]) : []

  conversations.value = loadedConversations.length > 0 ? loadedConversations : [createConversation()]

  const storedActiveId = localStorage.getItem(activeConversationKey(username)) ?? ''
  const activeExists = storedActiveId && conversations.value.some((conversation) => conversation.id === storedActiveId)

  activeConversationId.value = activeExists ? storedActiveId : conversations.value[0].id
}

const saveUserState = () => {
  const username = authUser.value?.username

  if (!username) {
    return
  }

  localStorage.setItem(AUTH_KEY, username)
  localStorage.setItem(conversationsKey(username), JSON.stringify(conversations.value))
  localStorage.setItem(activeConversationKey(username), activeConversationId.value)
}

const handleLogin = () => {
  const username = loginName.value.trim()
  const secret = password.value.trim()

  if (!username || !secret) {
    loginError.value = '请输入用户名和密码'
    return
  }

  authUser.value = { username }
  loginError.value = ''
  loadUserState(username)
  saveUserState()
  password.value = ''
}

const handleLogout = () => {
  authUser.value = null
  loginName.value = ''
  password.value = ''
  loginError.value = ''
  question.value = ''
  conversations.value = []
  activeConversationId.value = ''
  localStorage.removeItem(AUTH_KEY)
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

const savedUsername = localStorage.getItem(AUTH_KEY)

if (savedUsername) {
  authUser.value = { username: savedUsername }
  loadUserState(savedUsername)
}
</script>

<template>
  <div v-if="!authUser" class="login-screen">
    <section class="login-card">
      <p class="eyebrow">Simple Plan Agent</p>
      <h1>登录</h1>
      <p class="description">本地登录，仅用于区分不同对话历史。</p>

      <form class="login-form" @submit.prevent="handleLogin">
        <label class="field">
          <span>用户名</span>
          <input v-model="loginName" type="text" placeholder="输入用户名" autocomplete="username" />
        </label>

        <label class="field">
          <span>密码</span>
          <input v-model="password" type="password" placeholder="输入密码" autocomplete="current-password" />
        </label>

        <p v-if="loginError" class="error">{{ loginError }}</p>

        <button type="submit">登录</button>
      </form>
    </section>
  </div>

  <div v-else class="chat-app">
    <aside class="sidebar">
      <div class="sidebar-header">
        <div class="account">
          <span class="account-name">{{ authUser.username }}</span>
          <span class="account-note">本地登录</span>
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