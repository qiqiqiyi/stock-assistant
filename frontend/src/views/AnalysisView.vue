<template>
  <div class="analysis-view">
    <!-- 顶部股票搜索 + 切换 -->
    <div class="analysis-header">
      <div class="stock-selector">
        <button
          v-for="s in recentStocks"
          :key="s.code"
          class="recent-btn"
          :class="{ active: currentStock.code === s.code }"
          @click="switchStock(s)"
        >{{ s.name }}</button>
        <input v-model="searchCode" placeholder="输入代码..." class="code-input" @keyup.enter="addByCode" />
      </div>
      <div class="indicator-selector">
        <button
          v-for="ind in indicators"
          :key="ind"
          class="ind-btn"
          :class="{ active: activeIndicators.includes(ind) }"
          @click="toggleIndicator(ind)"
        >{{ ind }}</button>
      </div>
    </div>

    <!-- 主图区域 -->
    <div class="chart-main">
      <div class="main-chart-header">
        <div class="mc-name">
          {{ currentStock.name }}
          <span class="mc-code">{{ currentStock.code }}</span>
        </div>
        <div class="mc-price" :class="currentStock.change >= 0 ? 'up' : 'down'">
          {{ currentStock.price }}
          <span class="mc-pct">{{ currentStock.change >= 0 ? '+' : '' }}{{ currentStock.change }}%</span>
        </div>
        <div class="kline-type-btns">
          <button
            v-for="t in klineTypes"
            :key="t.id"
            :class="{ active: klineType === t.id }"
            @click="klineType = t.id; generateKlineData()"
          >{{ t.label }}</button>
        </div>
      </div>

      <!-- K线图 (SVG模拟) -->
      <div class="kline-chart">
        <svg :viewBox="`0 0 ${cW} ${cH}`" preserveAspectRatio="none" class="kline-svg">
          <!-- 网格 -->
          <line v-for="y in [60, 120, 180, 240]" :key="`hg${y}`"
            x1="0" :y1="y" :x2="cW" :y2="y"
            stroke="#1c2128" stroke-width="0.8"/>
          <line v-for="x in [100, 200, 300, 400, 500]" :key="`vg${x}`"
            :x1="x" y1="0" :x2="x" :y2="cH"
            stroke="#1c2128" stroke-width="0.8"/>

          <!-- K线实体 -->
          <g v-for="(k, i) in klineData" :key="i">
            <!-- 上下影线 -->
            <line
              :x1="k.x + k.w/2" :y1="k.highY"
              :x2="k.x + k.w/2" :y2="k.lowY"
              :stroke="k.up ? '#f5222d' : '#52c41a'"
              stroke-width="1"
            />
            <!-- 实体 -->
            <rect
              :x="k.x" :y="Math.min(k.openY, k.closeY)"
              :width="k.w"
              :height="Math.max(2, Math.abs(k.openY - k.closeY))"
              :fill="k.up ? '#f5222d' : '#52c41a'"
              :opacity="k.up ? 0.85 : 0.85"
            />
          </g>

          <!-- MA线 -->
          <path v-if="activeIndicators.includes('MA5')" :d="ma5Path" stroke="#f6c90e" stroke-width="1" fill="none" opacity="0.8"/>
          <path v-if="activeIndicators.includes('MA20')" :d="ma20Path" stroke="#1677ff" stroke-width="1" fill="none" opacity="0.8"/>
          <path v-if="activeIndicators.includes('MA60')" :d="ma60Path" stroke="#722ed1" stroke-width="1" fill="none" opacity="0.8"/>
        </svg>

        <!-- 十字光标指示 (简化) -->
        <div class="chart-legend" v-if="activeIndicators.length">
          <span v-if="activeIndicators.includes('MA5')" style="color:#f6c90e">MA5:{{ (currentStock.price * 0.98).toFixed(2) }}</span>
          <span v-if="activeIndicators.includes('MA20')" style="color:#1677ff">MA20:{{ (currentStock.price * 0.97).toFixed(2) }}</span>
          <span v-if="activeIndicators.includes('MA60')" style="color:#722ed1">MA60:{{ (currentStock.price * 0.95).toFixed(2) }}</span>
        </div>
      </div>

      <!-- 成交量图 -->
      <div class="volume-chart">
        <svg :viewBox="`0 0 ${cW} 60`" preserveAspectRatio="none" class="vol-svg">
          <rect
            v-for="(k, i) in klineData"
            :key="i"
            :x="k.x" :y="60 - k.volH"
            :width="k.w" :height="k.volH"
            :fill="k.up ? 'rgba(245,34,45,0.5)' : 'rgba(82,196,26,0.5)'"
          />
        </svg>
      </div>

      <!-- MACD / KDJ 指标区 -->
      <div v-if="activeIndicators.includes('MACD')" class="indicator-chart">
        <div class="ind-label">MACD</div>
        <svg :viewBox="`0 0 ${cW} 60`" preserveAspectRatio="none" style="width:100%;height:60px">
          <rect v-for="(k, i) in klineData" :key="i"
            :x="k.x + k.w/2 - 1" :y="k.macdUp ? 30 - k.macdH : 30"
            :width="2" :height="k.macdH"
            :fill="k.macdUp ? '#f5222d' : '#52c41a'"
            opacity="0.7"
          />
          <line x1="0" y1="30" :x2="cW" y2="30" stroke="#30363d" stroke-width="0.6"/>
        </svg>
      </div>
    </div>

    <!-- 底部：AI分析 + 指标摘要 -->
    <div class="analysis-footer">
      <div class="ai-analysis-card">
        <div class="aa-header">
          <span>🤖 AI 技术分析</span>
          <button class="btn-refresh" @click="refreshAnalysis">🔄 刷新</button>
        </div>
        <div class="aa-body">
          <div class="aa-summary">
            <div class="aa-score" :class="aiScore >= 60 ? 'up' : aiScore >= 40 ? 'neutral' : 'down'">
              <div class="score-num">{{ aiScore }}</div>
              <div class="score-label">综合评分</div>
            </div>
            <div class="aa-signals">
              <div class="signal-row" v-for="s in signals" :key="s.name">
                <span class="s-name">{{ s.name }}</span>
                <span class="s-signal" :class="s.cls">{{ s.signal }}</span>
              </div>
            </div>
          </div>
          <div class="aa-text">{{ aiAnalysisText }}</div>
        </div>
      </div>

      <div class="indicator-summary">
        <div class="is-title">指标详情</div>
        <div class="is-grid">
          <div class="is-item" v-for="item in indicatorDetails" :key="item.name">
            <div class="is-name">{{ item.name }}</div>
            <div class="is-value" :class="item.cls">{{ item.value }}</div>
            <div class="is-bar">
              <div class="is-fill" :style="{ width: item.pct + '%', background: item.color }"></div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const searchCode = ref('')
