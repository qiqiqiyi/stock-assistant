<template>
  <div class="chat-view">
    <!-- 左侧对话历史 -->
    <aside class="chat-sidebar">
      <div class="chat-sidebar-header">
        <button class="btn btn-primary new-chat-btn" @click="newChat">
          <span>＋</span> 新对话
        </button>
      </div>
      <div class="chat-history">
        <div
          v-for="(session, i) in chatSessions"
          :key="session.id"
          class="history-item"
          :class="{ active: currentSessionId === session.id }"
          @click="switchSession(session.id)"
        >
          <span class="history-icon">💬</span>
          <div class="history-info">
            <div class="history-title">{{ session.title }}</div>
            <div class="history-time">{{ session.time }}</div>
          </div>
          <button class="del-btn" @click.stop="deleteSession(i)">✕</button>
        </div>
      </div>
    </aside>

    <!-- 主聊天区域 -->
    <div class="chat-main">
      <!-- 消息列表 -->
      <div class="messages-container" ref="messagesEl">
        <!-- 空状态 -->
        <div v-if="!messages.length" class="empty-chat">
          <div class="empty-icon">🤖</div>
          <h2>你好，我是股票投资助手</h2>
          <p>我可以帮你分析个股、查询行情、解读财报、制定投资策略</p>
          <div class="quick-asks">
            <button
              v-for="q in quickQuestions"
              :key="q"
              class="quick-ask-btn"
              @click="sendMessage(q)"
            >{{ q }}</button>
          </div>
        </div>

        <!-- 消息列表 -->
        <template v-else>
          <div
            v-for="msg in messages"
            :key="msg.id"
            class="message-item"
            :class="msg.role"
          >
            <div class="msg-avatar">
              {{ msg.role === 'user' ? '👤' : '🤖' }}
            </div>
            <div class="msg-body">
              <div class="msg-meta">
                <span class="msg-sender">{{ msg.role === 'user' ? '我' : 'AI 助手' }}</span>
                <span class="msg-time">{{ msg.time }}</span>
              </div>
              <div class="msg-content" v-html="renderMessage(msg.content)"></div>
              <!-- 股票卡片 -->
              <div v-if="msg.stockCards && msg.stockCards.length" class="stock-cards">
                <StockCard
                  v-for="s in msg.stockCards"
                  :key="s.code"
                  :stock="s"
                />
              </div>
              <!-- 操作按钮 -->
              <div v-if="msg.role === 'assistant'" class="msg-actions">
                <button class="action-btn" @click="copyMsg(msg.content)">📋 复制</button>
                <button class="action-btn" @click="likeMsg(msg)">{{ msg.liked ? '❤️' : '🤍' }} 有用</button>
              </div>
            </div>
          </div>

          <!-- AI 打字动画 -->
          <div v-if="isTyping" class="message-item assistant typing-row">
            <div class="msg-avatar">🤖</div>
            <div class="msg-body">
              <div class="typing-indicator">
                <span></span><span></span><span></span>
              </div>
            </div>
          </div>
        </template>
      </div>

      <!-- 底部输入区 -->
      <div class="chat-input-area">
        <!-- 快捷功能栏 -->
        <div class="input-toolbar">
          <button
            v-for="tool in tools"
            :key="tool.label"
            class="tool-btn"
            :class="{ active: activeTool === tool.id }"
            @click="toggleTool(tool.id)"
          >
            {{ tool.icon }} {{ tool.label }}
          </button>
        </div>

        <div class="input-row">
          <textarea
            ref="inputEl"
            v-model="inputText"
            :placeholder="inputPlaceholder"
            @keydown.enter.exact.prevent="handleSend"
            @keydown.enter.shift.exact="inputText += '\n'"
            @input="autoResize"
            rows="1"
          ></textarea>
          <button
            class="send-btn"
            :disabled="!inputText.trim() || isTyping"
            @click="handleSend"
          >
            <span v-if="!isTyping">发送</span>
            <span v-else class="sending-spinner">⟳</span>
          </button>
        </div>
        <div class="input-hint">Enter 发送 · Shift+Enter 换行</div>
      </div>
    </div>

    <!-- 右侧上下文面板 -->
    <aside class="context-panel" v-if="showContext">
      <div class="context-header">
        <span>📌 相关信息</span>
        <button @click="showContext = false">✕</button>
      </div>
      <div class="context-body">
        <div class="context-section">
          <div class="context-title">提及的股票</div>
          <div
            v-for="s in mentionedStocks"
            :key="s.code"
            class="context-stock"
          >
            <span class="c-code">{{ s.code }}</span>
            <span class="c-name">{{ s.name }}</span>
            <span class="c-price" :class="s.change >= 0 ? 'up' : 'down'">
              {{ s.price }}
            </span>
          </div>
        </div>
        <div class="context-section">
          <div class="context-title">分析工具</div>
          <button class="ctx-tool-btn" @click="$router.push('/analysis')">📊 K线分析</button>
          <button class="ctx-tool-btn" @click="$router.push('/portfolio')">💼 加入组合</button>
          <button class="ctx-tool-btn" @click="$router.push('/watchlist')">⭐ 添加自选</button>
        </div>
      </div>
    </aside>
  </div>
