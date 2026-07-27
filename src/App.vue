<template>
  <div class="app-container" role="main" aria-label="排序算法可视化工具">
    <h1>排序算法可视化</h1>

    <div class="controls">
      <label>
        算法
        <select v-model="selectedAlgorithm" :disabled="isSorting">
          <option value="bubble">冒泡排序</option>
          <option value="selection">选择排序</option>
          <option value="insertion">插入排序</option>
          <option value="quick">快速排序</option>
          <option value="heap">堆排序</option>
          <option value="bucket">桶排序</option>
        </select>
      </label>

      <button class="btn-success" @click="generateArray" :disabled="isSorting">
        生成新数组
      </button>

      <button
        v-if="isSorting && sortPaused"
        class="btn-primary"
        @click="resumeSort"
      >
        继续
      </button>
      <button
        v-else
        class="btn-primary"
        @click="startSort"
        :disabled="sorted && !isSorting"
      >
        {{ isSorting ? '暂停' : '开始排序' }}
      </button>

      <button @click="resetSort" :disabled="!isSorting && !sorted">
        重置
      </button>

      <label>
        速度
        <input
          type="range"
          min="1"
          max="100"
          v-model.number="speed"
          class="slider"
          aria-label="动画速度调节"
        />
      </label>
    </div>

    <div class="chart-container" ref="chartRef">
      <div
        v-for="item in array"
        :key="item.id"
        class="bar"
        :class="item.state"
        :style="{ height: (item.value / maxVal) * 100 + '%' }"
        :title="`值: ${item.value}`"
      >
        <span v-if="array.length <= 35" class="bar-label">{{ item.value }}</span>
      </div>
      <div v-if="array.length === 0" class="empty-hint">点击「生成新数组」开始</div>
    </div>

    <div class="info">
      <span v-if="array.length">元素数: {{ array.length }}</span>
      <span>操作: {{ operationCount }}</span>
      <span>状态: {{ statusText }}</span>
    </div>
    <div class="footer-note">
      颜色图例:
      <span class="legend default"></span> 默认
      <span class="legend comparing"></span> 比较
      <span class="legend swapping"></span> 交换
      <span class="legend pivot"></span> 基准
      <span class="legend sorted"></span> 已排序
      &nbsp;| 快捷键: <kbd>空格</kbd> 开始/暂停 &nbsp;<kbd>R</kbd> 重置
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

// ---------- 响应式状态 ----------
const array = ref([])
const selectedAlgorithm = ref('bubble')
const speed = ref(50)
const isSorting = ref(false)
const sorted = ref(false)
const operationCount = ref(0)
const statusText = ref('就绪')
const maxVal = ref(100)

// 内部控制变量（不触发视图更新）
let sortCancel = false
let sortPaused = false
let pauseResolve = null

// ---------- 辅助函数 ----------
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms))
}

async function waitIfPausedOrCancelled() {
  while (sortPaused && !sortCancel) {
    await new Promise(resolve => {
      pauseResolve = resolve
    })
  }
  if (sortCancel) throw new Error('Cancelled')
}

// ---------- 生成随机数组 ----------
function generateArray() {
  if (isSorting.value) return
  const len = 30 + Math.floor(Math.random() * 20)
  const newArr = []
  for (let i = 0; i < len; i++) {
    newArr.push({
      id: i,
      value: Math.floor(Math.random() * 90) + 10,
      state: 'default'
    })
  }
  array.value = newArr
  maxVal.value = Math.max(...newArr.map(item => item.value))
  sorted.value = false
  operationCount.value = 0
  statusText.value = '已生成新数组'
  sortCancel = false
  sortPaused = false
  pauseResolve = null
}

// ---------- 重置 ----------
function resetSort() {
  if (isSorting.value) {
    sortCancel = true
    if (pauseResolve) {
      pauseResolve()
      pauseResolve = null
    }
    sortPaused = false
  }
  if (array.value.length) {
    array.value.forEach(item => {
      item.state = 'default'
    })
  }
  isSorting.value = false
  sorted.value = false
  operationCount.value = 0
  statusText.value = '已重置'
  sortCancel = false
  sortPaused = false
}

