<template>
  <div class="total-station">
    <!-- 画布容器 -->
    <div class="canvas-container border-2 border-gray-200 rounded-xl overflow-hidden shadow-sm bg-white mb-4" style="height: 400px;">
      <canvas 
        ref="canvasRef" 
        @click="handleCanvasClick"
        class="w-full h-full"  
        style="display: block;" 
      ></canvas>
    </div>

    
    <!-- 测量结果 -->
    <div class="measurement-results grid grid-cols-1 md:grid-cols-2 gap-4">
      <!-- 点信息卡片 -->
      <div class="bg-white p-4 rounded-xl border border-gray-100 shadow-sm">
        <h4 class="font-medium text-gray-800 mb-3 flex items-center">
          <i class="fa fa-map-marker mr-2 text-primary"></i>测量点信息
        </h4>
        <div v-if="points.length > 0" class="space-y-3">
          <div v-for="(point, idx) in points" :key="idx" class="flex justify-between text-sm">
            <span class="text-gray-600">点{{ idx + 1 }} 坐标:</span>
            <span class="font-medium text-gray-800">({{ point.x.toFixed(0) }}, {{ point.y.toFixed(0) }})</span>
          </div>
          <div v-for="(point, idx) in points" :key="idx" class="flex justify-between text-sm">
            <span class="text-gray-600">点{{ idx + 1 }} 高程:</span>
            <span class="font-medium text-gray-800">{{ point.z.toFixed(1) }}m</span>
          </div>
        </div>
        <div v-else class="text-sm text-gray-500">点击画布添加测量点</div>
      </div>
      
      <!-- 结果卡片 -->
      <div class="bg-white p-4 rounded-xl border border-gray-100 shadow-sm">
        <h4 class="font-medium text-gray-800 mb-3 flex items-center">
          <i class="fa fa-calculator mr-2 text-primary"></i>计算结果
        </h4>
        <div v-if="points.length >= 2" class="space-y-3 text-sm">
          <div class="flex justify-between">
            <span class="text-gray-600">图上距离:</span>
            <span class="font-medium text-gray-800">{{ mapDistance.toFixed(2) }}cm</span>
          </div>
          <div class="flex justify-between">
            <span class="text-gray-600">水平距离:</span>
            <span class="font-medium text-gray-800">{{ horizontalDistance.toFixed(2) }}m</span>
          </div>
          <div class="flex justify-between">
            <span class="text-gray-600">高差:</span>
            <span class="font-medium text-gray-800">{{ heightDifference.toFixed(2) }}m</span>
          </div>
          <div class="flex justify-between">
            <span class="text-gray-600">斜距:</span>
            <span class="font-medium text-gray-800">{{ slopeDistance.toFixed(2) }}m</span>
          </div>
        </div>
        <div v-else class="text-sm text-gray-500">需选择两个点计算结果</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick, onUnmounted } from 'vue'

const canvasRef = ref(null)
const canvas = ref(null)
const ctx = ref(null)
const points = ref([])
const isActive = ref(false)

// 测量结果
const mapDistance = ref(0)
const horizontalDistance = ref(0)
const heightDifference = ref(0)
const slopeDistance = ref(0)

// 接收外部参数
const props = defineProps({
  precision: { type: Number, default: 0.5 },
  scale: { type: Number, default: 1000 },
  active: { type: Boolean, default: false }
})

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
  resizeCanvas()
  window.addEventListener('resize', debouncedResize)
  drawCanvas()
}

// 清理画布资源
const cleanupCanvas = () => {
  window.removeEventListener('resize', debouncedResize)
  if (ctx.value) {
    ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height)
  }
}

// 防抖resize
const debouncedResize = debounce(() => {
  if (isActive.value) resizeCanvas()
}, 100)

// 画布尺寸调整
const resizeCanvas = () => {
  if (!canvas.value) return
  const container = canvas.value.parentElement
  // 宽度随容器自适应，高度固定为400px（与等高线一致）
  canvas.value.width = container.clientWidth
  canvas.value.height = 400  // 强制固定高度
  drawCanvas()
}

// 绘制画布
const drawCanvas = () => {
  if (!ctx.value) return
  ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height)
  drawGrid()
  points.value.forEach((point, idx) => drawPoint(point, idx + 1))
  if (points.value.length >= 2) drawLineBetweenPoints()
}

