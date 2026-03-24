<template>
  <main class="page">
    <h1>100000 条数据渲染对比</h1>

    <section class="compare-grid">
      <div class="panel">
        <h2>虚拟列表</h2>
        <p class="meta">仅渲染可视区附近节点，支持不同高度 item（自动测量）。</p>
        <div class="actions">
          <input
            v-model.number="targetIndex"
            class="index-input"
            type="number"
            min="0"
            :max="listData.length - 1"
            placeholder="输入索引"
          />
          <button class="mount-btn" @click="handleScrollToIndex">滚动到指定索引</button>
        </div>
        <p v-if="virtualMountCost > 0" class="time">挂载耗时：{{ virtualMountCost.toFixed(1) }} ms</p>
        <VirtualList
          ref="virtualListRef"
          :items="listData"
          :estimated-item-height="70"
          @mounted="onVirtualMounted"
        />
      </div>

      <div class="panel">
        <h2>普通列表</h2>
        <p class="meta">一次性渲染 100000 个真实 DOM 节点，容易导致页面整体卡顿。</p>
        <p class="meta">整体渲染完成才会不那么卡顿，但快速滑动时仍可能卡顿、留白。</p>
        <div class="actions">
          <button v-if="!showNormalList" class="mount-btn" @click="showNormalList = true">
            开始渲染普通列表（100000）
          </button>
          <p v-if="normalMountCost > 0" class="time">挂载耗时：{{ normalMountCost.toFixed(1) }} ms</p>
        </div>
        <NormalList v-if="showNormalList" :items="listData" @mounted="onNormalMounted" />
        <div v-else class="placeholder">点击按钮后再渲染普通列表，避免初始化时拖慢整个页面。</div>
      </div>
    </section>
  </main>
</template>

<script setup>
import { ref } from 'vue'
import VirtualList from './components/VirtualList.vue'
import NormalList from './components/NormalList.vue'

const listData = ref(
  Array.from({ length: 100000 }, (_, i) => ({
    id: i,
    height: 44 + (i % 4) * 18,
    text:
      i % 5 === 0
        ? `Item ${i} - 这是较长文本，用于模拟多行内容和不同高度场景。`
        : `Item ${i}`
  }))
)

const showNormalList = ref(false)
const virtualMountCost = ref(0)
const normalMountCost = ref(0)
const targetIndex = ref(5000)
const virtualListRef = ref(null)

const onVirtualMounted = (cost) => {
  virtualMountCost.value = cost
}

const onNormalMounted = (cost) => {
  normalMountCost.value = cost
}

const handleScrollToIndex = () => {
  if (!virtualListRef.value) {
    return
  }

  const maxIndex = listData.value.length - 1
  const safeIndex = Math.max(0, Math.min(Number(targetIndex.value) || 0, maxIndex))
  targetIndex.value = safeIndex
  virtualListRef.value.scrollToIndex(safeIndex, 'start')
}
</script>

<style scoped>
.page {
  padding: 24px;
  box-sizing: border-box;
}

h1 {
  margin: 0 0 20px;
  font-size: 28px;
}

h2 {
  margin: 0 0 12px;
  font-size: 18px;
}

.compare-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

.panel {
  min-width: 0;
}

.meta {
  margin: 0 0 8px;
  color: #666;
  text-align: left;
  font-size: 14px;
}

.actions {
  margin-bottom: 8px;
  text-align: left;
  display: flex;
  gap: 8px;
  align-items: center;
  flex-wrap: wrap;
}

.index-input {
  width: 180px;
  border: 1px solid #ccc;
  border-radius: 6px;
  padding: 8px 10px;
}

.mount-btn {
  border: 1px solid #2f6feb;
  background: #2f6feb;
  color: #fff;
  border-radius: 6px;
  padding: 8px 12px;
  cursor: pointer;
}

.time {
  margin: 6px 0 0;
  text-align: left;
  color: #0b7a4b;
  font-weight: 600;
}

.placeholder {
  height: 500px;
  border: 1px dashed #bbb;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #666;
  padding: 12px;
  box-sizing: border-box;
}

@media (max-width: 960px) {
  .compare-grid {
    grid-template-columns: 1fr;
  }
}
</style>