// ---------- 暂停 / 继续 ----------
function resumeSort() {
  if (isSorting.value && sortPaused) {
    sortPaused = false
    statusText.value = '排序中...'
    if (pauseResolve) {
      pauseResolve()
      pauseResolve = null
    }
  }
}

// ---------- 排序可视化操作 ----------
async function swap(arr, i, j, delay) {
  if (i === j) return
  arr[i].state = 'swapping'
  arr[j].state = 'swapping'
  operationCount.value++
  await sleep(delay)
  const temp = arr[i].value
  arr[i].value = arr[j].value
  arr[j].value = temp
  arr[i].state = 'default'
  arr[j].state = 'default'
  await sleep(delay / 2)
}

async function compare(arr, i, j, delay) {
  arr[i].state = 'comparing'
  arr[j].state = 'comparing'
  operationCount.value++
  await sleep(delay)
  const result = arr[i].value > arr[j].value
  arr[i].state = 'default'
  arr[j].state = 'default'
  return result
}

function markSorted(arr, index) {
  arr[index].state = 'sorted'
}

async function setPivot(arr, index, delay) {
  arr[index].state = 'pivot'
  await sleep(delay)
}

// ---------- 排序算法 ----------
// 1. 冒泡排序
async function bubbleSort(arr, delay) {
  const n = arr.length
  for (let i = 0; i < n - 1; i++) {
    let swapped = false
    for (let j = 0; j < n - 1 - i; j++) {
      await waitIfPausedOrCancelled()
      if (await compare(arr, j, j + 1, delay)) {
        await swap(arr, j, j + 1, delay)
        swapped = true
      }
    }
    markSorted(arr, n - 1 - i)
    if (!swapped) break
  }
  finalizeSorted(arr)
}

// 2. 选择排序
async function selectionSort(arr, delay) {
  const n = arr.length
  for (let i = 0; i < n - 1; i++) {
    let minIdx = i
    for (let j = i + 1; j < n; j++) {
      await waitIfPausedOrCancelled()
      if (await compare(arr, minIdx, j, delay)) minIdx = j
    }
    if (minIdx !== i) await swap(arr, i, minIdx, delay)
    markSorted(arr, i)
  }
  markSorted(arr, n - 1)
}

// 3. 插入排序
async function insertionSort(arr, delay) {
  const n = arr.length
  for (let i = 1; i < n; i++) {
    let j = i
    while (j > 0) {
      await waitIfPausedOrCancelled()
      if (!(await compare(arr, j - 1, j, delay))) break
      await swap(arr, j - 1, j, delay)
      j--
    }
  }
  finalizeSorted(arr)
}

// 4. 快速排序
async function quickSort(arr, left, right, delay) {
  if (left >= right) {
    if (left >= 0 && left < arr.length) markSorted(arr, left)
    return
  }
  const pivotIdx = right
  await setPivot(arr, pivotIdx, delay)
  let i = left

  for (let j = left; j < right; j++) {
    await waitIfPausedOrCancelled()
    if (!(await compare(arr, j, pivotIdx, delay))) {
      if (i !== j) await swap(arr, i, j, delay)
      i++
    }
  }
  await swap(arr, i, right, delay)
  markSorted(arr, i)
  await quickSort(arr, left, i - 1, delay)
  await quickSort(arr, i + 1, right, delay)
}

// 5. 堆排序
async function heapSort(arr, delay) {
  const n = arr.length
  for (let i = Math.floor(n / 2) - 1; i >= 0; i--) {
    await heapify(arr, n, i, delay)
  }
  for (let i = n - 1; i > 0; i--) {
    await waitIfPausedOrCancelled()
    await swap(arr, 0, i, delay)
    markSorted(arr, i)
    await heapify(arr, i, 0, delay)
  }
  markSorted(arr, 0)
}

