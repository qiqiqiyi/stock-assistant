<template>
  <div class="watchlist-view">
    <!-- 头部操作栏 -->
    <div class="wl-header">
      <div class="header-left">
        <h2>自选股</h2>
        <span class="badge-count">{{ stocks.length }} 只</span>
      </div>
      <div class="header-right">
        <div class="search-add">
          <input v-model="addCode" placeholder="输入股票代码添加..." @keyup.enter="addStock" />
          <button class="btn btn-primary" @click="addStock">+ 添加</button>
        </div>
        <select v-model="sortBy" class="sort-select">
          <option value="default">默认排序</option>
          <option value="change_desc">涨幅 ↓</option>
          <option value="change_asc">跌幅 ↓</option>
          <option value="amount">成交额 ↓</option>
        </select>
      </div>
    </div>

    <!-- 涨跌统计 -->
    <div class="stat-bar">
      <div class="stat-item">
        <span class="stat-label">今日上涨</span>
        <span class="stat-val up">{{ upCount }} 只</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">今日下跌</span>
        <span class="stat-val down">{{ downCount }} 只</span>
      </div>
      <div class="stat-item">
        <span class="stat-label">平均涨幅</span>
        <span class="stat-val" :class="avgChange >= 0 ? 'up' : 'down'">
          {{ avgChange >= 0 ? '+' : '' }}{{ avgChange.toFixed(2) }}%
        </span>
      </div>
      <div class="stat-item">
        <span class="stat-label">最强个股</span>
        <span class="stat-val up">{{ topStock?.name }} +{{ topStock?.change }}%</span>
      </div>
    </div>

    <!-- 股票卡片网格 -->
    <div class="wl-grid">
      <div
        v-for="s in sortedStocks"
        :key="s.code"
        class="wl-card"
        :class="{ 'card-up': s.change > 0, 'card-down': s.change < 0 }"
      >
        <!-- 卡片头部 -->
        <div class="wc-header">
          <div class="wc-name-block">
            <span class="wc-name">{{ s.name }}</span>
            <span class="wc-code">{{ s.code }}</span>
          </div>
          <div class="wc-actions">
            <button class="icon-btn" @click="viewAnalysis(s)" title="技术分析">📊</button>
            <button class="icon-btn" @click="addToPortfolio(s)" title="加入组合">💼</button>
            <button class="icon-btn danger" @click="removeStock(s.code)" title="删除">✕</button>
          </div>
        </div>

        <!-- 价格区 -->
        <div class="wc-price-block">
          <span class="wc-price" :class="s.change >= 0 ? 'up' : 'down'">{{ s.price }}</span>
          <div class="wc-change-group">
            <span class="chg-pct" :class="s.change >= 0 ? 'tag-up' : 'tag-down'">
              {{ s.change >= 0 ? '+' : '' }}{{ s.change }}%
            </span>
            <span class="chg-abs" :class="s.change >= 0 ? 'up' : 'down'">
              {{ s.change >= 0 ? '+' : '' }}{{ (s.price * s.change / 100).toFixed(2) }}
            </span>
          </div>
        </div>

        <!-- 迷你火花线 -->
        <div class="wc-sparkline">
          <svg :viewBox="`0 0 80 30`" preserveAspectRatio="none" style="width:100%;height:30px">
            <path :d="s.spark" :stroke="s.change >= 0 ? '#f5222d' : '#52c41a'" stroke-width="1.2" fill="none" />
          </svg>
        </div>

        <!-- 指标行 -->
        <div class="wc-metrics">
          <div class="wm">
            <span class="wm-l">量比</span>
            <span class="wm-v">{{ s.volRatio }}</span>
          </div>
          <div class="wm">
            <span class="wm-l">换手</span>
            <span class="wm-v">{{ s.turnover }}</span>
          </div>
          <div class="wm">
            <span class="wm-l">PE</span>
            <span class="wm-v">{{ s.pe }}x</span>
          </div>
        </div>

        <!-- AI 摘要 -->
        <div class="wc-ai-hint" v-if="s.aiHint">
          <span class="ai-icon">🤖</span>
          <span>{{ s.aiHint }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()
const addCode = ref('')
const sortBy = ref('default')

// 生成火花线
const genSpark = (change) => {
  const pts = []
  let v = 15
  for (let i = 0; i <= 20; i++) {
    v += (Math.random() - (change > 0 ? 0.45 : 0.55)) * 2
    v = Math.max(2, Math.min(28, v))
    pts.push([i * 4, v])
  }
  return pts.map((p, i) => `${i === 0 ? 'M' : 'L'} ${p[0]} ${p[1]}`).join(' ')
}

const stocks = ref([
  { code: '600519.SH', name: '贵州茅台', price: 1768.00, change: 1.23, volRatio: '1.24', turnover: '0.45%', pe: 28.6, aiHint: '近期均线多头排列，量能温和放大' },
  { code: '300750.SZ', name: '宁德时代', price: 215.30, change: 2.41, volRatio: '2.18', turnover: '1.67%', pe: 45.2, aiHint: '主力大单净流入，短期看多' },
  { code: '002594.SZ', name: '比亚迪', price: 302.45, change: 3.12, volRatio: '3.02', turnover: '2.34%', pe: 38.7, aiHint: '突破前高压力位，注意追高风险' },
  { code: '601318.SH', name: '中国平安', price: 46.23, change: -1.05, volRatio: '0.87', turnover: '0.92%', pe: 8.4, aiHint: '处于盘整区间，等待方向选择' },
  { code: '600036.SH', name: '招商银行', price: 36.78, change: -0.22, volRatio: '0.65', turnover: '0.38%', pe: 6.8, aiHint: '股息率 4.2%，防御性资产' },
  { code: '000001.SZ', name: '平安银行', price: 11.22, change: 0.54, volRatio: '1.12', turnover: '0.56%', pe: 5.6, aiHint: '低估值修复逻辑，基本面稳健' },
  { code: '688111.SH', name: '金山办公', price: 64.70, change: 0.93, volRatio: '1.45', turnover: '0.87%', pe: 52.3, aiHint: '国产替代逻辑持续，AI赋能可期' },
  { code: '000858.SZ', name: '五粮液', price: 148.50, change: -0.87, volRatio: '0.78', turnover: '0.63%', pe: 21.4, aiHint: '高端白酒消费有所降速，注意仓位' },
  { code: '600900.SH', name: '长江电力', price: 27.45, change: 0.33, volRatio: '0.92', turnover: '0.21%', pe: 18.2, aiHint: '高股息防御性标的，适合配置' },
  { code: '601888.SH', name: '中国中免', price: 98.30, change: 1.88, volRatio: '1.56', turnover: '0.78%', pe: 32.5, aiHint: '海南自贸港政策催化，中长期向好' },
  { code: '300059.SZ', name: '东方财富', price: 18.45, change: -0.54, volRatio: '1.23', turnover: '1.45%', pe: 24.7, aiHint: '券商行情联动，市场活跃度回升受益' },
  { code: '002415.SZ', name: '海康威视', price: 28.90, change: 1.62, volRatio: '1.34', turnover: '0.89%', pe: 19.3, aiHint: 'AI视觉赛道，估值处于历史低位' }
].map(s => ({ ...s, spark: genSpark(s.change) })))

const upCount = computed(() => stocks.value.filter(s => s.change > 0).length)
const downCount = computed(() => stocks.value.filter(s => s.change < 0).length)
const avgChange = computed(() => {
  const sum = stocks.value.reduce((a, s) => a + s.change, 0)
  return sum / stocks.value.length
})
const topStock = computed(() => [...stocks.value].sort((a, b) => b.change - a.change)[0])

const sortedStocks = computed(() => {
  const arr = [...stocks.value]
  if (sortBy.value === 'change_desc') return arr.sort((a, b) => b.change - a.change)
  if (sortBy.value === 'change_asc') return arr.sort((a, b) => a.change - b.change)
  if (sortBy.value === 'amount') return arr.sort((a, b) => parseFloat(b.pe) - parseFloat(a.pe))
  return arr
})

const addStock = () => {
  if (!addCode.value.trim()) return
  // 模拟添加
  const code = addCode.value.toUpperCase()
  if (!stocks.value.find(s => s.code === code)) {
    const change = parseFloat((Math.random() * 6 - 3).toFixed(2))
    stocks.value.push({
      code,
      name: '未知股票',
      price: parseFloat((Math.random() * 200 + 10).toFixed(2)),
      change,
      volRatio: (Math.random() + 0.5).toFixed(2),
      turnover: (Math.random() * 2).toFixed(2) + '%',
      pe: (Math.random() * 40 + 5).toFixed(1),
      aiHint: null,
      spark: genSpark(change)
    })
  }
  addCode.value = ''
}

const removeStock = (code) => {
  stocks.value = stocks.value.filter(s => s.code !== code)
}

const viewAnalysis = (s) => {
  router.push({ path: '/analysis', query: { code: s.code } })
}

const addToPortfolio = (s) => {
  router.push({ path: '/portfolio', query: { add: s.code } })
}
</script>

<style lang="scss" scoped>
$bg-dark: #0d1117;
$bg-card: #161b22;
$bg-hover: #1c2128;
$border-color: #30363d;
$text-primary: #e6edf3;
$text-secondary: #8b949e;
$primary: #1677ff;

.watchlist-view {
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  padding: 16px;
  gap: 12px;
}

/* ===== HEADER ===== */
.wl-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-shrink: 0;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 8px;
  h2 { font-size: 18px; color: $text-primary; }
}

