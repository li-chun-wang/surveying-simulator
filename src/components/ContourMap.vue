<template>
  <div class="contour-map">
    <!-- 画布容器 -->
    <div class="canvas-container border-2 border-gray-200 rounded-xl overflow-hidden shadow-sm bg-white mb-4" style="height: 400px;">
      <canvas 
        ref="canvasRef" 
        @click="handleCellClick"
        class="w-full h-full"  
        style="display: block;"  
      ></canvas>
    </div>
    
    <!-- 控制按钮 -->
    <div class="contour-controls flex flex-wrap items-center gap-3">
      <div class="flex gap-3">
        <button 
          @click="generateFlatTerrain"
          class="px-4 py-2 bg-white border border-gray-200 rounded-lg hover:bg-gray-50 transition flex items-center"
        >
          <i class="fa fa-square-o mr-2 text-gray-600"></i>平坦地形
        </button>
        <button 
          @click="generateHillTerrain"
          class="px-4 py-2 bg-white border border-gray-200 rounded-lg hover:bg-gray-50 transition flex items-center"
        >
          <i class="fa fa-mound mr-2 text-gray-600"></i>山丘地形
        </button>
        <button 
          @click="generateValleyTerrain"
          class="px-4 py-2 bg-white border border-gray-200 rounded-lg hover:bg-gray-50 transition flex items-center"
        >
          <i class="fa fa-downtime mr-2 text-gray-600"></i>山谷地形
        </button>
      </div>
      
      <div class="flex gap-3 ml-auto">
        <div class="contour-interval flex items-center gap-2">
          <label class="text-sm font-medium text-gray-600">等高距:</label>
          <select v-model="contourInterval" class="px-3 py-2 border border-gray-200 rounded-lg text-sm">
            <option value="5">5m</option>
            <option value="10">10m</option>
            <option value="20">20m</option>
          </select>
        </div>
        <button 
          @click="drawContours"
          class="px-4 py-2 bg-primary text-white rounded-lg hover:bg-primary/90 transition flex items-center"
        >
          <i class="fa fa-draw-polygon mr-2"></i>绘制等高线
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick, onUnmounted } from 'vue'

const canvasRef = ref(null)
const canvas = ref(null)
const ctx = ref(null)
const cellCount = 14
const terrainData = ref([])
const contourInterval = ref(10)
const precision = ref(0.5)
const contourLines = ref({})
const isActive = ref(false)

// 接收外部参数
const props = defineProps({
  precision: { type: Number, default: 0.5 },
  active: { type: Boolean, default: false }
})

watch(() => props.precision, (newVal) => precision.value = newVal)

// 监听激活状态
watch(() => props.active, (newVal) => {
  isActive.value = newVal
  if (newVal) {
    nextTick(() => initCanvas())
  } else {
    cleanupCanvas()
  }
}, { immediate: true })

// 初始化画布
const initCanvas = () => {
  if (!canvasRef.value) return
  canvas.value = canvasRef.value
  ctx.value = canvas.value.getContext('2d')
  initTerrainData()
  resizeCanvas()
  window.addEventListener('resize', debouncedResize)
}

// 清理画布
const cleanupCanvas = () => {
  window.removeEventListener('resize', debouncedResize)
  if (ctx.value) {
    ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height)
  }
}

// 初始化地形数组
const initTerrainData = () => {
  terrainData.value = []
  for (let y = 0; y < cellCount; y++) {
    terrainData.value.push(new Array(cellCount).fill(150)) // 默认高程150m
  }
  contourLines.value = {}
}

// 防抖resize
const debouncedResize = debounce(() => {
  if (isActive.value) resizeCanvas()
}, 100)

// 画布尺寸调整
const resizeCanvas = () => {
  if (!canvas.value || !ctx.value) return
  const container = canvas.value.parentElement
  // 宽度随容器自适应，高度固定为400px（与全站仪一致）
  canvas.value.width = container.clientWidth
  canvas.value.height = 400  // 强制固定高度
  drawGrid()
}

// 绘制网格+地形+等高线
const drawGrid = () => {
  if (!ctx.value || !terrainData.value.length) return
  ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height)
  const cellW = canvas.value.width / cellCount
  const cellH = canvas.value.height / cellCount

  // 1. 绘制网格
  ctx.value.strokeStyle = '#f0f0f0'
  ctx.value.lineWidth = 1
  for (let y = 0; y <= cellCount; y++) {
    ctx.value.beginPath()
    ctx.value.moveTo(0, y * cellH)
    ctx.value.lineTo(canvas.value.width, y * cellH)
    ctx.value.stroke()
  }
  for (let x = 0; x <= cellCount; x++) {
    ctx.value.beginPath()
    ctx.value.moveTo(x * cellW, 0)
    ctx.value.lineTo(x * cellW, canvas.value.height)
    ctx.value.stroke()
  }

  // 2. 绘制地形
  for (let y = 0; y < cellCount; y++) {
    const row = terrainData.value[y]
    for (let x = 0; x < cellCount; x++) {
      const height = row[x]
      drawColoredCell(x, y, height, cellW, cellH)
    }
  }

  // 3. 绘制等高线
  drawCompleteContourLines(cellW, cellH)
}