</template>

<script setup>
import { ref, nextTick, computed } from 'vue'
import StockCard from '@/components/StockCard.vue'
import dayjs from 'dayjs'

const messagesEl = ref(null)
const inputEl = ref(null)
const inputText = ref('')
const isTyping = ref(false)
const activeTool = ref(null)
const showContext = ref(true)
const currentSessionId = ref('1')

const tools = [
  { id: 'stock', icon: '📊', label: '股票行情' },
  { id: 'news', icon: '📰', label: '最新资讯' },
  { id: 'report', icon: '📋', label: '财报分析' },
  { id: 'strategy', icon: '🎯', label: '投资策略' }
]

const quickQuestions = [
  '📈 分析一下茅台最近的走势',
  '🔍 帮我找一些高股息的蓝筹股',
  '📊 今天A股市场整体情况如何？',
  '💡 新能源板块的投资机会在哪里？',
  '⚠️ 如何判断一只股票是否被高估？'
]

const inputPlaceholder = computed(() => {
  const map = {
    stock: '输入股票代码或名称，查询实时行情...',
    news: '输入关键词，搜索相关新闻资讯...',
    report: '输入公司名称，获取财报分析...',
    strategy: '描述你的投资目标，获取策略建议...'
  }
  return map[activeTool.value] || '问我任何关于股票投资的问题...'
})

// 对话历史
const chatSessions = ref([
  { id: '1', title: '茅台投资价值分析', time: '今天 14:30' },
  { id: '2', title: '新能源板块机会', time: '昨天 10:22' },
  { id: '3', title: '年终投资组合复盘', time: '上周五' }
])

const mentionedStocks = ref([
  { code: '600519.SH', name: '贵州茅台', price: '1768.00', change: 1.23 },
  { code: '300750.SZ', name: '宁德时代', price: '215.30', change: 2.41 }
])

// 当前会话消息
const messages = ref([
  {
    id: 1,
    role: 'assistant',
    content: '你好！我是你的AI股票投资助手 📈\n\n我可以帮你：\n- **查询个股行情**和技术面分析\n- **解读财务报告**，分析公司基本面\n- **挖掘投资机会**，识别板块轮动\n- **评估风险收益**，制定投资策略\n\n请告诉我你想了解什么？',
    time: '14:28',
    liked: false,
    stockCards: []
  },
  {
    id: 2,
    role: 'user',
    content: '帮我分析一下贵州茅台 600519 的投资价值',
    time: '14:29'
  },
  {
    id: 3,
    role: 'assistant',
    content: `**贵州茅台 (600519.SH) 投资价值分析** 🥃

**基本面概览**
- 当前股价：**¥1,768.00**（↑ +1.23%）
- 市盈率：**28.6x**（行业均值 32x）
- 市净率：**8.2x**
- ROE：**32.4%**（连续5年 > 30%）
- 股息率：**1.8%**

**竞争优势**
1. ✅ **强大的品牌护城河** — 茅台酱香型白酒具有不可替代的品牌价值
2. ✅ **稀缺产能** — 赤水河特定地理环境限制了产能扩张
3. ✅ **消费升级受益** — 高端白酒消费持续增长

**风险提示**
- ⚠️ 估值相对偏高，短期波动风险
- ⚠️ 政策监管对高端消费的影响
- ⚠️ 宏观经济下行对商务消费的压制

**投资建议**：适合长期持有，当前价格对于长线投资者具备一定吸引力。建议分批建仓，控制仓位在整体组合的 10-15%。`,
    time: '14:30',
    liked: false,
    stockCards: [
      {
        code: '600519.SH',
        name: '贵州茅台',
        price: 1768.00,
        change: 1.23,
        volume: '2.34万手',
        amount: '41.32亿',
        high: 1781.00,
        low: 1750.00,
        pe: 28.6,
        pb: 8.2
      }
    ]
  }
])

