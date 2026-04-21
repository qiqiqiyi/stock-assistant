<template>
  <div class="portfolio-view">
    <!-- 资产概览卡片 -->
    <div class="overview-cards">
      <div class="ov-card main-card">
        <div class="ov-label">总资产</div>
        <div class="ov-value">¥{{ totalAsset.toLocaleString('zh-CN', { minimumFractionDigits: 2 }) }}</div>
        <div class="ov-sub">
          <span class="ov-change" :class="totalReturn >= 0 ? 'up' : 'down'">
            {{ totalReturn >= 0 ? '+' : '' }}¥{{ Math.abs(totalReturn).toFixed(2) }}
          </span>
          <span class="ov-pct" :class="totalReturnPct >= 0 ? 'up' : 'down'">
            ({{ totalReturnPct >= 0 ? '+' : '' }}{{ totalReturnPct.toFixed(2) }}%)
          </span>
        </div>
      </div>
      <div class="ov-card">
        <div class="ov-label">今日盈亏</div>
        <div class="ov-value" :class="todayPnl >= 0 ? 'up' : 'down'">
          {{ todayPnl >= 0 ? '+' : '' }}¥{{ todayPnl.toFixed(2) }}
        </div>
        <div class="ov-sub text-secondary">{{ todayPnlPct >= 0 ? '+' : '' }}{{ todayPnlPct.toFixed(2) }}%</div>
      </div>
      <div class="ov-card">
        <div class="ov-label">持仓市值</div>
        <div class="ov-value">¥{{ holdingValue.toLocaleString('zh-CN', { minimumFractionDigits: 2 }) }}</div>
        <div class="ov-sub text-secondary">{{ holdings.length }} 只持仓</div>
      </div>
      <div class="ov-card">
        <div class="ov-label">可用资金</div>
        <div class="ov-value">¥{{ availableCash.toLocaleString('zh-CN', { minimumFractionDigits: 2 }) }}</div>
        <div class="ov-sub text-secondary">仓位 {{ positionRatio.toFixed(1) }}%</div>
      </div>
    </div>

    <!-- 主体区域 -->
    <div class="pf-body">
      <!-- 左侧：持仓列表 -->
      <div class="holdings-panel">
        <div class="panel-header">
          <span class="panel-title">持仓明细</span>
          <button class="btn btn-ghost sm" @click="showAddModal = true">+ 新增持仓</button>
        </div>

        <!-- 持仓表格 -->
        <div class="holdings-table">
          <div class="ht-header">
            <span>股票</span>
            <span>持仓量</span>
            <span>成本</span>
            <span>现价</span>
            <span>盈亏</span>
            <span>操作</span>
          </div>
          <div
            v-for="h in holdings"
            :key="h.code"
            class="ht-row"
          >
            <div class="ht-name">
              <div class="htn">{{ h.name }}</div>
              <div class="htc">{{ h.code }}</div>
            </div>
            <div class="ht-cell">{{ h.shares.toLocaleString() }}</div>
            <div class="ht-cell">¥{{ h.cost }}</div>
            <div class="ht-cell" :class="h.price >= h.cost ? 'up' : 'down'">¥{{ h.price }}</div>
            <div class="ht-cell">
              <div :class="h.pnl >= 0 ? 'up' : 'down'">
                {{ h.pnl >= 0 ? '+' : '' }}¥{{ h.pnl.toFixed(0) }}
              </div>
              <div class="small-pct" :class="h.pnlPct >= 0 ? 'up' : 'down'">
                {{ h.pnlPct >= 0 ? '+' : '' }}{{ h.pnlPct.toFixed(2) }}%
              </div>
            </div>
            <div class="ht-actions">
              <button class="act-btn up-bg" @click="trade(h, 'buy')">买</button>
              <button class="act-btn down-bg" @click="trade(h, 'sell')">卖</button>
            </div>
          </div>
        </div>
      </div>

      <!-- 右侧：图表区 -->
      <div class="charts-panel">
        <!-- 仓位分布饼图 (SVG) -->
        <div class="pf-card">
          <div class="pc-title">仓位分布</div>
          <div class="pie-container">
            <svg viewBox="0 0 120 120" class="pie-svg">
              <g transform="translate(60,60)">
                <path
                  v-for="(seg, i) in pieSegments"
                  :key="i"
                  :d="seg.path"
                  :fill="seg.color"
                  opacity="0.85"
                  stroke="#0d1117"
                  stroke-width="1"
                />
                <!-- 中心文字 -->
                <text x="0" y="-4" text-anchor="middle" fill="#e6edf3" font-size="9" font-weight="600">持仓</text>
                <text x="0" y="8" text-anchor="middle" fill="#8b949e" font-size="7">{{ positionRatio.toFixed(1) }}%</text>
              </g>
            </svg>
            <div class="pie-legend">
              <div
                v-for="(seg, i) in pieSegments"
                :key="i"
                class="legend-item"
              >
                <span class="legend-dot" :style="{ background: seg.color }"></span>
                <span class="legend-name">{{ seg.name }}</span>
                <span class="legend-pct">{{ seg.pct.toFixed(1) }}%</span>
              </div>
            </div>
          </div>
        </div>

        <!-- 收益曲线 -->
        <div class="pf-card">
          <div class="pc-title">
            <span>历史收益</span>
            <div class="period-btns">
              <button
                v-for="p in ['1M', '3M', '6M', '1Y']"
                :key="p"
                :class="{ active: retPeriod === p }"
                @click="retPeriod = p"
              >{{ p }}</button>
            </div>
          </div>
          <div class="ret-chart">
            <svg viewBox="0 0 300 100" preserveAspectRatio="none" style="width:100%;height:100px">
              <defs>
                <linearGradient id="retGrad" x1="0" y1="0" x2="0" y2="1">
                  <stop offset="0%" stop-color="#1677ff" stop-opacity="0.3"/>
                  <stop offset="100%" stop-color="#1677ff" stop-opacity="0"/>
                </linearGradient>
              </defs>
              <path :d="retAreaPath" fill="url(#retGrad)" />
              <path :d="retLinePath" stroke="#1677ff" stroke-width="1.5" fill="none" />
              <!-- 基准线 -->
              <line x1="0" :y1="baseLineY" x2="300" :y2="baseLineY" stroke="#30363d" stroke-width="0.8" stroke-dasharray="4,3" />
            </svg>
          </div>
          <div class="ret-stats">
            <div class="rs-item">
              <span class="rs-label">累计收益</span>
              <span class="up rs-val">+18.43%</span>
            </div>
            <div class="rs-item">
              <span class="rs-label">最大回撤</span>
              <span class="down rs-val">-8.24%</span>
            </div>
            <div class="rs-item">
              <span class="rs-label">夏普比率</span>
              <span class="rs-val">1.42</span>
            </div>
          </div>
        </div>

        <!-- 风险提示 -->
        <div class="pf-card risk-card">
          <div class="pc-title">⚠️ 风险提示</div>
          <div class="risk-items">
            <div class="risk-item warn" v-for="r in risks" :key="r">{{ r }}</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const showAddModal = ref(false)
