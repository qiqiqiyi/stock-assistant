<template>
  <div class="stock-card" @click="$emit('click', stock)">
    <div class="card-header">
      <div class="name-code">
        <span class="s-name">{{ stock.name }}</span>
        <span class="s-code">{{ stock.code }}</span>
      </div>
      <div class="price-block">
        <span class="price">¥{{ stock.price }}</span>
        <span class="change-tag" :class="stock.change >= 0 ? 'tag-up' : 'tag-down'">
          {{ stock.change >= 0 ? '+' : '' }}{{ stock.change }}%
        </span>
      </div>
    </div>
    <div class="card-metrics">
      <div class="metric" v-if="stock.volume">
        <span class="m-label">成交量</span>
        <span class="m-val">{{ stock.volume }}</span>
      </div>
      <div class="metric" v-if="stock.amount">
        <span class="m-label">成交额</span>
        <span class="m-val">{{ stock.amount }}</span>
      </div>
      <div class="metric" v-if="stock.high">
        <span class="m-label">最高</span>
        <span class="m-val up">{{ stock.high }}</span>
      </div>
      <div class="metric" v-if="stock.low">
        <span class="m-label">最低</span>
        <span class="m-val down">{{ stock.low }}</span>
      </div>
      <div class="metric" v-if="stock.pe">
        <span class="m-label">市盈率</span>
        <span class="m-val">{{ stock.pe }}x</span>
      </div>
      <div class="metric" v-if="stock.pb">
        <span class="m-label">市净率</span>
        <span class="m-val">{{ stock.pb }}x</span>
      </div>
    </div>
  </div>
</template>

<script setup>
defineProps({
  stock: { type: Object, required: true }
})
defineEmits(['click'])
</script>

<style lang="scss" scoped>
$bg-dark: #0d1117;
$bg-hover: #1c2128;
$border-color: #30363d;
$text-primary: #e6edf3;
$text-secondary: #8b949e;

.stock-card {
  background: $bg-dark;
  border: 1px solid $border-color;
  border-radius: 8px;
  padding: 10px 12px;
  min-width: 200px;
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    border-color: #1677ff;
    background: $bg-hover;
  }
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.name-code {
  display: flex;
  flex-direction: column;
  gap: 1px;
  .s-name { font-size: 13px; font-weight: 600; color: $text-primary; }
  .s-code { font-size: 10px; color: $text-secondary; }
}

.price-block {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 2px;
  .price { font-size: 15px; font-weight: 700; color: $text-primary; }
}

.change-tag {
  display: inline-flex;
  padding: 1px 6px;
  border-radius: 3px;
  font-size: 11px;
  font-weight: 600;
  &.tag-up { background: rgba(245,34,45,0.1); color: #f5222d; }
  &.tag-down { background: rgba(82,196,26,0.1); color: #52c41a; }
}

.card-metrics {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 4px;
}

.metric {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  .m-label { color: $text-secondary; }
  .m-val { color: $text-primary; font-weight: 500; }
}

.up { color: #f5222d; }
.down { color: #52c41a; }
</style>
