<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue'

const props = defineProps<{
  src: string
  alt?: string
  title?: string
  imgClass?: string
  containerClass?: string
}>()

const isOpen = ref(false)
const scale = ref(1.0)
const translateX = ref(0)
const translateY = ref(0)
const isDragging = ref(false)
const dragStart = { x: 0, y: 0 }

const zoomPercentage = computed(() => `${Math.round(scale.value * 100)}%`)

function openModal(e: MouseEvent) {
  e.stopPropagation()
  resetTransform()
  isOpen.value = true
}

function closeModal() {
  isOpen.value = false
  resetTransform()
}

function resetTransform() {
  scale.value = 1.0
  translateX.value = 0
  translateY.value = 0
  isDragging.value = false
}

function zoomIn(e?: MouseEvent) {
  e?.stopPropagation()
  scale.value = Math.min(4.0, Number((scale.value + 0.3).toFixed(2)))
}

function zoomOut(e?: MouseEvent) {
  e?.stopPropagation()
  scale.value = Math.max(0.5, Number((scale.value - 0.3).toFixed(2)))
}

function handleWheel(e: WheelEvent) {
  e.preventDefault()
  e.stopPropagation()
  const delta = e.deltaY < 0 ? 0.2 : -0.2
  const newScale = Math.min(4.0, Math.max(0.5, Number((scale.value + delta).toFixed(2))))
  scale.value = newScale
}

function handleDoubleClick(e: MouseEvent) {
  e.stopPropagation()
  if (scale.value <= 1.05) {
    scale.value = 2.0
  } else {
    resetTransform()
  }
}

function startDrag(e: MouseEvent) {
  e.stopPropagation()
  // Only drag on primary mouse click
  if (e.button !== 0) return
  isDragging.value = true
  dragStart.x = e.clientX - translateX.value
  dragStart.y = e.clientY - translateY.value
}

function onDrag(e: MouseEvent) {
  if (!isDragging.value) return
  e.preventDefault()
  translateX.value = e.clientX - dragStart.x
  translateY.value = e.clientY - dragStart.y
}

function stopDrag() {
  isDragging.value = false
}

function handleKeydown(e: KeyboardEvent) {
  if (!isOpen.value) return
  if (e.key === 'Escape') {
    closeModal()
  } else if (e.key === '+' || e.key === '=') {
    zoomIn()
  } else if (e.key === '-' || e.key === '_') {
    zoomOut()
  } else if (e.key === '0') {
    resetTransform()
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
  window.addEventListener('mouseup', stopDrag)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  window.removeEventListener('mouseup', stopDrag)
})
</script>