async function heapify(arr, n, i, delay) {
  let largest = i
  const l = 2 * i + 1
  const r = 2 * i + 2

  if (l < n) {
    await waitIfPausedOrCancelled()
    if (await compare(arr, l, largest, delay)) largest = l
  }
  if (r < n) {
    await waitIfPausedOrCancelled()
    if (await compare(arr, r, largest, delay)) largest = r
  }
  if (largest !== i) {
    await swap(arr, i, largest, delay)
    await heapify(arr, n, largest, delay)
  }
}

// 6. 桶排序（计数排序可视化版本）
async function bucketSort(arr, delay) {
  const n = arr.length
  let max = arr[0].value
  for (let i = 1; i < n; i++) {
    if (arr[i].value > max) max = arr[i].value
  }
  const bucket = new Array(max + 1).fill(0)
  for (let i = 0; i < n; i++) {
    bucket[arr[i].value]++
  }
  let idx = 0
  for (let val = 0; val <= max; val++) {
    while (bucket[val] > 0) {
      await waitIfPausedOrCancelled()
      let found = -1
      for (let k = idx; k < n; k++) {
        if (arr[k].value === val) {
          found = k
          break
        }
      }
      if (found !== -1 && found !== idx) {
        await swap(arr, idx, found, delay)
      }
      markSorted(arr, idx)
      bucket[val]--
      idx++
    }
  }
}

function finalizeSorted(arr) {
  for (let i = 0; i < arr.length; i++) {
    arr[i].state = 'sorted'
  }
}

// ---------- 排序调度 ----------
async function runSort() {
  if (isSorting.value) return
  sortCancel = false
  sortPaused = false
  isSorting.value = true
  sorted.value = false
  statusText.value = '排序中...'
  operationCount.value = 0

  const arr = array.value
  const delay = Math.floor(400 / (speed.value / 10 + 1))

  try {
    switch (selectedAlgorithm.value) {
      case 'bubble': await bubbleSort(arr, delay); break
      case 'selection': await selectionSort(arr, delay); break
      case 'insertion': await insertionSort(arr, delay); break
      case 'quick': await quickSort(arr, 0, arr.length - 1, delay); break
      case 'heap': await heapSort(arr, delay); break
      case 'bucket': await bucketSort(arr, delay); break
    }
    if (!sortCancel) {
      sorted.value = true
      statusText.value = '排序完成'
    }
  } catch (err) {
    if (err.message !== 'Cancelled') {
      console.error('排序出错:', err)
      statusText.value = '出错'
    }
  } finally {
    isSorting.value = false
    sortPaused = false
    sortCancel = false
    pauseResolve = null
  }
}

function startSort() {
  if (isSorting.value && !sortPaused) {
    sortPaused = true
    statusText.value = '已暂停'
    return
  }
  if (isSorting.value && sortPaused) {
    resumeSort()
    return
  }
  if (sorted.value) {
    resetSort()
    setTimeout(runSort, 50)
  } else {
    runSort()
  }
}

// ---------- 键盘快捷键 ----------
function handleKeydown(e) {
  // 忽略输入框中的按键
  if (e.target.tagName === 'INPUT' || e.target.tagName === 'SELECT') return

  if (e.code === 'Space') {
    e.preventDefault()
    if (!isSorting.value || sortPaused) {
      startSort()
    } else if (isSorting.value && !sortPaused) {
      sortPaused = true
      statusText.value = '已暂停'
    }
  }
  if (e.code === 'KeyR' && !isSorting.value) {
    e.preventDefault()
    resetSort()
  }
}

onMounted(() => {
  generateArray()
  window.addEventListener('keydown', handleKeydown)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeydown)
})
</script>