const klineType = ref('day')
const cW = 600
const cH = 300

const indicators = ['MA5', 'MA20', 'MA60', 'MACD', 'KDJ', 'RSI', 'BOLL']
const activeIndicators = ref(['MA5', 'MA20', 'MACD'])

const klineTypes = [
  { id: '1min', label: '1分' },
  { id: '5min', label: '5分' },
  { id: '30min', label: '30分' },
  { id: 'day', label: '日K' },
  { id: 'week', label: '周K' },
  { id: 'month', label: '月K' }
]

const recentStocks = ref([
  { code: '600519.SH', name: '贵州茅台', price: 1768.00, change: 1.23 },
  { code: '300750.SZ', name: '宁德时代', price: 215.30, change: 2.41 },
  { code: '002594.SZ', name: '比亚迪', price: 302.45, change: 3.12 },
  { code: '000001.SZ', name: '平安银行', price: 11.22, change: 0.54 }
])

const currentStock = ref(recentStocks.value[0])
const klineData = ref([])
const aiScore = ref(72)

const aiAnalysisText = computed(() => {
  return `${currentStock.value.name} 当前处于${aiScore.value >= 60 ? '强势' : '弱势'}格局。\n` +
    `短期均线呈多头排列，MACD 柱状线持续放大，成交量较前期有所扩张。\n` +
    `技术面来看，支撑位在 ¥${(currentStock.value.price * 0.96).toFixed(2)}，` +
    `压力位在 ¥${(currentStock.value.price * 1.05).toFixed(2)}。\n` +
    `建议：可轻仓试探，止损设置在 ¥${(currentStock.value.price * 0.94).toFixed(2)} 以下。`
})

const signals = computed(() => [
  { name: 'MACD', signal: '金叉', cls: 'up' },
  { name: 'KDJ', signal: '超买区', cls: 'warn' },
  { name: 'RSI(14)', signal: `${(50 + Math.random() * 20).toFixed(1)}`, cls: 'neutral' },
  { name: '均线', signal: '多头排列', cls: 'up' },
  { name: 'BOLL', signal: '中轨上方', cls: 'up' }
])