<template>
  <div
    class="relative group cursor-zoom-in inline-block w-full overflow-hidden"
    :class="containerClass"
    @click="openModal"
  >
    <img
      :src="src"
      :alt="alt || title || 'image'"
      class="transition-all duration-300 group-hover:brightness-105"
      :class="imgClass"
    />
    <!-- Floating zoom icon badge at top-right -->
    <div
      class="absolute top-1.5 right-1.5 px-2 py-0.5 rounded-full bg-slate-950/85 border border-cyan-400/50 text-cyan-300 text-[10px] flex items-center gap-1 opacity-0 group-hover:opacity-100 transition-opacity duration-200 pointer-events-none shadow-lg backdrop-blur-sm"
    >
      <span class="text-xs">🔍</span>
      <span class="font-mono text-[9px] font-bold">點擊放大</span>
    </div>

    <!-- Lightbox Modal via Teleport -->
    <Teleport to="body">
      <Transition name="zoom-fade">
        <div
          v-if="isOpen"
          class="fixed inset-0 z-[99999] bg-slate-950/90 backdrop-blur-md flex flex-col items-center justify-between p-4 select-none overflow-hidden"
          @click="closeModal"
        >
          <!-- Top Bar: Title & Close Button -->
          <div
            class="w-full flex items-center justify-between px-6 pt-2 z-10"
            @click.stop
          >
            <!-- Left/Center Title -->
            <div
              v-if="title || alt"
              class="px-4 py-1.5 rounded-full bg-slate-900/90 border border-slate-700/80 text-cyan-300 font-mono text-xs shadow-xl backdrop-blur-sm flex items-center gap-2"
            >
              <span class="w-2 h-2 rounded-full bg-cyan-400 animate-pulse"></span>
              <span>{{ title || alt }}</span>
            </div>
            <div v-else></div>

            <!-- Right: Hint & Close Button -->
            <div class="flex items-center gap-3">
              <span class="text-xs font-mono text-slate-400 hidden sm:inline-block">
                滾輪縮放 · 拖曳平移 · ESC關閉
              </span>
              <button
                class="w-9 h-9 rounded-full bg-slate-800/90 hover:bg-cyan-500/20 text-slate-200 hover:text-cyan-400 border border-slate-700/80 hover:border-cyan-400/60 flex items-center justify-center transition-all duration-200 text-lg font-bold shadow-xl"
                title="關閉 (ESC)"
                @click="closeModal"
              >
                ✕
              </button>
            </div>
          </div>

          <!-- Enlarged Image Viewport (Theater-scale sizing) -->
          <div
            class="flex-1 w-full flex items-center justify-center overflow-hidden my-auto"
            @click.self="closeModal"
            @wheel="handleWheel"
            @mousemove="onDrag"
          >
            <div
              class="relative transition-transform duration-75 select-none"
              :class="isDragging ? 'cursor-grabbing' : 'cursor-grab'"
              :style="{
                transform: `translate(${translateX}px, ${translateY}px) scale(${scale})`,
              }"
              @mousedown="startDrag"
              @dblclick="handleDoubleClick"
              @click.stop
            >
              <!--
                Image forced to Theater Dimensions:
                h-[72vh] ensures even a 500px image expands to 1200px+ across the screen,
                dramatically boosting the magnification!
              -->
              <img
                :src="src"
                :alt="alt || title || 'enlarged image'"
                class="max-w-[88vw] h-[72vh] object-contain rounded-lg shadow-2xl border border-cyan-500/40 ring-1 ring-cyan-400/20 pointer-events-none"
                draggable="false"
              />
            </div>
          </div>

          <!-- Bottom Floating Controls Capsule -->
          <div
            class="px-5 py-2 rounded-full bg-slate-900/95 border border-cyan-500/40 shadow-2xl backdrop-blur-md flex items-center gap-4 z-10 font-mono text-xs text-slate-200 mb-2"
            @click.stop
          >
            <!-- Zoom Out Button -->
            <button
              class="w-7 h-7 rounded-full bg-slate-800 hover:bg-cyan-500/30 text-slate-200 hover:text-cyan-300 border border-slate-700 flex items-center justify-center transition-all text-sm font-bold"
              title="縮小 (-)"
              @click="zoomOut"
            >
              －
            </button>

            <!-- Zoom Scale Display -->
            <span class="text-cyan-400 font-bold min-w-[50px] text-center tracking-wider">
              {{ zoomPercentage }}
            </span>

            <!-- Zoom In Button -->
            <button
              class="w-7 h-7 rounded-full bg-slate-800 hover:bg-cyan-500/30 text-slate-200 hover:text-cyan-300 border border-slate-700 flex items-center justify-center transition-all text-sm font-bold"
              title="放大 (+)"
              @click="zoomIn"
            >
              ＋
            </button>

            <span class="w-[1px] h-4 bg-slate-700 mx-1"></span>

            <!-- Reset Button -->
            <button
              class="px-2.5 py-1 rounded bg-slate-800 hover:bg-cyan-500/30 text-slate-300 hover:text-cyan-300 border border-slate-700 text-[11px] font-medium transition-all flex items-center gap-1"
              title="重設大小與位置 (0)"
              @click="resetTransform"
            >
              <span>↺</span>
              <span>重設</span>
            </button>
          </div>
        </div>
      </Transition>
    </Teleport>
  </div>
</template>

<style scoped>
.zoom-fade-enter-active,
.zoom-fade-leave-active {
  transition: opacity 0.22s ease, transform 0.22s ease;
}

.zoom-fade-enter-from,
.zoom-fade-leave-to {
  opacity: 0;
  transform: scale(0.96);
}
</style>
