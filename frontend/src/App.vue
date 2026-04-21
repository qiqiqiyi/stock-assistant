<template>
  <div class="app-layout">
    <!-- 侧边栏 -->
    <aside class="sidebar" :class="{ collapsed: sidebarCollapsed }">
      <!-- Logo -->
      <div class="sidebar-logo">
        <div class="logo-icon">📈</div>
        <transition name="fade-slide">
          <span v-if="!sidebarCollapsed" class="logo-text">股票助手</span>
        </transition>
      </div>

      <!-- 导航菜单 -->
      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          class="nav-item"
          :class="{ active: currentPath === item.path }"
        >
          <span class="nav-icon">{{ item.icon }}</span>
          <transition name="fade-slide">
            <span v-if="!sidebarCollapsed" class="nav-label">{{ item.label }}</span>
          </transition>
          <transition name="fade-slide">
            <span v-if="!sidebarCollapsed && item.badge" class="nav-badge">{{ item.badge }}</span>
          </transition>
        </router-link>
      </nav>

      <!-- 底部 -->
      <div class="sidebar-footer">
        <button class="collapse-btn" @click="sidebarCollapsed = !sidebarCollapsed">
          <span>{{ sidebarCollapsed ? '▶' : '◀' }}</span>
        </button>
      </div>
    </aside>

    <!-- 主内容区 -->
    <main class="main-content">
      <!-- 顶部状态栏 -->
      <header class="top-bar">
        <div class="top-bar-left">
          <h1 class="page-title">{{ currentTitle }}</h1>
          <div class="market-status" :class="marketStatus.status">
            <span class="status-dot"></span>
            {{ marketStatus.text }}
          </div>
        </div>
        <div class="top-bar-right">
          <!-- 搜索框 -->
          <div class="search-box">
            <span class="search-icon">🔍</span>
            <input
              v-model="searchQuery"
              placeholder="搜索股票代码或名称..."
              @keyup.enter="handleSearch"
              @focus="showSearchHint = true"
              @blur="setTimeout(() => showSearchHint = false, 200)"
            />
            <!-- 搜索提示 -->
            <div v-if="showSearchHint && searchSuggestions.length" class="search-dropdown">
              <div
                v-for="s in searchSuggestions"
                :key="s.code"
                class="search-item"
                @mousedown="selectStock(s)"
              >
                <span class="s-code">{{ s.code }}</span>
                <span class="s-name">{{ s.name }}</span>
                <span class="s-price" :class="s.change >= 0 ? 'up' : 'down'">
                  {{ s.price }} ({{ s.change >= 0 ? '+' : '' }}{{ s.change }}%)
                </span>
              </div>
            </div>
          </div>

          <!-- 指数快捷显示 -->
          <div class="index-mini" v-for="idx in indexMini" :key="idx.code">
            <span class="idx-name">{{ idx.name }}</span>
            <span class="idx-val" :class="idx.change >= 0 ? 'up' : 'down'">
              {{ idx.price }}
            </span>
            <span class="idx-chg" :class="idx.change >= 0 ? 'up' : 'down'">
              {{ idx.change >= 0 ? '+' : '' }}{{ idx.change }}%
            </span>
          </div>

          <!-- 用户头像 -->
          <div class="user-avatar">
            <span>👤</span>
          </div>
        </div>
      </header>

      <!-- 路由视图 -->
      <div class="page-body">
        <router-view v-slot="{ Component }">
          <transition name="page-fade" mode="out-in">
            <component :is="Component" />
          </transition>
        </router-view>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

const sidebarCollapsed = ref(false)
const searchQuery = ref('')
const showSearchHint = ref(false)

const navItems = [
  { path: '/chat', icon: '🤖', label: 'AI 投资助手', badge: null },
  { path: '/market', icon: '📊', label: '市场行情', badge: null },
  { path: '/watchlist', icon: '⭐', label: '自选股', badge: '12' },
  { path: '/portfolio', icon: '💼', label: '投资组合', badge: null },
  { path: '/analysis', icon: '📈', label: '技术分析', badge: null }
]

const currentPath = computed(() => route.path)
const currentTitle = computed(() => {
  const item = navItems.find(n => n.path === route.path)
  return item ? item.label : '股票投资助手'
})