const indicatorDetails = computed(() => [
  { name: 'RSI(6)', value: '68.4', cls: 'warn', pct: 68, color: '#faad14' },
  { name: 'RSI(14)', value: '62.1', cls: '', pct: 62, color: '#1677ff' },
  { name: 'KDJ-K', value: '78.3', cls: 'warn', pct: 78, color: '#faad14' },
  { name: 'KDJ-D', value: '71.6', cls: '', pct: 72, color: '#722ed1' },
  { name: '量比', value: '1.54', cls: 'up', pct: 60, color: '#f5222d' },
  { name: '换手率', value: '1.23%', cls: '', pct: 30, color: '#13c2c2' }
])

const generateKlineData = () => {
  const n = klineType.value.includes('min') ? 60 : 60
  const barW = (cW / n) * 0.7
  const base = currentStock.value.price
  const arr = []
  let prev = base * 0.95
  for (let i = 0; i < n; i++) {
    const open = prev
    const close = open + (Math.random() - 0.48) * base * 0.015
    const high = Math.max(open, close) + Math.random() * base * 0.008
    const low = Math.min(open, close) - Math.random() * base * 0.008
    const minP = base * 0.88
    const maxP = base * 1.08
    const rangeP = maxP - minP || 1
    const toY = (v) => cH - ((v - minP) / rangeP) * (cH - 20) - 10
    const vol = Math.random() * 50 + 5
    arr.push({
      x: i * (cW / n),
      w: barW,
      openY: toY(open),
      closeY: toY(close),
      highY: toY(high),
      lowY: toY(low),
      up: close >= open,
      volH: vol,
      macdUp: Math.random() > 0.45,
      macdH: Math.random() * 20 + 2
    })
    prev = close
  }
  klineData.value = arr
}

const makeMaPath = (offset) => {
  if (!klineData.value.length) return ''
  return klineData.value.map((k, i) => {
    const y = (k.openY + k.closeY) / 2 + Math.sin(i * 0.3) * offset
    const x = k.x + k.w / 2
    return `${i === 0 ? 'M' : 'L'} ${x} ${y}`
  }).join(' ')
}

const ma5Path = computed(() => makeMaPath(3))
const ma20Path = computed(() => makeMaPath(8))
const ma60Path = computed(() => makeMaPath(15))

const toggleIndicator = (ind) => {
  const idx = activeIndicators.value.indexOf(ind)
  if (idx >= 0) activeIndicators.value.splice(idx, 1)
  else activeIndicators.value.push(ind)
}

const switchStock = (s) => {
  currentStock.value = s
  aiScore.value = Math.floor(Math.random() * 30 + 50)
  generateKlineData()
}

const addByCode = () => {
  if (searchCode.value.trim()) {
    const code = searchCode.value.trim().toUpperCase()
    if (!recentStocks.value.find(s => s.code === code)) {
      const ch = parseFloat((Math.random() * 6 - 3).toFixed(2))
      const newS = { code, name: code, price: parseFloat((Math.random() * 200 + 10).toFixed(2)), change: ch }
      recentStocks.value.unshift(newS)
      if (recentStocks.value.length > 6) recentStocks.value.pop()
    }
    searchCode.value = ''
  }
}

const refreshAnalysis = () => {
  aiScore.value = Math.floor(Math.random() * 30 + 50)
}

onMounted(() => {
  generateKlineData()
  if (route.query.code) {
    const found = recentStocks.value.find(s => s.code === route.query.code)
    if (found) switchStock(found)
  }
})
</script>

<style lang="scss" scoped>
$bg-dark: #0d1117;
$bg-card: #161b22;
$bg-hover: #1c2128;
$border-color: #30363d;
$text-primary: #e6edf3;
$text-secondary: #8b949e;
$primary: #1677ff;

.analysis-view {
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  padding: 12px 16px;
  gap: 10px;
}

/* ===== HEADER ===== */
.analysis-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  flex-shrink: 0;
}

.stock-selector {
  display: flex;
  gap: 6px;
  align-items: center;
  flex-wrap: wrap;
}

.recent-btn {
  padding: 4px 10px;
  border-radius: 16px;
  background: $bg-card;
  border: 1px solid $border-color;
  color: $text-secondary;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.15s;
  &:hover { border-color: $primary; color: $primary; }
  &.active { background: rgba(22,119,255,0.12); border-color: $primary; color: $primary; }
}

.code-input {
  background: $bg-card;
  border: 1px solid $border-color;
  color: $text-primary;
  padding: 4px 10px;
  border-radius: 6px;
  font-size: 12px;
  outline: none;
  width: 120px;
  &:focus { border-color: $primary; }
  &::placeholder { color: $text-secondary; }
}