// 绘制网格
const drawGrid = () => {
  const gridSize = 50
  ctx.value.strokeStyle = '#e5e7eb'
  ctx.value.lineWidth = 1
  // 水平线
  for (let y = 0; y <= canvas.value.height; y += gridSize) {
    ctx.value.beginPath()
    ctx.value.moveTo(0, y)
    ctx.value.lineTo(canvas.value.width, y)
    ctx.value.stroke()
  }
  // 垂直线
  for (let x = 0; x <= canvas.value.width; x += gridSize) {
    ctx.value.beginPath()
    ctx.value.moveTo(x, 0)
    ctx.value.lineTo(x, canvas.value.height)
    ctx.value.stroke()
  }
}

// 绘制测量点
const drawPoint = (point, idx) => {
  const fillColor = idx === 1 ? '#165DFF' : '#36BFFA'
  // 白色边框
  ctx.value.fillStyle = '#fff'
  ctx.value.beginPath()
  ctx.value.arc(point.x, point.y, 8, 0, Math.PI * 2)
  ctx.value.fill()
  // 点本身
  ctx.value.fillStyle = fillColor
  ctx.value.beginPath()
  ctx.value.arc(point.x, point.y, 6, 0, Math.PI * 2)
  ctx.value.fill()
  // 点标签
  ctx.value.fillStyle = fillColor
  ctx.value.font = '12px sans-serif'
  ctx.value.fillText(`点${idx}`, point.x + 12, point.y - 12)
}

// 绘制两点连线
const drawLineBetweenPoints = () => {
  const p1 = points.value[0]
  const p2 = points.value[1]
  // 实际斜线
  ctx.value.strokeStyle = '#165DFF'
  ctx.value.lineWidth = 2.5
  ctx.value.setLineDash([])
  ctx.value.beginPath()
  ctx.value.moveTo(p1.x, p1.y)
  ctx.value.lineTo(p2.x, p2.y)
  ctx.value.stroke()
  // 水平投影线
  ctx.value.strokeStyle = '#c0c0c0'
  ctx.value.lineWidth = 1.5
  ctx.value.setLineDash([8, 4])
  ctx.value.beginPath()
  ctx.value.moveTo(p1.x, p1.y)
  ctx.value.lineTo(p2.x, p1.y)
  ctx.value.stroke()
}

// 处理点击事件
const handleCanvasClick = (e) => {
  if (!isActive.value || !canvas.value) return
  const rect = canvas.value.getBoundingClientRect()
  const scaleX = canvas.value.width / rect.width
  const scaleY = canvas.value.height / rect.height
  const x = (e.clientX - rect.left) * scaleX
  const y = (e.clientY - rect.top) * scaleY

  // 随机高程
  const z = 100 + Math.random() * 100
  if (points.value.length < 2) {
    points.value.push({ x, y, z })
  } else {
    points.value = [{ x, y, z }]
  }
  drawCanvas()
  calculateResults()
}

// 计算测量结果
const calculateResults = () => {
  if (points.value.length < 2) {
    mapDistance.value = 0
    horizontalDistance.value = 0
    heightDifference.value = 0
    slopeDistance.value = 0
    return
  }

  const p1 = points.value[0]
  const p2 = points.value[1]
  
  // 像素距离
  const pixelDx = p2.x - p1.x
  const pixelDy = p2.y - p1.y
  const pixelDistance = Math.sqrt(pixelDx * pixelDx + pixelDy * pixelDy)
  
  // 图上距离（1像素=0.1cm）
  mapDistance.value = roundToPrecision(pixelDistance * 0.1, 0.1)
  
  // 实际水平距离
  horizontalDistance.value = roundToPrecision(
    (mapDistance.value * props.scale) / 100,
    props.precision
  )
  
  // 高差
  heightDifference.value = roundToPrecision(
    Math.abs(p2.z - p1.z),
    props.precision
  )
  
  // 斜距
  slopeDistance.value = roundToPrecision(
    Math.sqrt(Math.pow(horizontalDistance.value, 2) + Math.pow(heightDifference.value, 2)),
    props.precision
  )
}

// 监听比例尺变化
watch(() => props.scale, () => {
  if (points.value.length >= 2) calculateResults()
})

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
  points.value = []
  calculateResults()
  drawCanvas()
}

// 组件卸载时清理
onUnmounted(() => {
  cleanupCanvas()
})

defineExpose({ reset, resizeCanvas })
</script>