let msgId = 10

const scrollToBottom = async () => {
  await nextTick()
  if (messagesEl.value) {
    messagesEl.value.scrollTop = messagesEl.value.scrollHeight
  }
}

const renderMessage = (content) => {
  // 简单的 Markdown 渲染
  return content
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
    .replace(/\*(.+?)\*/g, '<em>$1</em>')
    .replace(/`(.+?)`/g, '<code>$1</code>')
    .replace(/\n/g, '<br/>')
    .replace(/✅/g, '<span class="tag tag-blue">✅</span>')
    .replace(/⚠️/g, '<span class="tag tag-gray">⚠️</span>')
}

const aiResponses = [
  (q) => `正在为您分析 **"${q}"** 相关信息...\n\n根据最新市场数据，我为您整理了以下分析要点：\n\n1. **市场动态** — 当前市场整体处于震荡筑底阶段，成交量有所萎缩\n2. **板块表现** — 相关板块近期表现分化，建议关注龙头标的\n3. **操作建议** — 建议保持谨慎，做好风险控制，仓位不宜过重\n\n如需深入分析，请告诉我具体的股票代码。`,
  (q) => `关于 **"${q}"**，以下是我的分析：\n\n**核心逻辑**\n该方向的投资逻辑主要集中在：政策支持、业绩改善、估值修复三个方面。\n\n**主要风险**\n- 宏观经济不确定性\n- 行业竞争加剧\n- 估值切换压力\n\n**参考标的**\n可重点关注行业龙头，结合个人风险偏好决策。\n\n⚠️ **免责声明**：以上内容仅供参考，不构成投资建议。`,
]

const sendMessage = async (text) => {
  if (!text?.trim()) return
  const content = text || inputText.value.trim()
  if (!content) return

  // 添加用户消息
  messages.value.push({
    id: ++msgId,
    role: 'user',
    content,
    time: dayjs().format('HH:mm')
  })
  inputText.value = ''
  await scrollToBottom()

  // 模拟AI回复
  isTyping.value = true
  await scrollToBottom()

  await new Promise(r => setTimeout(r, 1200 + Math.random() * 1000))

  const response = aiResponses[Math.floor(Math.random() * aiResponses.length)](content)
  messages.value.push({
    id: ++msgId,
    role: 'assistant',
    content: response,
    time: dayjs().format('HH:mm'),
    liked: false,
    stockCards: []
  })
  isTyping.value = false
  await scrollToBottom()
}

const handleSend = () => sendMessage(inputText.value)

const autoResize = (e) => {
  const el = e.target
  el.style.height = 'auto'
  el.style.height = Math.min(el.scrollHeight, 140) + 'px'
}

const copyMsg = (content) => {
  navigator.clipboard.writeText(content).then(() => {
    // Toast
  })
}

const likeMsg = (msg) => { msg.liked = !msg.liked }
const toggleTool = (id) => { activeTool.value = activeTool.value === id ? null : id }
const newChat = () => {
  const id = Date.now().toString()
  chatSessions.value.unshift({ id, title: '新对话', time: '刚刚' })
  currentSessionId.value = id
  messages.value = []
}
const switchSession = (id) => { currentSessionId.value = id }
const deleteSession = (i) => { chatSessions.value.splice(i, 1) }
</script>