const retPeriod = ref('3M')

const availableCash = ref(52340.50)
const holdings = ref([
  { code: '600519.SH', name: '贵州茅台', shares: 10, cost: 1680.00, price: 1768.00 },
  { code: '300750.SZ', name: '宁德时代', shares: 100, cost: 220.00, price: 215.30 },
  { code: '002594.SZ', name: '比亚迪', shares: 50, cost: 280.00, price: 302.45 },
  { code: '601318.SH', name: '中国平安', shares: 200, cost: 50.00, price: 46.23 },
  { code: '600900.SH', name: '长江电力', shares: 300, cost: 25.00, price: 27.45 }
].map(h => ({
  ...h,
  pnl: (h.price - h.cost) * h.shares,
  pnlPct: (h.price - h.cost) / h.cost * 100
})))

const holdingValue = computed(() => holdings.value.reduce((s, h) => s + h.price * h.shares, 0))
const totalAsset = computed(() => holdingValue.value + availableCash.value)
const totalCost = computed(() => holdings.value.reduce((s, h) => s + h.cost * h.shares, 0))
const totalReturn = computed(() => holdingValue.value - totalCost.value)
const totalReturnPct = computed(() => totalReturn.value / totalCost.value * 100)
const todayPnl = computed(() => holdings.value.reduce((s, h) => s + h.pnl * 0.1, 0))
const todayPnlPct = computed(() => todayPnl.value / holdingValue.value * 100)
const positionRatio = computed(() => holdingValue.value / totalAsset.value * 100)

