<script setup lang="ts">
import type { DataConnection } from 'peerjs'
import { useDark, useToggle } from '@vueuse/core'
import { ArrowRight, Clipboard, Loader2, Lock, Moon, SendHorizontal, Sun } from 'lucide-vue-next'
import Peer from 'peerjs'
import { computed, nextTick, onMounted, onUnmounted, ref, useTemplateRef, watch } from 'vue'
import { Button } from '@/components/ui/button'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { Input } from '@/components/ui/input'
import { Separator } from '@/components/ui/separator'

type Status = 'init' | 'ready' | 'connecting' | 'connected' | 'disconnected'
type MessageFrom = 'me' | 'other'

interface Message {
  id: number
  text: string
  from: MessageFrom
  time: string
}

const status = ref<Status>('init')
const connectId = ref('')
const peerId = ref('')
const messages = ref<Message[]>([])
const inputText = ref('')
const copyLabel = ref('复制')
const errorMsg = ref('')
const safetyCode = ref('')
const composerFocused = ref(false)

const messagesEl = useTemplateRef<HTMLElement>('messagesEl')
const composerEl = useTemplateRef<HTMLTextAreaElement>('composerEl')

const isDark = useDark({
  selector: 'html',
  attribute: 'class',
  valueDark: 'dark',
  valueLight: '',
  storageKey: 'tomsg-color-scheme',
})
const toggleDark = useToggle(isDark)

let peer: Peer | null = null
let conn: DataConnection | null = null
let msgId = 0

const canConnect = computed(() => status.value === 'ready' && peerId.value.length >= 6)
const composerRaised = computed(() => composerFocused.value || !!inputText.value.trim())

function genId() {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'
  const pool = chars.split('')
  return Array.from({ length: 6 }, () => {
    const i = Math.floor(Math.random() * pool.length)
    return pool.splice(i, 1)[0]
  }).join('')
}

function timestamp() {
  const d = new Date()
  return `${String(d.getHours()).padStart(2, '0')}:${String(d.getMinutes()).padStart(2, '0')}`
}

function pushMessage(text: string, from: MessageFrom, time = timestamp()) {
  messages.value.push({ id: ++msgId, text, from, time })
  nextTick(() => { messagesEl.value && (messagesEl.value.scrollTop = messagesEl.value.scrollHeight) })
}

const pushSystem = (text: string) => pushMessage(text, 'other', '')

function resizeComposer() {
  const el = composerEl.value
  if (!el) {
    return
  }
  const max = Math.max(160, Math.floor(window.innerHeight * 0.34))
  el.style.height = '0px'
  const next = Math.min(Math.max(el.scrollHeight, 44), max)
  el.style.height = `${next}px`
  el.style.overflowY = el.scrollHeight > next ? 'auto' : 'hidden'
}

watch(inputText, () => nextTick(resizeComposer))

async function getFingerprint() {
  const connection = (conn as any)?.peerConnection as RTCPeerConnection | undefined
  if (!connection) {
    return '未获取到连接'
  }
  const stats = await connection.getStats()
  let fp = ''
  stats.forEach((s: any) => {
    if (s.type === 'certificate' && s.fingerprint)
      fp = (s.fingerprint as string).split(':').slice(-6).join(':')
  })
  return fp || '未获取到指纹'
}

function bindConn() {
  if (!conn) {
    return
  }
  status.value = 'connecting'
  safetyCode.value = ''

  conn.on('open', async () => {
    status.value = 'connected'
    safetyCode.value = await getFingerprint()
    pushSystem('请与对方核对安全码以防止中间人攻击。')
  })
  conn.on('data', (data) => {
    typeof data === 'string' && pushMessage(data, 'other')
  })
  conn.on('close', () => {
    status.value = 'disconnected'
    safetyCode.value = ''; pushSystem('对方已断开连接')
  })
  conn.on('error', (e) => {
    errorMsg.value = e.message
  })
}

