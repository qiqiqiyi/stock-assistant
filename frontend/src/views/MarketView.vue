<template>
  <div class="market-view">
    <!-- 顶部市场概览 -->
    <div class="overview-bar">
      <div
        v-for="idx in indices"
        :key="idx.code"
        class="idx-card"
        :class="{ selected: selectedIndex === idx.code }"
        @click="selectedIndex = idx.code"
      >
        <div class="idx-name">{{ idx.name }}</div>
        <div class="idx-price" :class="idx.change >= 0 ? 'up' : 'down'">{{ idx.price }}</div>
        <div class="idx-info">
          <span :class="idx.change >= 0 ? 'up' : 'down'">
            {{ idx.change >= 0 ? '+' : '' }}{{ idx.change }}%
          </span>
          <span class="idx-amplitude">振幅 {{ idx.amplitude }}%</span>
        </div>
      </div>

      <!-- 市场资金 -->
      <div class="fund-flow-mini">
        <div class="ff-title">今日资金流向</div>
        <div class="ff-item">
          <span>北向资金</span>
          <span class="up">+45.3亿</span>
        </div>
        <div class="ff-item">
          <span>南向资金</span>
          <span class="down">-12.1亿</span>
        </div>
      </div>
    </div>

    <!-- 主体内容 -->
    <div class="market-body">
      <!-- 左侧股票列表 -->
      <div class="stock-list-panel">
        <!-- Tab切换 -->
        <div class="list-tabs">
          <button
            v-for="tab in listTabs"
            :key="tab.id"
            class="list-tab"
            :class="{ active: activeTab === tab.id }"
            @click="activeTab = tab.id"
          >{{ tab.label }}</button>
        </div>

        <!-- 表头 -->
        <div class="table-header">
          <span class="th-name">名称/代码</span>
          <span class="th-price">最新价</span>
          <span class="th-chg">涨跌幅</span>
          <span class="th-vol">成交额</span>
        </div>

        <!-- 列表 -->
        <div class="stock-table">
          <div
            v-for="s in filteredStocks"
            :key="s.code"
            class="stock-row"
            :class="{ selected: selectedStock?.code === s.code }"
            @click="selectStock(s)"
          >
            <div class="td-name">
              <div class="sn">{{ s.name }}</div>
              <div class="sc">{{ s.code }}</div>
            </div>
            <div class="td-price" :class="s.change >= 0 ? 'up' : 'down'">
              {{ s.price }}
            </div>
            <div class="td-chg">
              <span class="chg-badge" :class="s.change >= 0 ? 'badge-up' : 'badge-down'">
                {{ s.change >= 0 ? '+' : '' }}{{ s.change }}%
              </span>
            </div>
            <div class="td-vol text-secondary">{{ s.amount }}</div>
          </div>
        </div>
      </div>

      <!-- 右侧图表区 -->
      <div class="chart-panel" v-if="selectedStock">
        <!-- 股票信息头部 -->
        <div class="chart-header">
          <div class="stock-info-main">
            <h3>{{ selectedStock.name }}</h3>
            <span class="code-badge">{{ selectedStock.code }}</span>
          </div>
          <div class="price-main">
            <span class="big-price" :class="selectedStock.change >= 0 ? 'up' : 'down'">
              {{ selectedStock.price }}
            </span>
            <div class="price-sub">
              <span :class="selectedStock.change >= 0 ? 'up' : 'down'">
                {{ selectedStock.change >= 0 ? '+' : '' }}{{ selectedStock.change }}%
              </span>
              <span class="text-secondary">
                {{ selectedStock.change >= 0 ? '+' : '' }}{{ (selectedStock.price * selectedStock.change / 100).toFixed(2) }}
              </span>
            </div>
          </div>
          <div class="chart-period-tabs">
            <button
              v-for="p in periods"
              :key="p"
              :class="{ active: chartPeriod === p }"
              @click="chartPeriod = p; generateChartData()"
            >{{ p }}</button>
          </div>
        </div>

        <!-- K线图区域 -->
        <div class="chart-area">
          <svg class="mini-chart" :viewBox="`0 0 ${svgW} ${svgH}`" preserveAspectRatio="none">
            <!-- 网格线 -->
            <line
              v-for="y in gridLines"
              :key="y"
              :x1="0" :y1="y" :x2="svgW" :y2="y"
              stroke="#30363d" stroke-width="0.5"
            />
            <!-- 面积填充 -->
            <defs>
              <linearGradient id="areaGrad" x1="0" y1="0" x2="0" y2="1">
                <stop offset="0%" :stop-color="selectedStock.change >= 0 ? '#f5222d' : '#52c41a'" stop-opacity="0.2"/>
                <stop offset="100%" :stop-color="selectedStock.change >= 0 ? '#f5222d' : '#52c41a'" stop-opacity="0"/>
              </linearGradient>
            </defs>
            <path :d="areaPath" fill="url(#areaGrad)" />
            <!-- 折线 -->
            <path :d="linePath" :stroke="selectedStock.change >= 0 ? '#f5222d' : '#52c41a'" stroke-width="1.5" fill="none" />
          </svg>
        </div>

        <!-- 指标行 -->
        <div class="detail-metrics">
          <div class="dm-item" v-for="m in detailMetrics" :key="m.label">
            <span class="dm-label">{{ m.label }}</span>
            <span class="dm-value" :class="m.color">{{ m.value }}</span>
          </div>
        </div>

        <!-- 涨跌分析 -->
        <div class="analysis-cards">
          <div class="ac-card">
            <div class="ac-title">📊 技术面</div>
            <div class="ac-items">
              <div class="ac-item" v-for="t in techItems" :key="t.label">
                <span>{{ t.label }}</span>
                <span :class="t.cls">{{ t.value }}</span>
              </div>
            </div>
          </div>
          <div class="ac-card">
            <div class="ac-title">🏢 基本面</div>
            <div class="ac-items">
              <div class="ac-item" v-for="f in fundItems" :key="f.label">
                <span>{{ f.label }}</span>
                <span :class="f.cls">{{ f.value }}</span>
              </div>
            </div>
          </div>
          <div class="ac-card">
            <div class="ac-title">💰 资金面</div>
            <div class="ac-items">
              <div class="ac-item" v-for="c in capitalItems" :key="c.label">
                <span>{{ c.label }}</span>
                <span :class="c.cls">{{ c.value }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 未选中股票时的提示 -->
      <div class="chart-panel empty-chart" v-else>
        <div class="empty-hint">
          <span style="font-size:40px">📊</span>
          <p>点击左侧股票查看详细行情</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const selectedIndex = ref('sh')
const activeTab = ref('hot')
const chartPeriod = ref('1个月')
const selectedStock = ref(null)
const svgW = 600
const svgH = 150

const periods = ['1周', '1个月', '3个月', '6个月', '1年']

const listTabs = [
  { id: 'hot', label: '🔥 热门' },
  { id: 'up', label: '涨幅榜' },
  { id: 'down', label: '跌幅榜' },
  { id: 'vol', label: '成交量' }
]

const indices = ref([
  { code: 'sh', name: '上证指数', price: '3287.45', change: 0.82, amplitude: 1.24 },
  { code: 'sz', name: '深证成指', price: '10654.23', change: 1.15, amplitude: 1.67 },
  { code: 'cy', name: '创业板指', price: '2143.67', change: -0.34, amplitude: 0.98 },
  { code: 'ke', name: '科创50', price: '986.32', change: 1.56, amplitude: 2.13 },
  { code: 'hs300', name: '沪深300', price: '3924.18', change: 0.67, amplitude: 1.05 }
])

const allStocks = [
  { code: '600519.SH', name: '贵州茅台', price: 1768.00, change: 1.23, amount: '41.3亿', sector: '食品饮料', pe: 28.6, pb: 8.2, roe: 32.4 },
  { code: '300750.SZ', name: '宁德时代', price: 215.30, change: 2.41, amount: '89.7亿', sector: '电力设备', pe: 45.2, pb: 6.8, roe: 18.3 },
  { code: '002594.SZ', name: '比亚迪', price: 302.45, change: 3.12, amount: '76.5亿', sector: '汽车', pe: 38.7, pb: 5.6, roe: 16.2 },
  { code: '601318.SH', name: '中国平安', price: 46.23, change: -1.05, amount: '35.2亿', sector: '非银金融', pe: 8.4, pb: 1.2, roe: 14.5 },
  { code: '600036.SH', name: '招商银行', price: 36.78, change: -0.22, amount: '28.9亿', sector: '银行', pe: 6.8, pb: 1.1, roe: 16.8 },
  { code: '000001.SZ', name: '平安银行', price: 11.22, change: 0.54, amount: '12.4亿', sector: '银行', pe: 5.6, pb: 0.7, roe: 12.3 },
  { code: '688111.SH', name: '金山办公', price: 64.70, change: 0.93, amount: '8.2亿', sector: '计算机', pe: 52.3, pb: 8.9, roe: 22.1 },
  { code: '000858.SZ', name: '五粮液', price: 148.50, change: -0.87, amount: '22.7亿', sector: '食品饮料', pe: 21.4, pb: 6.1, roe: 28.9 },
  { code: '600900.SH', name: '长江电力', price: 27.45, change: 0.33, amount: '18.6亿', sector: '公用事业', pe: 18.2, pb: 3.4, roe: 19.5 },
  { code: '601888.SH', name: '中国中免', price: 98.30, change: 1.88, amount: '15.3亿', sector: '商贸零售', pe: 32.5, pb: 4.7, roe: 20.1 }
]

const filteredStocks = computed(() => {
  if (activeTab.value === 'up') return [...allStocks].sort((a, b) => b.change - a.change)
  if (activeTab.value === 'down') return [...allStocks].sort((a, b) => a.change - b.change)
  if (activeTab.value === 'vol') return [...allStocks].sort((a, b) => parseFloat(b.amount) - parseFloat(a.amount))
  return allStocks
})

// 图表数据
const chartPoints = ref([])

const generateChartData = () => {
  const n = { '1周': 7, '1个月': 30, '3个月': 90, '6个月': 180, '1年': 250 }[chartPeriod.value] || 30
  const base = selectedStock.value?.price || 100
  const arr = []
  let v = base * 0.95
  for (let i = 0; i < n; i++) {
    v = v + (Math.random() - 0.48) * base * 0.012
    arr.push(v)
  }
  chartPoints.value = arr
}

const gridLines = [30, 60, 90, 120]

const linePath = computed(() => {
  if (!chartPoints.value.length) return ''
  const pts = chartPoints.value
  const minV = Math.min(...pts)
  const maxV = Math.max(...pts)
  const rangeV = maxV - minV || 1
  const xStep = svgW / (pts.length - 1)
  return pts.map((v, i) => {
    const x = i * xStep
    const y = svgH - ((v - minV) / rangeV) * (svgH - 10) - 5
    return `${i === 0 ? 'M' : 'L'} ${x} ${y}`
  }).join(' ')
})

const areaPath = computed(() => {
  if (!chartPoints.value.length) return ''
  return linePath.value + ` L ${svgW} ${svgH} L 0 ${svgH} Z`
})

const detailMetrics = computed(() => {
  if (!selectedStock.value) return []
  const s = selectedStock.value
  return [
    { label: '今开', value: (s.price * 0.99).toFixed(2), color: '' },
    { label: '昨收', value: (s.price * (1 - s.change / 100)).toFixed(2), color: '' },
    { label: '最高', value: (s.price * 1.015).toFixed(2), color: 'up' },
    { label: '最低', value: (s.price * 0.985).toFixed(2), color: 'down' },
    { label: '成交额', value: s.amount, color: '' },
    { label: '换手率', value: `${(Math.random() * 3 + 0.5).toFixed(2)}%`, color: '' },
    { label: '市盈率', value: `${s.pe}x`, color: '' },
    { label: '市净率', value: `${s.pb}x`, color: '' }
  ]
})

const techItems = computed(() => [
  { label: 'MACD', value: '金叉', cls: 'up' },
  { label: 'KDJ', value: '超买', cls: 'warning' },
  { label: '均线', value: '多头排列', cls: 'up' },
  { label: 'RSI(14)', value: '62.4', cls: '' }
])

const fundItems = computed(() => {
  if (!selectedStock.value) return []
  return [
    { label: 'ROE', value: `${selectedStock.value.roe}%`, cls: 'up' },
    { label: 'PE(TTM)', value: `${selectedStock.value.pe}x`, cls: '' },
    { label: 'PB', value: `${selectedStock.value.pb}x`, cls: '' },
    { label: '营收增速', value: '+18.3%', cls: 'up' }
  ]
})

const capitalItems = computed(() => [
  { label: '主力净流入', value: '+2.34亿', cls: 'up' },
  { label: '超大单', value: '+1.12亿', cls: 'up' },
  { label: '北向资金', value: '+0.45亿', cls: 'up' },
  { label: '融资余额', value: '12.4亿', cls: '' }
])

const selectStock = (s) => {
  selectedStock.value = s
  generateChartData()
}

onMounted(() => {
  selectStock(allStocks[0])
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

.market-view {
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

/* ===== OVERVIEW BAR ===== */
.overview-bar {
  display: flex;
  gap: 8px;
  padding: 10px 16px;
  border-bottom: 1px solid $border-color;
  background: $bg-card;
  flex-shrink: 0;
  overflow-x: auto;
}

.idx-card {
  min-width: 120px;
  padding: 8px 12px;
  border-radius: 8px;
  border: 1px solid $border-color;
  cursor: pointer;
  transition: all 0.2s;

  &:hover { border-color: $primary; }
  &.selected {
    border-color: $primary;
    background: rgba(22, 119, 255, 0.06);
  }

  .idx-name { font-size: 11px; color: $text-secondary; margin-bottom: 2px; }
  .idx-price { font-size: 15px; font-weight: 700; }
  .idx-info { display: flex; gap: 6px; font-size: 10px; }
  .idx-amplitude { color: $text-secondary; }
}

.fund-flow-mini {
  margin-left: auto;
  padding: 8px 12px;
  border-radius: 8px;
  border: 1px solid $border-color;
  min-width: 130px;

  .ff-title { font-size: 10px; color: $text-secondary; margin-bottom: 4px; }
  .ff-item {
    display: flex;
    justify-content: space-between;
    font-size: 11px;
    color: $text-secondary;
    margin-bottom: 2px;
  }
}

/* ===== BODY ===== */
.market-body {
  flex: 1;
  display: flex;
  overflow: hidden;
}

/* ===== LEFT LIST ===== */
.stock-list-panel {
  width: 300px;
  min-width: 300px;
  border-right: 1px solid $border-color;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.list-tabs {
  display: flex;
  gap: 0;
  border-bottom: 1px solid $border-color;
  padding: 0 8px;
}

.list-tab {
  padding: 8px 12px;
  background: transparent;
  border: none;
  color: $text-secondary;
  font-size: 12px;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: all 0.2s;

  &:hover { color: $text-primary; }
  &.active {
    color: $primary;
    border-bottom-color: $primary;
  }
}

.table-header {
  display: grid;
  grid-template-columns: 1fr 70px 70px 60px;
  padding: 6px 10px;
  font-size: 10px;
  color: $text-secondary;
  border-bottom: 1px solid $border-color;
  background: $bg-card;
}

.stock-table {
  flex: 1;
  overflow-y: auto;
}

.stock-row {
  display: grid;
  grid-template-columns: 1fr 70px 70px 60px;
  padding: 8px 10px;
  cursor: pointer;
  transition: background 0.15s;
  border-bottom: 1px solid rgba(48, 54, 61, 0.4);

  &:hover { background: $bg-hover; }
  &.selected { background: rgba(22, 119, 255, 0.06); }
}

.td-name {
  .sn { font-size: 12px; color: $text-primary; font-weight: 500; }
  .sc { font-size: 10px; color: $text-secondary; }
}

.td-price {
  font-size: 13px;
  font-weight: 600;
  text-align: right;
}

.td-chg {
  display: flex;
  justify-content: flex-end;
  align-items: center;
}

.chg-badge {
  font-size: 11px;
  font-weight: 600;
  padding: 1px 5px;
  border-radius: 3px;
  &.badge-up { background: rgba(245,34,45,0.1); color: #f5222d; }
  &.badge-down { background: rgba(82,196,26,0.1); color: #52c41a; }
}

.td-vol { font-size: 11px; color: $text-secondary; text-align: right; }

/* ===== CHART PANEL ===== */
.chart-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  padding: 16px;
  gap: 12px;

  &.empty-chart {
    align-items: center;
    justify-content: center;
    .empty-hint {
      text-align: center;
      color: $text-secondary;
      p { margin-top: 8px; }
    }
  }
}

.chart-header {
  display: flex;
  align-items: flex-start;
  gap: 16px;
  flex-wrap: wrap;
}

.stock-info-main {
  display: flex;
  align-items: center;
  gap: 8px;
  h3 { font-size: 18px; color: $text-primary; }
  .code-badge {
    font-size: 11px;
    color: $text-secondary;
    background: $bg-hover;
    padding: 2px 6px;
    border-radius: 4px;
  }
}

.price-main {
  .big-price { font-size: 28px; font-weight: 700; }
  .price-sub { display: flex; gap: 8px; font-size: 12px; margin-top: 2px; }
}

.chart-period-tabs {
  display: flex;
  gap: 4px;
  margin-left: auto;

  button {
    background: transparent;
    border: 1px solid $border-color;
    color: $text-secondary;
    padding: 3px 10px;
    border-radius: 4px;
    font-size: 11px;
    cursor: pointer;
    transition: all 0.15s;

    &:hover { border-color: $primary; color: $primary; }
    &.active {
      background: $primary;
      border-color: $primary;
      color: #fff;
    }
  }
}

.chart-area {
  height: 180px;
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
  overflow: hidden;
}

.mini-chart {
  width: 100%;
  height: 100%;
}

.detail-metrics {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
  padding: 10px;
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
}

.dm-item {
  display: flex;
  flex-direction: column;
  gap: 2px;
  .dm-label { font-size: 10px; color: $text-secondary; }
  .dm-value { font-size: 13px; font-weight: 500; color: $text-primary; }
}

.analysis-cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
}

.ac-card {
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
  padding: 10px 12px;

  .ac-title {
    font-size: 11px;
    font-weight: 600;
    color: $text-secondary;
    margin-bottom: 8px;
  }

  .ac-items { display: flex; flex-direction: column; gap: 5px; }
  .ac-item {
    display: flex;
    justify-content: space-between;
    font-size: 11px;
    span:first-child { color: $text-secondary; }
    span:last-child { font-weight: 500; }
  }
}

.up { color: #f5222d !important; }
.down { color: #52c41a !important; }
.warning { color: #faad14 !important; }
.text-secondary { color: $text-secondary; }
</style>
