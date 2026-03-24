<template>
    <div ref="containerRef" class="container" @scroll="onScroll">
        <div class="phantom" :style="{ height: totalHeight + 'px' }"></div>

        <div class="list">
            <div
                v-for="row in visibleData"
                :key="row.item.id ?? row.index"
                :ref="(el) => setItemRef(el, row.index)"
                class="item"
                :style="{
                    transform: `translate3d(0, ${Math.round(row.top)}px, 0)`,
                    minHeight: row.item && row.item.height ? row.item.height + 'px' : null,
                    borderTop: row.index === 0 ? 'none' : '1px solid #e6e6e6'
                }"
            >
                {{ row.item.text }}
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, computed, onMounted, watch, nextTick, onBeforeUnmount } from 'vue'

const emit = defineEmits(['mounted'])

const props = defineProps({
    items: {
        type: Array,
        default: () => []
    },
    // 兼容旧参数，固定高度时仍可直接传 itemHeight
    itemHeight: {
        type: Number,
        default: 50
    },
    // 可变高度场景下用于预估高度，提升滚动定位稳定性
    estimatedItemHeight: {
        type: Number,
        default: 50
    },
    // 可选：外部提供每一项高度（如 item.height）
    getItemHeight: {
        type: Function,
        default: null
    }
})

const setupStartTime = performance.now()
const overscanCount = 8
const overscanPixels = 800

const containerRef = ref(null)
const containerHeight = ref(0)
const scrollTop = ref(0)
const itemRefs = ref({})

const sizes = ref([])
const tree = ref([])
const treeVersion = ref(0)
const itemsLength = computed(() => props.items.length)

const getDefaultHeight = (index) => {
    if (typeof props.getItemHeight === 'function') {
        const customHeight = props.getItemHeight(props.items[index], index)
        if (typeof customHeight === 'number' && customHeight > 0) {
            return customHeight
        }
    }

    const item = props.items[index]
    if (item && typeof item.height === 'number' && item.height > 0) {
        return item.height
    }

    return props.estimatedItemHeight || props.itemHeight
}

const initFenwick = () => {
    const n = itemsLength.value
    const nextSizes = new Array(n)
    for (let i = 0; i < n; i += 1) {
        nextSizes[i] = getDefaultHeight(i)
    }

    const nextTree = new Array(n + 1).fill(0)
    for (let i = 1; i <= n; i += 1) {
        nextTree[i] += nextSizes[i - 1]
        const parent = i + (i & -i)
        if (parent <= n) {
            nextTree[parent] += nextTree[i]
        }
    }

    sizes.value = nextSizes
    tree.value = nextTree
    treeVersion.value += 1
}

const fenwickPrefixSum = (count) => {
    let i = Math.max(0, Math.min(count, itemsLength.value))
    let sum = 0
    while (i > 0) {
        sum += tree.value[i]
        i -= i & -i
    }
    return sum
}

const findIndexByOffset = (offsetTop) => {
    const n = itemsLength.value
    if (n === 0) {
        return 0
    }

    let target = Math.max(0, offsetTop)
    let idx = 0
    let bitMask = 1
    while ((bitMask << 1) <= n) {
        bitMask <<= 1
    }

    while (bitMask !== 0) {
        const next = idx + bitMask
        if (next <= n && tree.value[next] <= target) {
            idx = next
            target -= tree.value[next]
        }
        bitMask >>= 1
    }

    return Math.min(idx, n - 1)
}

const updateSizesBatch = (updates) => {
    const n = itemsLength.value
    if (!n || !updates.length) {
        return
    }

    let changed = false
    for (let k = 0; k < updates.length; k += 1) {
        const { index, size } = updates[k]
        if (index < 0 || index >= n) {
            continue
        }

        const prevSize = sizes.value[index]
        const delta = size - prevSize
        if (Math.abs(delta) < 1) {
            continue
        }

        sizes.value[index] = size
        for (let i = index + 1; i <= n; i += i & -i) {
            tree.value[i] += delta
        }
        changed = true
    }

    if (changed) {
        treeVersion.value += 1
    }
}

// 总高度
const totalHeight = computed(() => {
    treeVersion.value
    return fenwickPrefixSum(itemsLength.value)
})

// 起始索引
const startIndex = computed(() => {
    treeVersion.value
    return findIndexByOffset(scrollTop.value)
})

const renderStartIndex = computed(() => {
    return Math.max(0, startIndex.value - overscanCount)
})

