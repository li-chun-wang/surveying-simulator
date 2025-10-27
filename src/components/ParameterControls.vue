<template>
  <div class="parameter-controls bg-white p-5 rounded-xl shadow-md border border-gray-100">
    <h3 class="text-lg font-semibold mb-4 text-gray-800 flex items-center">
      <i class="fa fa-sliders mr-2 text-primary"></i>参数设置
    </h3>
    
    <!-- 测量精度 -->
    <div class="control-item mb-5">
      <label class="block text-sm font-medium mb-2 text-gray-600">测量精度: {{ localPrecision }}m</label>
      <input 
        type="range" 
        min="0.1" 
        max="5" 
        step="0.1" 
        v-model="localPrecision"
        class="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer accent-primary"
      >
    </div>
    
    <!-- 比例尺 -->
    <div class="control-item mb-6">
      <label class="block text-sm font-medium mb-2 text-gray-600">比例尺: 1:{{ localScale }}</label>
      <input 
        type="range" 
        min="100" 
        max="10000" 
        step="100" 
        v-model="localScale"
        class="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer accent-primary"
      >
    </div>
    
    <div class="control-buttons flex gap-3">
      <button 
        @click="$emit('reset')"
        class="flex-1 px-4 py-2 bg-gray-100 hover:bg-gray-200 text-gray-700 rounded-lg transition flex items-center justify-center"
      >
        <i class="fa fa-refresh mr-2"></i>重置
      </button>
      <button 
        @click="$emit('switch-mode')"
        class="flex-1 px-4 py-2 bg-primary hover:bg-primary/90 text-white rounded-lg transition flex items-center justify-center"
      >
        <i class="fa fa-exchange mr-2"></i>
        {{ currentMode === 'totalStation' ? '等高线模式' : '全站仪模式' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { defineProps, defineEmits, ref, watch } from 'vue'

const props = defineProps({
  precision: { type: Number, default: 0.5 },
  scale: { type: Number, default: 1000 },
  currentMode: { type: String, default: 'totalStation' }
})

const emits = defineEmits(['update:precision', 'update:scale', 'reset', 'switch-mode'])

// 本地变量处理
const localPrecision = ref(Number(props.precision))
const localScale = ref(Number(props.scale))

// 同步数据
watch(localPrecision, (newVal) => emits('update:precision', Number(newVal)))
watch(localScale, (newVal) => emits('update:scale', Number(newVal)))
watch(() => props.precision, (newVal) => localPrecision.value = Number(newVal))
watch(() => props.scale, (newVal) => localScale.value = Number(newVal))
</script>

<style scoped>
.bg-primary { background-color: #165DFF; }
/* 美化滑块样式 */
input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #165DFF;
  cursor: pointer;
}
</style>