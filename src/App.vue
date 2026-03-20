<script setup lang="ts">
import type { DataConnection } from 'peerjs'
import { ArrowRight, Clipboard, LoaderCircle, Lock } from 'lucide-vue-next'
import Peer from 'peerjs'
import { nextTick, onMounted, onUnmounted, ref } from 'vue'
import { Button } from '@/components/ui/button'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { Input } from '@/components/ui/input'
import { Separator } from '@/components/ui/separator'
import { Textarea } from './components/ui/textarea'

type Status = 'init' | 'ready' | 'connecting' | 'connected' | 'disconnected'

interface Message {
  id: number
  text: string
  from: 'me' | 'them'
  time: string
}

const status = ref<Status>('init')
const myId = ref('')
const peerId = ref('')
const messages = ref<Message[]>([])
const inputText = ref('')
const copyLabel = ref('复制')
const errorMsg = ref('')
const messagesEl = ref<HTMLElement | null>(null)

let peer: Peer | null = null
let conn: DataConnection | null = null
let msgCounter = 0

function genId() {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'
  const pool = chars.split('')
  return Array.from({ length: 6 }, () => {
    const i = Math.floor(Math.random() * pool.length)
    return pool.splice(i, 1)[0]
  }).join('')
}

function now() {
  const d = new Date()
  return `${String(d.getHours()).padStart(2, '0')}:${String(d.getMinutes()).padStart(2, '0')}`
}

function addMessage(text: string, from: 'me' | 'them') {
  messages.value.push({ id: ++msgCounter, text, from, time: now() })
  nextTick(() => {
    if (messagesEl.value)
      messagesEl.value.scrollTop = messagesEl.value.scrollHeight
  })
}

function addSystem(text: string) {
  // 用 from: 'them' 占位，CSS 里单独处理 system 类
  messages.value.push({ id: ++msgCounter, text, from: 'them', time: '' })
}

function initPeer() {
  const id = genId()
  myId.value = id

  peer = new Peer(id, {
    host: '0.peerjs.com',
    port: 443,
    path: '/',
    secure: true,
    config: { iceServers: [{ urls: 'stun:stun.l.google.com:19302' }] },
  })

  peer.on('open', () => {
    status.value = 'ready'
  })

  // 被动等待对方连入
  peer.on('connection', (c) => {
    conn = c
    bindConn()
  })

  peer.on('error', (e) => {
    errorMsg.value = `PeerJS 错误：${e.message}`
    status.value = 'disconnected'
  })
}

function bindConn() {
  if (!conn)
    return
  status.value = 'connecting'

  conn.on('open', () => {
    status.value = 'connected'
    addSystem('加密连接已建立（DTLS over WebRTC）')
  })

  conn.on('data', (data) => {
    if (typeof data === 'string')
      addMessage(data, 'them')
  })

  conn.on('close', () => {
    status.value = 'disconnected'
    addSystem('对方已断开连接')
  })

  conn.on('error', (e) => {
    errorMsg.value = e.message
  })
}

// 主动连接对方
function connect() {
  if (!peer || !peerId.value.trim())
    return
  errorMsg.value = ''
  const id = peerId.value.trim().toUpperCase()

  if (id === myId.value) {
    errorMsg.value = '不能连接自己'
    return
  }

  status.value = 'connecting'
  conn = peer.connect(id, { reliable: true })
  bindConn()

  // 超时检测
  const timer = setTimeout(() => {
    if (status.value === 'connecting') {
      errorMsg.value = '连接超时，请确认对方已打开页面'
      status.value = 'ready'
    }
  }, 12_000)

  conn.on('open', () => clearTimeout(timer))
}

function sendMessage() {
  if (!inputText.value.trim() || !conn?.open)
    return
  const text = inputText.value.trim()
  conn.send(text)
  addMessage(text, 'me')
  inputText.value = ''
}

function handleKeydown(e: KeyboardEvent) {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault()
    sendMessage()
  }
}

async function copyId() {
  await navigator.clipboard.writeText(myId.value)
  copyLabel.value = '已复制 ✓'
  setTimeout(() => (copyLabel.value = '复制'), 1500)
}

onMounted(initPeer)

onUnmounted(() => {
  conn?.close()
  peer?.destroy()
})
</script>

