<template>
  <div class="surveying-tool max-w-7xl mx-auto p-4 sm:p-6 lg:p-8">
    <!-- 头部标题 -->
    <div class="header mb-8 pb-4 border-b border-gray-100">
      <h1 class="text-2xl md:text-3xl font-bold text-gray-800 text-center">
        <i class="fa fa-map-signs mr-2 text-primary"></i>测绘工具模拟器
      </h1>
    </div>
    
    <!-- 主内容区 -->
    <div class="main-content grid grid-cols-1 lg:grid-cols-5 gap-6">
      <!-- 左侧参数面板 -->
      <div class="lg:col-span-1">
        <div class="sticky top-4">
          <ParameterControls 
            :precision="Number(precision)"
            :scale="scale"
            :current-mode="currentMode"
            @update:precision="precision = $event"
            @update:scale="scale = $event"
            @reset="handleReset"
            @switch-mode="switchMode"
          />
        </div>
      </div>
      
      <!-- 右侧主功能区 -->
      <div class="lg:col-span-4">
        <div class="mode-header mb-4">
          <h2 class="text-xl font-semibold text-gray-800 flex items-center">
            <i 
              class="mr-2" 
              :class="currentMode === 'totalStation' ? 'fa fa-theater-masks text-primary' : 'fa fa-map-o text-primary'"
            ></i>
            {{ currentMode === 'totalStation' ? '全站仪模拟器' : '等高线绘制器' }}
          </h2>
        </div>
        
        <div class="mode-content">
          <TotalStation 
            v-if="currentMode === 'totalStation'"
            ref="totalStationRef"
            :precision="Number(precision)"
            :scale="scale"
            :active="currentMode === 'totalStation'"
          />
          
          <ContourMap 
            v-else
            ref="contourMapRef"
            :precision="Number(precision)"
            :active="currentMode === 'contourMap'"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import ParameterControls from './ParameterControls.vue'
import TotalStation from './TotalStation.vue'
import ContourMap from './ContourMap.vue'

// 状态变量
const precision = ref(0.5)
const scale = ref(1000)
const currentMode = ref('totalStation') // 'totalStation' 或 'contourMap'

// 组件引用
const totalStationRef = ref(null)
const contourMapRef = ref(null)

// 切换模式
const switchMode = () => {
  const newMode = currentMode.value === 'totalStation' ? 'contourMap' : 'totalStation'
  currentMode.value = newMode
  
  // 切换后等待DOM更新，强制触发画布尺寸重置
  nextTick(() => {
    if (newMode === 'totalStation' && totalStationRef.value) {
      totalStationRef.value.resizeCanvas()  // 调用全站仪的尺寸重置
    } else if (newMode === 'contourMap' && contourMapRef.value) {
      contourMapRef.value.resizeCanvas()    // 调用等高线的尺寸重置
    }
  })
}

// 重置当前工具
const handleReset = () => {
  if (currentMode.value === 'totalStation' && totalStationRef.value) {
    totalStationRef.value.reset()
  } else if (currentMode.value === 'contourMap' && contourMapRef.value) {
    contourMapRef.value.reset()
  }
}
</script>

<style scoped>
/* 模式标题下划线 */
.mode-header h2 {
  position: relative;
  padding-bottom: 6px;
}
.mode-header h2::after {
  content: '';
  position: absolute;
  left: 0;
  bottom: 0;
  width: 40px;
  height: 3px;
  background-color: #165DFF;
  border-radius: 2px;
}

/* 响应式调整 */
@media (max-width: 1024px) {
  .main-content {
    gap: 40px;
  }
}

@media (max-width: 768px) {
  .main-content {
    gap: 20px;
  }
  .header {
    margin-bottom: 40px;
  }
}
</style>