// 市场状态
const marketStatus = computed(() => {
  const now = new Date()
  const h = now.getHours()
  const m = now.getMinutes()
  const totalMin = h * 60 + m
  const day = now.getDay()
  const isWeekday = day >= 1 && day <= 5
  const isOpen = isWeekday && (
    (totalMin >= 9 * 60 + 30 && totalMin <= 11 * 60 + 30) ||
    (totalMin >= 13 * 60 && totalMin <= 15 * 60)
  )
  return isOpen
    ? { status: 'open', text: '交易中' }
    : { status: 'closed', text: '已收盘' }
})

// 模拟指数数据
const indexMini = ref([
  { code: 'sh', name: '上证', price: '3287.45', change: 0.82 },
  { code: 'sz', name: '深证', price: '10654.23', change: 1.15 },
  { code: 'cy', name: '创业板', price: '2143.67', change: -0.34 }
])

// 搜索建议（模拟）
const searchSuggestions = computed(() => {
  if (!searchQuery.value || searchQuery.value.length < 1) return []
  const q = searchQuery.value.toLowerCase()
  return mockStocks.filter(s =>
    s.code.toLowerCase().includes(q) || s.name.includes(q)
  ).slice(0, 6)
})

const mockStocks = [
  { code: '600519.SH', name: '贵州茅台', price: '1768.00', change: 1.23 },
  { code: '000858.SZ', name: '五粮液', price: '148.50', change: -0.87 },
  { code: '300750.SZ', name: '宁德时代', price: '215.30', change: 2.41 },
  { code: '000001.SZ', name: '平安银行', price: '11.22', change: 0.54 },
  { code: '600036.SH', name: '招商银行', price: '36.78', change: -0.22 },
  { code: '002594.SZ', name: '比亚迪', price: '302.45', change: 3.12 },
  { code: '601318.SH', name: '中国平安', price: '46.23', change: -1.05 },
  { code: '688111.SH', name: '金山办公', price: '64.70', change: 0.93 }
]

const handleSearch = () => {
  if (searchQuery.value) {
    // 跳转到分析页
    router.push({ path: '/analysis', query: { code: searchQuery.value } })
    searchQuery.value = ''
  }
}

const selectStock = (s) => {
  router.push({ path: '/analysis', query: { code: s.code } })
  searchQuery.value = ''
  showSearchHint.value = false
}

// 定时刷新指数 (模拟)
let timer = null
onMounted(() => {
  timer = setInterval(() => {
    indexMini.value = indexMini.value.map(idx => ({
      ...idx,
      change: parseFloat((idx.change + (Math.random() - 0.5) * 0.05).toFixed(2))
    }))
  }, 5000)
})
onUnmounted(() => clearInterval(timer))
</script>

<style lang="scss" scoped>
$sidebar-width: 220px;
$sidebar-collapsed: 64px;
$topbar-height: 56px;
$bg-dark: #0d1117;
$bg-card: #161b22;
$bg-hover: #1c2128;
$border-color: #30363d;
$text-primary: #e6edf3;
$text-secondary: #8b949e;
$primary: #1677ff;

.app-layout {
  display: flex;
  height: 100vh;
  overflow: hidden;
}

/* ===== SIDEBAR ===== */
.sidebar {
  width: $sidebar-width;
  min-width: $sidebar-width;
  background: $bg-card;
  border-right: 1px solid $border-color;
  display: flex;
  flex-direction: column;
  transition: width 0.25s ease, min-width 0.25s ease;
  overflow: hidden;

  &.collapsed {
    width: $sidebar-collapsed;
    min-width: $sidebar-collapsed;
  }
}