<style scoped>
/* ---------- 布局 ---------- */
.app-container {
  width: 1100px;
  max-width: 98vw;
  margin: 20px auto;
  background: #2d2d44;
  border-radius: 20px;
  padding: 30px 40px 40px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.6);
}

h1 {
  text-align: center;
  font-weight: 300;
  letter-spacing: 2px;
  margin-bottom: 15px;
  font-size: 28px;
  color: #c0c0e0;
}

/* ---------- 控制栏 ---------- */
.controls {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
  gap: 12px 18px;
  margin-bottom: 25px;
}

.controls label {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  color: #aaaac0;
}

.controls select,
.controls button {
  padding: 8px 16px;
  border-radius: 8px;
  border: none;
  background: #3e3e5a;
  color: #fff;
  font-size: 14px;
  cursor: pointer;
  transition: background 0.2s, opacity 0.2s;
}

.controls select:hover,
.controls button:hover:not(:disabled) {
  background: #5a5a7a;
}

.controls button:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.controls .btn-primary {
  background: #6c5ce7;
}
.controls .btn-primary:hover:not(:disabled) {
  background: #7d6ff0;
}

.controls .btn-success {
  background: #00b894;
}
.controls .btn-success:hover:not(:disabled) {
  background: #00c9a0;
}

.controls .slider {
  width: 100px;
  accent-color: #6c5ce7;
  cursor: pointer;
}

/* ---------- 图表区域 ---------- */
.chart-container {
  background: #1a1a2e;
  border-radius: 16px;
  padding: 24px 12px 12px;
  height: 340px;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  gap: 3px;
  overflow: hidden;
}

.bar {
  flex: 1 1 0;
  min-width: 2px;
  background: #4a6fa5;
  border-radius: 6px 6px 0 0;
  transition: background 0.15s ease, height 0.08s ease;
  position: relative;
  min-height: 4px;
}

.bar-label {
  position: absolute;
  top: -22px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 10px;
  color: #aaa;
  font-weight: 500;
  white-space: nowrap;
  pointer-events: none;
}

.bar.comparing  { background: #fdcb6e; }
.bar.swapping   { background: #e17055; }
.bar.sorted     { background: #00b894; }
.bar.pivot      { background: #a29bfe; }

.empty-hint {
  color: #5a5a7a;
  font-size: 18px;
  align-self: center;
}

/* ---------- 信息栏 ---------- */
.info {
  display: flex;
  justify-content: center;
  gap: 12px;
  flex-wrap: wrap;
  margin-top: 16px;
  font-size: 14px;
  color: #a0a0b8;
}

.info span {
  background: #25253a;
  padding: 4px 14px;
  border-radius: 20px;
}

/* ---------- 底部图例 ---------- */
.footer-note {
  text-align: center;
  margin-top: 18px;
  font-size: 13px;
  color: #6a6a82;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  flex-wrap: wrap;
}

.legend {
  display: inline-block;
  width: 14px;
  height: 14px;
  border-radius: 4px;
  vertical-align: middle;
}

.legend.default   { background: #4a6fa5; }
.legend.comparing { background: #fdcb6e; }
.legend.swapping  { background: #e17055; }
.legend.sorted    { background: #00b894; }
.legend.pivot     { background: #a29bfe; }

.footer-note kbd {
  background: #3e3e5a;
  color: #c0c0e0;
  padding: 1px 6px;
  border-radius: 4px;
  font-size: 12px;
  font-family: inherit;
  border: 1px solid #5a5a7a;
}

/* ---------- 响应式 ---------- */
@media (max-width: 768px) {
  .app-container {
    padding: 20px 16px 24px;
    border-radius: 12px;
  }
  h1 {
    font-size: 22px;
  }
  .chart-container {
    height: 280px;
    padding: 20px 8px 8px;
  }
  .controls {
    gap: 8px 10px;
  }
  .controls select,
  .controls button {
    padding: 6px 12px;
    font-size: 13px;
  }
  .footer-note {
    gap: 4px;
    font-size: 11px;
  }
}
</style>