<template>
  <div class="flex items-center justify-center h-screen bg-background ">
    <Card v-if="status !== 'connected'" class="w-full max-w-sm">
      <CardHeader>
        <CardTitle class="text-2xl">
          Tomsg
        </CardTitle>
        <CardDescription>
          P2P 加密聊天，消息经 DTLS 端到端加密，阅后即焚
        </CardDescription>
      </CardHeader>
      <CardContent>
        <div class="flex flex-col gap-2">
          <label class="text-xs text-[#bbb]">你的连接码</label>
          <div class="flex items-center justify-center p-2 gap-2 rounded-md border border-dashed">
            <span class="text-2xl tracking-[0.5em]">{{ myId }}</span>
            <Button variant="outline" size="icon" :disabled="!myId" class="hover:cursor-pointer" @click="copyId">
              <Clipboard />
            </Button>
          </div>
        </div>
        <div class="flex items-center gap-4 my-4">
          <Separator class="flex-1" />
          <span class="text-xs text-[#bbb] whitespace-nowrap">等待连接或</span>
          <Separator class="flex-1" />
        </div>
        <div class="flex gap-2">
          <Input
            v-model="peerId"
            class="input"
            placeholder="输入对方的连接码"
            maxlength="6"
            :disabled="status === 'connecting'"
            @input="peerId = peerId.toUpperCase()"
            @keydown.enter="connect"
          />
          <Button
            :disabled="status !== 'ready' || peerId.length < 6"
            @click="connect"
          >
            <LoaderCircle v-if="status === 'connecting'" class="animate-spin" />
            <ArrowRight v-else />
          </Button>
        </div>

        <div class="flex items-center justify-center mt-4">
          <div v-if="errorMsg" class="text-red-500 text-xs">
            {{ errorMsg }}
          </div>
          <div v-if="status === 'ready'" class="text-[#bbb] text-xs">
            服务器已就绪，等待连接
          </div>
          <div v-else class="text-[#bbb] text-xs">
            正在连接信令服务器...
          </div>
        </div>
      </CardContent>
    </Card>

    <Card v-else class="w-full max-w-lg">
      <CardHeader>
        <div class="flex py-4 justify-between border-b">
          <div class="flex items-center gap-2">
            <span class="relative flex size-2">
              <span class="absolute inline-flex h-full w-full animate-ping rounded-full bg-green-400 opacity-75" />
              <span class="relative inline-flex size-2 rounded-full bg-green-500" />
            </span>
            <span class="text-xs text-[#bbb]">已连接</span>
          </div>
          <div class="flex items-center gap-2 text-green-400">
            <Lock class="size-3" />
            <div class="text-xs">
              DTLS 加密
            </div>
          </div>
        </div>
      </CardHeader>
      <CardContent>
        <div ref="messagesEl" class="messages">
          <template v-for="msg in messages" :key="msg.id">
            <div v-if="!msg.time" class="text-center text-xs text-[#bbb]">
              {{ msg.text }}
            </div>
            <!-- 普通消息 -->
            <div v-else class="bubble" :class="[msg.from]">
              <div class="bubble-text">
                {{ msg.text }}
              </div>
              <div class="bubble-time">
                {{ msg.time }}
              </div>
            </div>
          </template>
        </div>

        <!-- 输入栏 -->
        <div class="input-bar">
          <Textarea
            v-model="inputText"
            placeholder="发送消息...（Enter 发送，Shift+Enter 换行）"
            rows="1"
            @keydown="handleKeydown"
          />
          <button
            class="send-btn"
            :disabled="!inputText.trim()"
            @click="sendMessage"
          >
            <svg
              viewBox="0 0 24 24" width="18" height="18" fill="none"
              stroke="currentColor" stroke-width="2"
              stroke-linecap="round" stroke-linejoin="round"
            >
              <line x1="22" y1="2" x2="11" y2="13" />
              <polygon points="22 2 15 22 11 13 2 9 22 2" />
            </svg>
          </button>
        </div>
      </CardContent>
    </Card>
  </div>
</template>

<style scoped>
/* ── 聊天面板 ── */
.chat-card {
  width: 420px;
  height: 580px;
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 1px 4px rgba(0,0,0,.08);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.bubble {
  max-width: 72%;
  display: flex;
  flex-direction: column;
}
.bubble.me { align-self: flex-end; align-items: flex-end; }
.bubble.them { align-self: flex-start; align-items: flex-start; }

.bubble-text {
  padding: 8px 12px;
  border-radius: 14px;
  font-size: 14px;
  line-height: 1.5;
  word-break: break-word;
  white-space: pre-wrap;
}
.bubble.me .bubble-text {
  background: #2563eb;
  color: #fff;
  border-bottom-right-radius: 3px;
}
.bubble.them .bubble-text {
  background: #f0f0f0;
  color: #111;
  border-bottom-left-radius: 3px;
}

.bubble-time {
  font-size: 10px;
  color: #bbb;
  margin-top: 3px;
  padding: 0 4px;
}

.input-bar {
  display: flex;
  align-items: flex-end;
  gap: 8px;
  padding: 10px 12px;
  border-top: 1px solid #f0f0f0;
  flex-shrink: 0;
}

.send-btn {
  width: 38px; height: 38px;
  border-radius: 50%;
  background: #2563eb;
  border: none;
  cursor: pointer;
  display: flex; align-items: center; justify-content: center;
  flex-shrink: 0;
  color: #fff;
  transition: opacity .15s;
}
.send-btn:disabled { opacity: .3; cursor: default; }
.send-btn:hover:not(:disabled) { background: #1d4ed8; }
</style>