// 绘制彩色单元格
const drawColoredCell = (x, y, height, cellW, cellH) => {
  if (!ctx.value) return
  // 高程范围：120-180m
  const normalized = Math.min(1, Math.max(0, (height - 120) / 60))
  // HSL颜色（蓝→红）
  const hue = 240 - normalized * 240
  ctx.value.fillStyle = `hsl(${hue}, 70%, 50%)`
  ctx.value.fillRect(x * cellW, y * cellH, cellW, cellH)
  
  // 高程文本
  ctx.value.fillStyle = normalized > 0.5 ? '#fff' : '#000'
  ctx.value.font = '10px sans-serif'
  ctx.value.textAlign = 'center'
  ctx.value.textBaseline = 'middle'
  ctx.value.fillText(height.toFixed(0), x * cellW + cellW/2, y * cellH + cellH/2)
}

// 点击单元格修改高程
const handleCellClick = (e) => {
  if (!isActive.value || !canvas.value) return
  const rect = canvas.value.getBoundingClientRect()
  const x = e.clientX - rect.left
  const y = e.clientY - rect.top
  const cellX = Math.floor(x / (canvas.value.width / cellCount))
  const cellY = Math.floor(y / (canvas.value.height / cellCount))

  if (cellX >= 0 && cellX < cellCount && cellY >= 0 && cellY < cellCount) {
    const current = terrainData.value[cellY][cellX]
    const newHeight = Math.min(180, Math.max(120, current + (Math.random() > 0.5 ? 5 : -5)))
    terrainData.value[cellY][cellX] = roundToPrecision(newHeight, precision.value)
    drawGrid()
  }
}

// 生成平坦地形
const generateFlatTerrain = () => {
  const baseHeight = 150
  for (let y = 0; y < cellCount; y++) {
    for (let x = 0; x < cellCount; x++) {
      terrainData.value[y][x] = baseHeight
    }
  }
  drawGrid()
}

// 生成山丘地形
const generateHillTerrain = () => {
  const centerX = cellCount / 2
  const centerY = cellCount / 2
  const maxHeight = 180
  const minHeight = 120

  for (let y = 0; y < cellCount; y++) {
    for (let x = 0; x < cellCount; x++) {
      const dist = Math.sqrt(Math.pow(x - centerX, 2) + Math.pow(y - centerY, 2))
      const normalizedDist = Math.min(1, dist / (cellCount / 2))
      const height = maxHeight - (maxHeight - minHeight) * normalizedDist
      terrainData.value[y][x] = roundToPrecision(height, precision.value)
    }
  }
  drawGrid()
}

// 生成山谷地形
const generateValleyTerrain = () => {
  const centerX = cellCount / 2
  const centerY = cellCount / 2
  const minHeight = 120
  const maxHeight = 180

  for (let y = 0; y < cellCount; y++) {
    for (let x = 0; x < cellCount; x++) {
      const dist = Math.sqrt(Math.pow(x - centerX, 2) + Math.pow(y - centerY, 2))
      const normalizedDist = Math.min(1, dist / (cellCount / 2))
      const height = minHeight + (maxHeight - minHeight) * normalizedDist
      terrainData.value[y][x] = roundToPrecision(height, precision.value)
    }
  }
  drawGrid()
}

// 绘制完整等高线
const drawContours = () => {
  if (!ctx.value || !terrainData.value.length) return
  const cellW = canvas.value.width / cellCount
  const cellH = canvas.value.height / cellCount
  contourLines.value = {}

  // 高程范围
  const minH = 120
  const maxH = 180
  const interval = Number(contourInterval.value)

  // 生成等高线
  for (let level = minH; level <= maxH; level += interval) {
    const points = getContourPoints(level, cellW, cellH)
    if (points.length > 2) {
      // 按角度排序形成闭合线
      const center = {
        x: points.reduce((sum, p) => sum + p.x, 0) / points.length,
        y: points.reduce((sum, p) => sum + p.y, 0) / points.length
      }
      const sortedPoints = points.sort((a, b) => {
        const angleA = Math.atan2(a.y - center.y, a.x - center.x)
        const angleB = Math.atan2(b.y - center.y, b.x - center.x)
        return angleA - angleB
      })
      contourLines.value[level] = sortedPoints
    }
  }

  drawGrid()
}