function initPeer() {
  connectId.value = genId()
  peer = new Peer(connectId.value, {
    host: '0.peerjs.com',
    port: 443,
    path: '/',
    secure: true,
    config: {
      iceServers: [
        {
          urls: 'stun:stun.l.google.com:19302',
        },
      ],
    },
  })
  peer.on('open', () => {
    status.value = 'ready'
  })
  peer.on('connection', (c) => {
    conn = c
    bindConn()
  })
  peer.on('error', (e) => {
    errorMsg.value = `PeerJS 错误：${e.message}`
    status.value = 'disconnected'
  })
}

function connect() {
  if (!peer || !peerId.value.trim()) {
    return
  }
  const id = peerId.value.trim().toUpperCase()
  if (id === connectId.value) {
    errorMsg.value = '不能连接自己'
    return
  }

  errorMsg.value = ''
  status.value = 'connecting'
  safetyCode.value = ''
  conn = peer.connect(id, { reliable: true })
  bindConn()

  const timer = setTimeout(() => {
    if (status.value === 'connecting') {
      errorMsg.value = '连接超时，请确认对方已打开页面'
      status.value = 'ready'
    }
  }, 10000)
  conn.on('open', () => clearTimeout(timer))
}

function sendMessage() {
  const text = inputText.value.trim()
  if (!text || !conn?.open) {
    return
  }
  conn.send(text)
  pushMessage(text, 'me')
  inputText.value = ''
  nextTick(resizeComposer)
}

function handleKeydown(e: KeyboardEvent) {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault()
    sendMessage()
  }
}

async function copyId() {
  await navigator.clipboard.writeText(connectId.value)
  copyLabel.value = '已复制'
  setTimeout(() => {
    copyLabel.value = '复制'
  }, 1500)
}

onMounted(() => {
  initPeer()
  nextTick(resizeComposer)
  window.addEventListener('resize', resizeComposer)
})

onUnmounted(() => {
  conn?.close()
  peer?.destroy()
  window.removeEventListener('resize', resizeComposer)
})
</script>

