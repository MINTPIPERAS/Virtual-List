<template>
  <div class="container">
    <div
      v-for="item in items"
      :key="item.id"
      class="item"
      :style="item.height ? { minHeight: item.height + 'px' } : null"
    >
      {{ item.text }}
    </div>
  </div>
</template>

<script setup>
import { onMounted } from 'vue'

const emit = defineEmits(['mounted'])

defineProps({
  items: {
    type: Array,
    default: () => []
  }
})

const setupStartTime = performance.now()

onMounted(() => {
  emit('mounted', performance.now() - setupStartTime)
})
</script>

<style scoped>
.container {
  height: 500px;
  overflow-y: auto;
  border: 1px solid #ccc;
}

.item {
  line-height: 1.4;
  border-bottom: 1px solid #eee;
  padding: 12px 10px;
  box-sizing: border-box;
  text-align: left;
}
</style>