.badge-count {
  background: rgba(22, 119, 255, 0.12);
  color: $primary;
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 10px;
  font-weight: 600;
}

.header-right {
  display: flex;
  gap: 10px;
  align-items: center;
}

.search-add {
  display: flex;
  gap: 6px;
  align-items: center;

  input {
    background: $bg-card;
    border: 1px solid $border-color;
    color: $text-primary;
    padding: 6px 12px;
    border-radius: 6px;
    font-size: 12px;
    outline: none;
    width: 180px;
    &:focus { border-color: $primary; }
    &::placeholder { color: $text-secondary; }
  }
}

.sort-select {
  background: $bg-card;
  border: 1px solid $border-color;
  color: $text-primary;
  padding: 5px 10px;
  border-radius: 6px;
  font-size: 12px;
  outline: none;
  cursor: pointer;
  &:focus { border-color: $primary; }
}

/* ===== STAT BAR ===== */
.stat-bar {
  display: flex;
  gap: 12px;
  padding: 10px 14px;
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
  flex-shrink: 0;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 8px;
  .stat-label { font-size: 12px; color: $text-secondary; }
  .stat-val { font-size: 13px; font-weight: 600; }
}

/* ===== GRID ===== */
.wl-grid {
  flex: 1;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 10px;
  overflow-y: auto;
  align-content: start;
}