<template>
  <div class="relative min-h-dvh bg-background dark:bg-black">
    <main class="relative mx-auto flex min-h-dvh w-full max-w-5xl flex-col items-center justify-start p-4 pt-16 sm:justify-center sm:p-6 sm:pt-20">
      <Button
        variant="outline"
        size="icon"
        class="absolute right-4 top-4 sm:right-6 sm:top-6 hover:cursor-pointer"
        @click="toggleDark()"
      >
        <Sun v-if="isDark" />
        <Moon v-else />
      </Button>
      <Card v-if="status !== 'connected'" class="w-full max-w-md gap-0 border-border/70 bg-card/80">
        <CardHeader class="pb-8">
          <CardTitle class="text-2xl tracking-tight">
            Tomsg
          </CardTitle>
          <CardDescription class="leading-relaxed">
            P2P 加密聊天，消息经 DTLS 端到端加密，阅后即焚
          </CardDescription>
        </CardHeader>
        <CardContent class="space-y-4">
          <div class="flex flex-col gap-2">
            <span class="text-xs text-muted-foreground">你的连接码</span>
            <div class="flex items-center justify-between gap-2 rounded-lg border border-dashed px-4 py-2">
              <span class="truncate font-mono text-xl tracking-[0.9em] sm:text-2xl">{{ connectId }}</span>
              <Button variant="secondary" :disabled="!connectId" @click="copyId">
                <Clipboard class="size-4" />
                <span class="text-xs">{{ copyLabel }}</span>
              </Button>
            </div>
          </div>
          <div class="flex items-center gap-4">
            <Separator class="flex-1" />
            <span class="whitespace-nowrap text-xs text-muted-foreground">等待连接或</span>
            <Separator class="flex-1" />
          </div>
          <div class="flex gap-2">
            <Input
              v-model="peerId"
              class="uppercase placeholder:text-xs"
              placeholder="输入对方的连接码"
              maxlength="6"
              :disabled="status === 'connecting'"
              @input="peerId = peerId.toUpperCase()"
              @keydown.enter="connect"
            />
            <Button variant="outline" size="icon" :disabled="!canConnect" @click="connect">
              <Loader2 v-if="status === 'connecting'" class="size-4 animate-spin" />
              <ArrowRight v-else class="size-4" />
            </Button>
          </div>
          <p class="text-center text-xs" :class="errorMsg ? 'text-destructive' : 'text-muted-foreground'">
            <template v-if="errorMsg">
              {{ errorMsg }}
            </template>
            <template v-else-if="status === 'ready'">
              服务器已就绪，等待连接
            </template>
            <template v-else>
              正在连接信令服务器...
            </template>
          </p>
        </CardContent>
      </Card>
      <Card v-else class="h-[calc(100dvh-6.5rem)] w-full max-w-3xl gap-0 overflow-hidden bg-card/80 sm:h-[calc(100dvh-8rem)] sm:max-h-180">
        <CardHeader class="border-b pb-4">
          <div class="flex items-center justify-between">
            <div class="flex items-center gap-2">
              <span class="relative flex size-2">
                <span class="absolute inline-flex size-2 animate-ping rounded-full bg-emerald-400 opacity-75" />
                <span class="relative inline-flex size-2 rounded-full bg-emerald-500" />
              </span>
              <span class="text-xs text-muted-foreground">WebRTC 已连接</span>
            </div>
            <div class="flex flex-col gap-1">
              <div class="flex items-center justify-end gap-2 text-emerald-600 dark:text-emerald-400">
                <Lock class="size-3" />
                <span class="text-xs">DTLS 加密</span>
              </div>
              <p v-if="safetyCode" class="text-xs text-muted-foreground">
                安全码 {{ safetyCode }}
              </p>
            </div>
          </div>
        </CardHeader>
        <CardContent class="flex h-full min-h-0 flex-1 flex-col pt-4">
          <div ref="messagesEl" class="flex min-h-0 flex-1 flex-col gap-4 overflow-y-auto p-3 sm:p-4">
            <template v-for="msg in messages" :key="msg.id">
              <p v-if="!msg.time" class="text-center text-xs text-muted-foreground">
                {{ msg.text }}
              </p>
              <div
                v-else
                class="flex max-w-[85%] flex-col gap-1 sm:max-w-[72%]"
                :class="msg.from === 'me' ? 'ml-auto items-end' : 'mr-auto items-start'"
              >
                <div
                  class="wrap-break-word whitespace-pre-wrap rounded-2xl px-3 py-2 text-sm leading-6"
                  :class="msg.from === 'me' ? 'rounded-br-sm bg-primary text-primary-foreground' : 'rounded-bl-sm bg-muted text-foreground'"
                >
                  {{ msg.text }}
                </div>
                <span class="px-1 text-[10px] text-muted-foreground">{{ msg.time }}</span>
              </div>
            </template>
          </div>
          <div
            class="mt-3 rounded-2xl border bg-background/70 p-2 transition-all duration-250"
            :class="composerRaised ? '-translate-y-2 ring-1 ring-primary/20' : 'shadow-sm'"
          >
            <div class="flex items-center gap-2">
              <textarea
                ref="composerEl"
                v-model="inputText"
                rows="1"
                placeholder="发送消息...（Enter 发送，Shift+Enter 换行）"
                class="w-full resize-none bg-transparent px-2 py-2 text-sm leading-6 outline-none placeholder:text-muted-foreground"
                @focus="composerFocused = true"
                @blur="composerFocused = false"
                @keydown="handleKeydown"
              />
              <Button
                size="icon"
                class="size-10 shrink-0 rounded-full transition-transform duration-200"
                :class="inputText.trim() ? 'scale-100' : 'scale-95'"
                :disabled="!inputText.trim()"
                @click="sendMessage"
              >
                <SendHorizontal class="size-4" />
              </Button>
            </div>
          </div>
        </CardContent>
      </Card>
    </main>
  </div>
</template>