// 饼图
const pieColors = ['#1677ff', '#f5222d', '#52c41a', '#faad14', '#722ed1', '#13c2c2']
const pieSegments = computed(() => {
  const total = holdingValue.value
  let startAngle = -Math.PI / 2
  return holdings.value.map((h, i) => {
    const pct = (h.price * h.shares) / total * 100
    const angle = (pct / 100) * 2 * Math.PI
    const endAngle = startAngle + angle
    const r = 45
    const x1 = Math.cos(startAngle) * r
    const y1 = Math.sin(startAngle) * r
    const x2 = Math.cos(endAngle) * r
    const y2 = Math.sin(endAngle) * r
    const largeArc = angle > Math.PI ? 1 : 0
    const path = `M 0 0 L ${x1} ${y1} A ${r} ${r} 0 ${largeArc} 1 ${x2} ${y2} Z`
    startAngle = endAngle
    return { path, color: pieColors[i % pieColors.length], name: h.name, pct }
  })
})

// 收益曲线
const retPoints = computed(() => {
  const n = { '1M': 30, '3M': 90, '6M': 180, '1Y': 250 }[retPeriod.value] || 90
  const pts = []
  let v = 50
  for (let i = 0; i < n; i++) {
    v += (Math.random() - 0.46) * 2
    v = Math.max(5, Math.min(95, v))
    pts.push(v)
  }
  return pts
})

const baseLineY = computed(() => {
  if (!retPoints.value.length) return 50
  return retPoints.value[0]
})

const retLinePath = computed(() => {
  const pts = retPoints.value
  if (!pts.length) return ''
  const xStep = 300 / (pts.length - 1)
  return pts.map((v, i) => `${i === 0 ? 'M' : 'L'} ${i * xStep} ${v}`).join(' ')
})

const retAreaPath = computed(() => {
  return retLinePath.value + ` L 300 100 L 0 100 Z`
})

const risks = [
  '当前单只股票最高仓位超过 30%，建议分散',
  '科技类持仓集中，行业风险较高',
  '建议留有 20%+ 现金应对市场波动'
]

const trade = (h, type) => {
  alert(`${type === 'buy' ? '买入' : '卖出'} ${h.name}`)
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

.portfolio-view {
  height: 100%;
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 16px;
  overflow: hidden;
}

/* ===== OVERVIEW CARDS ===== */
.overview-cards {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr 1fr;
  gap: 10px;
  flex-shrink: 0;
}

.ov-card {
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 10px;
  padding: 14px 16px;

  &.main-card {
    background: linear-gradient(135deg, rgba(22,119,255,0.1), rgba(22,119,255,0.03));
    border-color: rgba(22,119,255,0.25);
  }

  .ov-label { font-size: 11px; color: $text-secondary; margin-bottom: 4px; }
  .ov-value { font-size: 20px; font-weight: 700; color: $text-primary; }
  .ov-sub { display: flex; gap: 6px; font-size: 12px; margin-top: 3px; }
}

/* ===== BODY ===== */
.pf-body {
  flex: 1;
  display: flex;
  gap: 12px;
  overflow: hidden;
}

/* ===== HOLDINGS ===== */
.holdings-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 10px;
  flex-shrink: 0;

  .panel-title { font-size: 14px; font-weight: 600; color: $text-primary; }
}