// 获取等高线交点
const getContourPoints = (level, cellW, cellH) => {
  const points = []
  for (let y = 0; y < cellCount; y++) {
    for (let x = 0; x < cellCount; x++) {
      const current = terrainData.value[y][x]

      // 右侧边检测（添加精度容错，避免浮点误差导致的漏检）
      if (x < cellCount - 1) {
        const right = terrainData.value[y][x + 1]
        const currentDiff = level - current
        const rightDiff = level - right
        // 确保跨越高程线（容错0.1m）
        if (Math.sign(currentDiff) !== Math.sign(rightDiff) && Math.abs(currentDiff) > 0.1) {
          const ratio = Math.abs(currentDiff) / Math.abs(right - current)
          points.push({ 
            x: x * cellW + cellW, 
            y: y * cellH + cellH * ratio 
          })
        }
      }

      // 下侧边检测（同上容错处理）
      if (y < cellCount - 1) {
        const bottom = terrainData.value[y + 1][x]
        const currentDiff = level - current
        const bottomDiff = level - bottom
        if (Math.sign(currentDiff) !== Math.sign(bottomDiff) && Math.abs(currentDiff) > 0.1) {
          const ratio = Math.abs(currentDiff) / Math.abs(bottom - current)
          points.push({ 
            x: x * cellW + cellW * ratio, 
            y: y * cellH + cellH 
          })
        }
      }
    }
  }
  return points
}

// 绘制等高线（线+点+标签）
const drawCompleteContourLines = (cellW, cellH) => {
  if (!ctx.value || !Object.keys(contourLines.value).length) return

  const colorMap = { 5: '#8B00FF', 10: '#FF4500', 20: '#1E90FF' }
  const lineColor = colorMap[contourInterval.value] || '#FF4500'

  Object.keys(contourLines.value).forEach(levelStr => {
    const level = Number(levelStr)
    const points = contourLines.value[level]
    if (points.length < 3) return

    // 1. 绘制等高线（保持不变）
    ctx.value.strokeStyle = lineColor
    ctx.value.lineWidth = 3
    ctx.value.lineCap = 'round'
    ctx.value.lineJoin = 'round'
    ctx.value.beginPath()
    points.forEach((p, idx) => {
      idx === 0 ? ctx.value.moveTo(p.x, p.y) : ctx.value.lineTo(p.x, p.y)
    })
    if (points.length > 5) {
      ctx.value.lineTo(points[0].x, points[0].y) // 闭合线条
    }
    ctx.value.stroke()

    // 2. 绘制等高线点（保持不变）
    ctx.value.fillStyle = lineColor
    ctx.value.strokeStyle = '#fff'
    ctx.value.lineWidth = 2
    points.forEach(p => {
      ctx.value.beginPath()
      ctx.value.arc(p.x, p.y, 5, 0, Math.PI * 2)
      ctx.value.fill()
      ctx.value.stroke()
    })

    // 3. 绘制高程标签（修复位置偏移核心）
    if (points.length > 0) {
      const labelPoint = points[Math.floor(points.length / 4)]
      ctx.value.fillStyle = '#fff'
      ctx.value.strokeStyle = lineColor
      ctx.value.lineWidth = 2
      const text = `${level}m`
      ctx.value.font = '12px sans-serif'
      const textMetrics = ctx.value.measureText(text)
      const textWidth = textMetrics.width + 10 // 左右内边距各5px
      const textHeight = 18 // 固定高度，确保文字居中

      // 计算标签位置（居中对齐，避免偏左下）
      const labelX = labelPoint.x + 8 // 右移8px避开点
      const labelY = labelPoint.y - textHeight / 2 // 垂直居中

      // 绘制背景框（基于文字尺寸精确定位）
      ctx.value.fillRect(labelX, labelY, textWidth, textHeight)
      ctx.value.strokeRect(labelX, labelY, textWidth, textHeight)

      // 绘制文字（水平垂直双居中）
      ctx.value.fillStyle = lineColor
      ctx.value.textBaseline = 'middle' // 垂直居中
      ctx.value.textAlign = 'start' // 水平左对齐
      ctx.value.fillText(text, labelX + 5, labelY + textHeight / 2)
    }
  })
}


// 按精度四舍五入
const roundToPrecision = (value, precision) => {
  const factor = 1 / precision
  return Math.round(value * factor) / factor
}

// 防抖函数
function debounce(fn, delay) {
  let timer = null
  return function(...args) {
    clearTimeout(timer)
    timer = setTimeout(() => fn.apply(this, args), delay)
  }
}

// 重置功能
const reset = () => {
  initTerrainData()
  drawGrid()
}

// 组件卸载时清理
onUnmounted(() => {
  cleanupCanvas()
})

defineExpose({ reset, resizeCanvas })
</script>

<style scoped>
/* 图标兼容性处理 */
.fa-mound, .fa-downtime, .fa-draw-polygon {
  font-family: "FontAwesome";
  font-weight: normal;
  font-style: normal;
  font-size: 14px;
}
.fa-mound::before { content: "\f58d"; }
.fa-downtime::before { content: "\f349"; }
.fa-draw-polygon::before { content: "\f5ee"; }

/* 响应式按钮布局 */
@media (max-width: 768px) {
  .contour-controls {
    flex-direction: column;
    align-items: stretch;
  }
  .contour-controls > div {
    width: 100%;
    margin-bottom: 8px;
  }
  .contour-interval {
    justify-content: space-between;
  }
  select {
    width: 120px;
  }
}
</style>