.indicator-selector {
  display: flex;
  gap: 4px;
  flex-wrap: wrap;
}

.ind-btn {
  padding: 3px 8px;
  border-radius: 4px;
  background: transparent;
  border: 1px solid $border-color;
  color: $text-secondary;
  font-size: 11px;
  cursor: pointer;
  transition: all 0.15s;
  &:hover { border-color: $primary; color: $primary; }
  &.active { background: rgba(22,119,255,0.1); border-color: $primary; color: $primary; }
}

/* ===== CHART MAIN ===== */
.chart-main {
  flex: 1;
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.main-chart-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 8px 12px;
  border-bottom: 1px solid $border-color;
  background: rgba(255,255,255,0.01);
  flex-shrink: 0;
}

.mc-name {
  font-size: 15px;
  font-weight: 600;
  color: $text-primary;
  .mc-code { font-size: 11px; color: $text-secondary; margin-left: 6px; }
}

.mc-price {
  font-size: 20px;
  font-weight: 700;
  .mc-pct { font-size: 12px; margin-left: 6px; }
}

.kline-type-btns {
  display: flex;
  gap: 3px;
  margin-left: auto;
  button {
    padding: 3px 8px;
    border-radius: 4px;
    background: transparent;
    border: 1px solid $border-color;
    color: $text-secondary;
    font-size: 11px;
    cursor: pointer;
    &.active { background: $primary; border-color: $primary; color: #fff; }
  }
}

.kline-chart {
  flex: 1;
  position: relative;
  overflow: hidden;
  min-height: 0;
}

.kline-svg, .vol-svg {
  width: 100%;
  display: block;
}

.kline-svg { height: calc(100% - 10px); }

.chart-legend {
  position: absolute;
  top: 4px;
  left: 8px;
  display: flex;
  gap: 10px;
  font-size: 10px;
}

.volume-chart {
  height: 60px;
  border-top: 1px solid $border-color;
  flex-shrink: 0;
}

.indicator-chart {
  height: 70px;
  border-top: 1px solid $border-color;
  position: relative;
  flex-shrink: 0;
  .ind-label {
    position: absolute;
    top: 2px;
    left: 6px;
    font-size: 10px;
    color: $text-secondary;
  }
}

/* ===== FOOTER ===== */
.analysis-footer {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
  flex-shrink: 0;
}

.ai-analysis-card {
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
  padding: 12px;
}

.aa-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
  font-size: 12px;
  font-weight: 600;
  color: $text-secondary;
}

.btn-refresh {
  background: transparent;
  border: 1px solid $border-color;
  color: $text-secondary;
  padding: 2px 7px;
  border-radius: 4px;
  font-size: 10px;
  cursor: pointer;
  &:hover { background: $bg-hover; }
}

.aa-body { display: flex; flex-direction: column; gap: 8px; }

.aa-summary {
  display: flex;
  gap: 12px;
  align-items: flex-start;
}

.aa-score {
  text-align: center;
  padding: 8px;
  border-radius: 8px;
  min-width: 60px;
  &.up { background: rgba(245,34,45,0.08); }
  &.neutral { background: rgba(250,173,20,0.08); }
  &.down { background: rgba(82,196,26,0.08); }

  .score-num { font-size: 24px; font-weight: 700; }
  .score-label { font-size: 10px; color: $text-secondary; }
}

.aa-signals {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.signal-row {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  .s-name { color: $text-secondary; }
  .s-signal { font-weight: 500; }
}

.aa-text {
  font-size: 11px;
  color: $text-secondary;
  line-height: 1.6;
  white-space: pre-line;
}

.indicator-summary {
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
  padding: 12px;
}

.is-title {
  font-size: 12px;
  font-weight: 600;
  color: $text-secondary;
  margin-bottom: 8px;
}

.is-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}

.is-item {
  .is-name { font-size: 10px; color: $text-secondary; }
  .is-value { font-size: 13px; font-weight: 600; color: $text-primary; margin: 2px 0; }
  .is-bar {
    height: 3px;
    background: rgba(255,255,255,0.06);
    border-radius: 2px;
    overflow: hidden;
  }
  .is-fill {
    height: 100%;
    border-radius: 2px;
    transition: width 0.5s ease;
  }
}

.up { color: #f5222d !important; }
.down { color: #52c41a !important; }
.warn { color: #faad14 !important; }
.neutral { color: $text-secondary; }
</style>