.holdings-table {
  flex: 1;
  overflow-y: auto;
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.ht-header {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr 1fr 1.5fr 1fr;
  padding: 8px 12px;
  font-size: 10px;
  color: $text-secondary;
  border-bottom: 1px solid $border-color;
  background: rgba(255,255,255,0.02);
  flex-shrink: 0;
}

.ht-row {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr 1fr 1.5fr 1fr;
  padding: 10px 12px;
  border-bottom: 1px solid rgba(48, 54, 61, 0.4);
  transition: background 0.15s;
  align-items: center;

  &:hover { background: $bg-hover; }
  &:last-child { border-bottom: none; }
}

.ht-name {
  .htn { font-size: 13px; color: $text-primary; font-weight: 500; }
  .htc { font-size: 10px; color: $text-secondary; }
}

.ht-cell { font-size: 12px; color: $text-primary; }
.small-pct { font-size: 10px; }

.ht-actions {
  display: flex;
  gap: 4px;
}

.act-btn {
  padding: 3px 8px;
  border: none;
  border-radius: 4px;
  font-size: 11px;
  cursor: pointer;
  font-weight: 500;

  &.up-bg { background: rgba(245,34,45,0.12); color: #f5222d; }
  &.down-bg { background: rgba(82,196,26,0.12); color: #52c41a; }
}

/* ===== CHARTS ===== */
.charts-panel {
  width: 280px;
  min-width: 280px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  overflow-y: auto;
}

.pf-card {
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
  padding: 12px;
  flex-shrink: 0;
}

.pc-title {
  font-size: 12px;
  font-weight: 600;
  color: $text-secondary;
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;

  .period-btns {
    display: flex;
    gap: 3px;
    button {
      background: transparent;
      border: 1px solid $border-color;
      color: $text-secondary;
      padding: 1px 6px;
      border-radius: 3px;
      font-size: 10px;
      cursor: pointer;
      &.active { background: $primary; border-color: $primary; color: #fff; }
    }
  }
}

.pie-container {
  display: flex;
  gap: 10px;
  align-items: center;
}

.pie-svg {
  width: 90px;
  height: 90px;
  flex-shrink: 0;
}

.pie-legend {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  .legend-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    flex-shrink: 0;
  }
  .legend-name { flex: 1; color: $text-secondary; }
  .legend-pct { color: $text-primary; font-weight: 500; }
}

.ret-chart {
  height: 100px;
  margin-bottom: 8px;
}

.ret-stats {
  display: flex;
  justify-content: space-between;
  .rs-item {
    display: flex;
    flex-direction: column;
    gap: 1px;
    .rs-label { font-size: 9px; color: $text-secondary; }
    .rs-val { font-size: 12px; font-weight: 600; color: $text-primary; }
  }
}

.risk-card {
  border-color: rgba(250, 173, 20, 0.25);
}

.risk-items {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.risk-item {
  font-size: 11px;
  padding: 5px 8px;
  border-radius: 5px;
  &.warn {
    background: rgba(250, 173, 20, 0.07);
    color: #faad14;
    border-left: 2px solid #faad14;
  }
}

.btn {
  display: inline-flex;
  align-items: center;
  padding: 5px 10px;
  border-radius: 5px;
  border: 1px solid $border-color;
  cursor: pointer;
  font-size: 11px;
  font-weight: 500;
  transition: all 0.2s;
  background: transparent;
  color: $text-secondary;
  &.btn-ghost {
    &:hover { background: $bg-hover; color: $text-primary; }
  }
  &.sm { font-size: 11px; padding: 4px 8px; }
}

.up { color: #f5222d !important; }
.down { color: #52c41a !important; }
.text-secondary { color: $text-secondary; }
</style>