const endIndex = computed(() => {
    treeVersion.value
    const max = itemsLength.value
    const targetOffset = scrollTop.value + containerHeight.value + overscanPixels
    const visibleEnd = findIndexByOffset(targetOffset)
    return Math.min(max, visibleEnd + 1 + overscanCount)
})

// 可见数据
const visibleData = computed(() => {
    treeVersion.value
    const start = renderStartIndex.value
    const end = endIndex.value
    const rows = []

    for (let i = start; i < end; i += 1) {
        rows.push({
            item: props.items[i],
            index: i,
            top: fenwickPrefixSum(i)
        })
    }

    return rows
})

// const onScroll = (e) => {
//     scrollTop.value = e.target.scrollTop
// }
let ticking = false
let latestScrollTop = 0

const onScroll = (e) => {
    latestScrollTop = e.target.scrollTop

    if (!ticking) {
        requestAnimationFrame(() => {
            scrollTop.value = latestScrollTop
            ticking = false
        })
        ticking = true
    }
}

const setItemRef = (el, index) => {
    if (el) {
        itemRefs.value[index] = el
        return
    }

    delete itemRefs.value[index]
}

const measureVisibleHeights = () => {
    const updates = []

    visibleData.value.forEach((row) => {
        const el = itemRefs.value[row.index]
        if (!el) {
            return
        }

        const measured = el.offsetHeight
        if (measured > 0) {
            updates.push({ index: row.index, size: measured })
        }
    })

    updateSizesBatch(updates)
}

let measureRaf = 0
const scheduleMeasure = () => {
    if (measureRaf) {
        return
    }

    measureRaf = requestAnimationFrame(() => {
        measureRaf = 0
        measureVisibleHeights()
    })
}

const scrollToOffset = (offset) => {
    const el = containerRef.value
    if (!el) {
        return
    }

    const maxTop = Math.max(0, totalHeight.value - containerHeight.value)
    const nextTop = Math.max(0, Math.min(offset, maxTop))
    el.scrollTop = nextTop
    scrollTop.value = nextTop
    scheduleMeasure()
}

const scrollToIndex = (index, align = 'start') => {
    const n = itemsLength.value
    if (!n) {
        return
    }

    const target = Math.max(0, Math.min(index, n - 1))
    const itemTop = fenwickPrefixSum(target)
    const itemBottom = fenwickPrefixSum(target + 1)
    const viewportHeight = containerHeight.value || containerRef.value?.clientHeight || 0
    const itemHeight = Math.max(0, itemBottom - itemTop)

    let nextTop = itemTop
    if (align === 'center') {
        nextTop = itemTop - (viewportHeight - itemHeight) / 2
    } else if (align === 'end') {
        nextTop = itemBottom - viewportHeight
    } else if (align === 'auto') {
        const currentTop = scrollTop.value
        const currentBottom = currentTop + viewportHeight
        if (itemTop < currentTop) {
            nextTop = itemTop
        } else if (itemBottom > currentBottom) {
            nextTop = itemBottom - viewportHeight
        } else {
            nextTop = currentTop
        }
    }

    scrollToOffset(nextTop)
}

let resizeObserver = null

defineExpose({
    scrollToIndex,
    scrollToOffset
})

watch(
    () => props.items,
    () => {
        itemRefs.value = {}
        initFenwick()
    },
    { immediate: true }
)

watch(
    visibleData,
    async () => {
        await nextTick()
        scheduleMeasure()
    },
    { flush: 'post' }
)

onMounted(() => {
    containerHeight.value = containerRef.value.clientHeight
    scheduleMeasure()
    emit('mounted', performance.now() - setupStartTime)

    if (typeof ResizeObserver !== 'undefined' && containerRef.value) {
        resizeObserver = new ResizeObserver((entries) => {
            const entry = entries[0]
            if (!entry) {
                return
            }
            containerHeight.value = entry.contentRect.height
        })
        resizeObserver.observe(containerRef.value)
    }
})

onBeforeUnmount(() => {
    if (measureRaf) {
        cancelAnimationFrame(measureRaf)
        measureRaf = 0
    }

    if (resizeObserver) {
        resizeObserver.disconnect()
        resizeObserver = null
    }
})
</script>

<style scoped>
.container {
    height: 500px;
    overflow-y: auto;
    border: 1px solid #ccc;
    position: relative;
}

.phantom {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
}

.list {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
}

.item {
    position: absolute;
    left: 0;
    right: 0;
    min-height: 50px;
    line-height: 1.4;
    padding: 12px 10px;
    box-sizing: border-box;
    text-align: left;
    background: #fff;
    will-change: transform;
}
</style>