/* ===== CARD ===== */
.wl-card {
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 10px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  transition: all 0.2s;

  &:hover {
    border-color: #484f58;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  }

  &.card-up { border-top: 2px solid rgba(245, 34, 45, 0.4); }
  &.card-down { border-top: 2px solid rgba(82, 196, 26, 0.4); }
}

.wc-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
}

.wc-name-block {
  .wc-name { display: block; font-size: 13px; font-weight: 600; color: $text-primary; }
  .wc-code { font-size: 10px; color: $text-secondary; }
}

.wc-actions {
  display: flex;
  gap: 2px;
  opacity: 0;
  transition: opacity 0.2s;
}

.wl-card:hover .wc-actions { opacity: 1; }

.icon-btn {
  background: transparent;
  border: none;
  cursor: pointer;
  font-size: 12px;
  padding: 2px 4px;
  border-radius: 3px;
  transition: background 0.15s;
  &:hover { background: $bg-hover; }
  &.danger:hover { background: rgba(255, 77, 79, 0.1); }
}

.wc-price-block {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
}

.wc-price {
  font-size: 20px;
  font-weight: 700;
}

.wc-change-group {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 1px;
}

.chg-pct {
  font-size: 11px;
  font-weight: 600;
  padding: 1px 5px;
  border-radius: 3px;
  &.tag-up { background: rgba(245,34,45,0.1); color: #f5222d; }
  &.tag-down { background: rgba(82,196,26,0.1); color: #52c41a; }
}

.chg-abs { font-size: 10px; }

.wc-sparkline {
  height: 30px;
  background: rgba(255,255,255,0.02);
  border-radius: 4px;
  overflow: hidden;
}

.wc-metrics {
  display: flex;
  justify-content: space-between;
}

.wm {
  display: flex;
  flex-direction: column;
  gap: 1px;
  .wm-l { font-size: 9px; color: $text-secondary; }
  .wm-v { font-size: 11px; color: $text-primary; font-weight: 500; }
}

.wc-ai-hint {
  display: flex;
  align-items: flex-start;
  gap: 4px;
  padding: 5px 7px;
  background: rgba(22, 119, 255, 0.05);
  border: 1px solid rgba(22, 119, 255, 0.15);
  border-radius: 5px;
  font-size: 10px;
  color: $text-secondary;
  line-height: 1.4;

  .ai-icon { flex-shrink: 0; font-size: 10px; }
}

.up { color: #f5222d !important; }
.down { color: #52c41a !important; }

.btn {
  display: inline-flex;
  align-items: center;
  padding: 6px 12px;
  border-radius: 6px;
  border: none;
  cursor: pointer;
  font-size: 12px;
  font-weight: 500;
  transition: all 0.2s;

  &.btn-primary {
    background: $primary;
    color: #fff;
    &:hover { background: #0958d9; }
  }
}
</style>