<style lang="scss" scoped>
$bg-dark: #0d1117;
$bg-card: #161b22;
$bg-hover: #1c2128;
$border-color: #30363d;
$text-primary: #e6edf3;
$text-secondary: #8b949e;
$primary: #1677ff;

.chat-view {
  display: flex;
  height: 100%;
  overflow: hidden;
}

/* ===== LEFT SIDEBAR ===== */
.chat-sidebar {
  width: 240px;
  min-width: 240px;
  border-right: 1px solid $border-color;
  display: flex;
  flex-direction: column;
  background: $bg-card;
}

.chat-sidebar-header {
  padding: 12px;
  border-bottom: 1px solid $border-color;
}

.new-chat-btn {
  width: 100%;
  padding: 8px;
  font-size: 13px;
}

.chat-history {
  flex: 1;
  overflow-y: auto;
  padding: 8px;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.history-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 10px;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s;

  &:hover {
    background: $bg-hover;
    .del-btn { opacity: 1; }
  }

  &.active {
    background: rgba(22, 119, 255, 0.1);
  }

  .history-icon { font-size: 14px; }
  .history-info { flex: 1; min-width: 0; }
  .history-title {
    font-size: 12px;
    color: $text-primary;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .history-time { font-size: 10px; color: $text-secondary; }

  .del-btn {
    opacity: 0;
    background: transparent;
    border: none;
    color: $text-secondary;
    cursor: pointer;
    font-size: 10px;
    padding: 2px 4px;
    border-radius: 3px;
    &:hover { color: #ff4d4f; }
    transition: opacity 0.15s;
  }
}

/* ===== MAIN ===== */
.chat-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

/* ===== MESSAGES ===== */
.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

/* ===== EMPTY ===== */
.empty-chat {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  padding: 40px;
  margin: auto;

  .empty-icon { font-size: 56px; margin-bottom: 16px; }
  h2 { font-size: 22px; color: $text-primary; margin-bottom: 8px; }
  p { color: $text-secondary; margin-bottom: 24px; max-width: 400px; }
}

.quick-asks {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  justify-content: center;
  max-width: 580px;
}

.quick-ask-btn {
  background: $bg-card;
  border: 1px solid $border-color;
  color: $text-secondary;
  padding: 8px 14px;
  border-radius: 20px;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;

  &:hover {
    border-color: $primary;
    color: $primary;
    background: rgba(22, 119, 255, 0.05);
  }
}

/* ===== MESSAGE ===== */
.message-item {
  display: flex;
  gap: 12px;
  animation: fadeIn 0.3s ease;

  &.user {
    flex-direction: row-reverse;

    .msg-body { align-items: flex-end; }
    .msg-meta { flex-direction: row-reverse; }
    .msg-content {
      background: $primary;
      color: #fff;
    }
  }

  &.assistant {
    .msg-content {
      background: $bg-card;
      border: 1px solid $border-color;
    }
  }
}

.msg-avatar {
  font-size: 20px;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $bg-card;
  border: 1px solid $border-color;
  flex-shrink: 0;
}

.msg-body {
  display: flex;
  flex-direction: column;
  gap: 4px;
  max-width: 70%;
}

.msg-meta {
  display: flex;
  align-items: center;
  gap: 6px;
  .msg-sender { font-size: 11px; color: $text-secondary; font-weight: 500; }
  .msg-time { font-size: 10px; color: #656d76; }
}

.msg-content {
  padding: 10px 14px;
  border-radius: 10px;
  font-size: 13px;
  line-height: 1.7;
  word-break: break-word;

  :deep(strong) { font-weight: 600; }
  :deep(code) {
    background: rgba(255,255,255,0.1);
    padding: 1px 4px;
    border-radius: 3px;
    font-family: monospace;
    font-size: 12px;
  }
}

.stock-cards {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 6px;
}

.msg-actions {
  display: flex;
  gap: 6px;
  margin-top: 4px;
}

.action-btn {
  background: transparent;
  border: 1px solid $border-color;
  color: $text-secondary;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 11px;
  cursor: pointer;
  transition: all 0.15s;
  &:hover { background: $bg-hover; color: $text-primary; }
}

/* ===== TYPING ===== */
.typing-row .msg-body {
  justify-content: center;
}
.typing-indicator {
  display: flex;
  gap: 4px;
  padding: 12px 16px;
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 10px;

  span {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: $text-secondary;
    animation: bounce 1.2s infinite;
    &:nth-child(2) { animation-delay: 0.2s; }
    &:nth-child(3) { animation-delay: 0.4s; }
  }
}

@keyframes bounce {
  0%, 60%, 100% { transform: translateY(0); }
  30% { transform: translateY(-5px); }
}

/* ===== INPUT AREA ===== */
.chat-input-area {
  border-top: 1px solid $border-color;
  padding: 12px 16px;
  background: $bg-card;
}

.input-toolbar {
  display: flex;
  gap: 6px;
  margin-bottom: 10px;
}

.tool-btn {
  background: transparent;
  border: 1px solid $border-color;
  color: $text-secondary;
  padding: 4px 10px;
  border-radius: 16px;
  font-size: 11px;
  cursor: pointer;
  transition: all 0.15s;

  &:hover { border-color: $primary; color: $primary; }
  &.active {
    background: rgba(22, 119, 255, 0.12);
    border-color: $primary;
    color: $primary;
  }
}

.input-row {
  display: flex;
  gap: 10px;
  align-items: flex-end;

  textarea {
    flex: 1;
    background: $bg-dark;
    border: 1px solid $border-color;
    border-radius: 10px;
    padding: 10px 14px;
    color: $text-primary;
    font-size: 13px;
    resize: none;
    outline: none;
    line-height: 1.5;
    min-height: 40px;
    max-height: 140px;
    transition: border-color 0.2s;

    &::placeholder { color: $text-secondary; }
    &:focus { border-color: $primary; }
  }
}

.send-btn {
  background: $primary;
  border: none;
  color: #fff;
  padding: 10px 18px;
  border-radius: 10px;
  cursor: pointer;
  font-size: 13px;
  font-weight: 500;
  transition: all 0.2s;
  white-space: nowrap;

  &:hover:not(:disabled) { background: #0958d9; }
  &:disabled { opacity: 0.5; cursor: not-allowed; }

  .sending-spinner {
    display: inline-block;
    animation: spin 0.8s linear infinite;
  }
}

.input-hint {
  font-size: 10px;
  color: $text-secondary;
  text-align: right;
  margin-top: 5px;
}

/* ===== RIGHT PANEL ===== */
.context-panel {
  width: 220px;
  min-width: 220px;
  border-left: 1px solid $border-color;
  background: $bg-card;
  display: flex;
  flex-direction: column;
}

.context-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 14px;
  border-bottom: 1px solid $border-color;
  font-size: 12px;
  font-weight: 600;
  color: $text-primary;

  button {
    background: transparent;
    border: none;
    color: $text-secondary;
    cursor: pointer;
    font-size: 11px;
    &:hover { color: $text-primary; }
  }
}

.context-body {
  flex: 1;
  overflow-y: auto;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.context-section {
  .context-title {
    font-size: 10px;
    font-weight: 600;
    color: $text-secondary;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    margin-bottom: 8px;
  }
}

.context-stock {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 6px 8px;
  border-radius: 6px;
  border: 1px solid $border-color;
  margin-bottom: 4px;
  font-size: 11px;

  .c-code { color: $text-secondary; }
  .c-name { flex: 1; color: $text-primary; }
  .c-price { font-weight: 500; }
}

.ctx-tool-btn {
  display: block;
  width: 100%;
  text-align: left;
  background: transparent;
  border: 1px solid $border-color;
  color: $text-secondary;
  padding: 6px 10px;
  border-radius: 6px;
  font-size: 11px;
  cursor: pointer;
  margin-bottom: 4px;
  transition: all 0.15s;

  &:hover {
    background: $bg-hover;
    color: $text-primary;
    border-color: #484f58;
  }
}

.up { color: #f5222d; }
.down { color: #52c41a; }

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(6px); }
  to { opacity: 1; transform: translateY(0); }
}
@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
</style>