.sidebar-logo {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 18px 16px;
  border-bottom: 1px solid $border-color;
  white-space: nowrap;
  overflow: hidden;

  .logo-icon {
    font-size: 22px;
    flex-shrink: 0;
  }

  .logo-text {
    font-size: 16px;
    font-weight: 700;
    color: $text-primary;
    background: linear-gradient(135deg, #1677ff, #69c0ff);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
  }
}

.sidebar-nav {
  flex: 1;
  padding: 12px 8px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  overflow: hidden;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 10px;
  border-radius: 8px;
  color: $text-secondary;
  text-decoration: none;
  transition: all 0.2s;
  white-space: nowrap;
  overflow: hidden;
  cursor: pointer;

  &:hover {
    background: $bg-hover;
    color: $text-primary;
  }

  &.active {
    background: rgba(22, 119, 255, 0.12);
    color: #1677ff;

    .nav-icon {
      filter: none;
    }
  }

  .nav-icon {
    font-size: 18px;
    flex-shrink: 0;
  }

  .nav-label {
    font-size: 13px;
    font-weight: 500;
    flex: 1;
  }

  .nav-badge {
    background: rgba(22, 119, 255, 0.15);
    color: #1677ff;
    font-size: 10px;
    padding: 1px 6px;
    border-radius: 10px;
    font-weight: 600;
  }
}

.sidebar-footer {
  padding: 12px 8px;
  border-top: 1px solid $border-color;
  display: flex;
  justify-content: flex-end;
}

.collapse-btn {
  background: transparent;
  border: 1px solid $border-color;
  color: $text-secondary;
  border-radius: 6px;
  width: 28px;
  height: 28px;
  cursor: pointer;
  font-size: 10px;
  transition: all 0.2s;

  &:hover {
    background: $bg-hover;
    color: $text-primary;
  }
}

/* ===== MAIN ===== */
.main-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: $bg-dark;
}

/* ===== TOP BAR ===== */
.top-bar {
  height: $topbar-height;
  border-bottom: 1px solid $border-color;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
  background: $bg-card;
  flex-shrink: 0;
  gap: 16px;
}

.top-bar-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.page-title {
  font-size: 16px;
  font-weight: 600;
  color: $text-primary;
  white-space: nowrap;
}

.market-status {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 10px;

  .status-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
  }

  &.open {
    color: #52c41a;
    background: rgba(82, 196, 26, 0.1);
    .status-dot { background: #52c41a; animation: pulse 1.5s infinite; }
  }

  &.closed {
    color: #8b949e;
    background: rgba(139, 148, 158, 0.1);
    .status-dot { background: #8b949e; }
  }
}

.top-bar-right {
  display: flex;
  align-items: center;
  gap: 16px;
  flex: 1;
  justify-content: flex-end;
}

/* ===== SEARCH ===== */
.search-box {
  position: relative;
  display: flex;
  align-items: center;
  gap: 8px;
  background: $bg-dark;
  border: 1px solid $border-color;
  border-radius: 8px;
  padding: 6px 12px;
  min-width: 240px;

  &:focus-within {
    border-color: #1677ff;
    box-shadow: 0 0 0 2px rgba(22, 119, 255, 0.1);
  }

  .search-icon { color: $text-secondary; font-size: 13px; }

  input {
    background: transparent;
    border: none;
    outline: none;
    color: $text-primary;
    font-size: 13px;
    width: 100%;

    &::placeholder { color: $text-secondary; }
  }
}

.search-dropdown {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  right: 0;
  background: $bg-card;
  border: 1px solid $border-color;
  border-radius: 8px;
  overflow: hidden;
  z-index: 1000;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.4);
}

.search-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  cursor: pointer;
  transition: background 0.15s;

  &:hover { background: $bg-hover; }

  .s-code { color: $text-secondary; font-size: 11px; min-width: 90px; }
  .s-name { color: $text-primary; flex: 1; }
  .s-price { font-size: 12px; font-weight: 500; }
}

/* ===== INDEX MINI ===== */
.index-mini {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  padding: 4px 8px;
  background: $bg-dark;
  border: 1px solid $border-color;
  border-radius: 6px;
  white-space: nowrap;

  .idx-name { color: $text-secondary; }
  .idx-val { color: $text-primary; font-weight: 500; }
  .idx-chg { font-size: 11px; }
}

/* ===== USER ===== */
.user-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: rgba(22, 119, 255, 0.15);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 14px;
  border: 1px solid $border-color;

  &:hover { border-color: #1677ff; }
}

/* ===== PAGE ===== */
.page-body {
  flex: 1;
  overflow: hidden;
}

/* ===== TRANSITIONS ===== */
.fade-slide-enter-active,
.fade-slide-leave-active { transition: all 0.2s ease; }
.fade-slide-enter-from,
.fade-slide-leave-to { opacity: 0; transform: translateX(-6px); }

.page-fade-enter-active,
.page-fade-leave-active { transition: opacity 0.2s ease; }
.page-fade-enter-from,
.page-fade-leave-to { opacity: 0; }

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}
</style>
