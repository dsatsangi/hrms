<template>
  <div class="w-full">
    <div class="flex items-center justify-between mb-2">
      <div class="text-sm font-medium text-gray-700">
        {{ title }}
      </div>
      <div v-if="statusText" class="text-xs text-gray-500">{{ statusText }}</div>
    </div>

    <div
      class="relative w-full overflow-hidden rounded border bg-gray-50"
      :style="{ height: height + 'px' }"
    >
      <!-- Live video -->
      <video
        v-show="!captured && !error"
        ref="videoEl"
        class="absolute inset-0 w-full h-full object-cover"
        muted
        playsinline
        autoplay
      ></video>

      <!-- Captured preview -->
      <canvas
        v-show="captured && !error"
        ref="canvasEl"
        class="absolute inset-0 w-full h-full object-cover"
      ></canvas>

      <!-- Overlay: loading -->
      <div
        v-if="loading && !error"
        class="absolute inset-0 flex items-center justify-center bg-white/60"
      >
        <FeatherIcon name="loader" class="w-5 h-5 animate-spin text-gray-600" />
      </div>

      <!-- Overlay: error -->
      <div v-if="error" class="absolute inset-0 flex flex-col items-center justify-center gap-2 p-3 text-center">
        <FeatherIcon name="alert-triangle" class="w-5 h-5 text-amber-500" />
        <div class="text-sm text-gray-700">{{ error }}</div>
        <div class="text-xs text-gray-500">Check camera permissions and try again.</div>
        <div class="flex gap-2 mt-2">
          <Button size="sm" @click="retry">{{ __("Retry") }}</Button>
        </div>
      </div>

      <!-- Controls -->
      <div v-if="!error" class="absolute bottom-2 left-0 right-0 flex items-center justify-center gap-3 px-3">
        <template v-if="!captured">
          <Button size="sm" class="px-3" :disabled="!streamReady || loading" @click="capture">
            <template #prefix>
              <FeatherIcon name="camera" class="w-4" />
            </template>
            {{ __("Snap") }}
          </Button>
        </template>
        <template v-else>
          <Button size="sm" class="px-3" variant="subtle" @click="retake">
            <template #prefix>
              <FeatherIcon name="rotate-ccw" class="w-4" />
            </template>
            {{ __("Retake") }}
          </Button>
          <Button size="sm" class="px-3" @click="emitUsePhoto">
            <template #prefix>
              <FeatherIcon name="check-circle" class="w-4" />
            </template>
            {{ __("Use Photo") }}
          </Button>
        </template>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref, watch, defineExpose, inject } from 'vue'
import { Button, FeatherIcon } from 'frappe-ui'

const __ = inject('$translate')

// Props
const props = defineProps({
  title: { type: String, default: 'Face Scan (optional)' },
  height: { type: Number, default: 170 },
  autoStart: { type: Boolean, default: false },
})

// Emits
const emit = defineEmits(['captured'])

// State
const videoEl = ref(null)
const canvasEl = ref(null)
const stream = ref(null)
const loading = ref(false)
const captured = ref(false)
const error = ref('')
const streamReady = ref(false)
const statusText = ref('')

async function start() {
  if (stream.value || loading.value) return
  error.value = ''
  loading.value = true
  statusText.value = __('Starting camera…')
  try {
    const s = await navigator.mediaDevices.getUserMedia({
      video: {
        facingMode: 'user',
        width: { ideal: 1280 },
        height: { ideal: 720 },
      },
      audio: false,
    })
    stream.value = s
    if (videoEl.value) {
      videoEl.value.srcObject = s
      await videoEl.value.play().catch(() => {})
    }
    streamReady.value = true
    statusText.value = __('Align your face and tap Snap')
  } catch (e) {
    console.error('FaceScan start error:', e)
    error.value = e?.message || __('Unable to access camera')
  } finally {
    loading.value = false
  }
}

function stop() {
  if (stream.value) {
    for (const t of stream.value.getTracks?.() || []) t.stop()
    stream.value = null
  }
  streamReady.value = false
}

function capture() {
  if (!videoEl.value || !canvasEl.value) return
  const w = videoEl.value.videoWidth
  const h = videoEl.value.videoHeight
  if (!w || !h) return
  canvasEl.value.width = w
  canvasEl.value.height = h
  const ctx = canvasEl.value.getContext('2d')
  ctx.drawImage(videoEl.value, 0, 0, w, h)
  captured.value = true
}

function retake() {
  captured.value = false
}

function emitUsePhoto() {
  if (!canvasEl.value) return
  try {
    canvasEl.value.toBlob((blob) => {
      if (blob) emit('captured', blob)
    }, 'image/jpeg', 0.92)
  } catch (e) {
    console.error('FaceScan toBlob error:', e)
  }
}

function retry() {
  stop()
  start()
}

onMounted(() => {
  if (props.autoStart) start()
})

onBeforeUnmount(() => {
  stop()
})

// Expose controls to parent (so parent can start/stop when modal opens/closes)
defineExpose({ start, stop })
</script>

<style scoped>
/* Prevent iOS Safari from adding full-screen behavior */
video { 
  transform: scaleX(-1); /* mirror for front camera for UX */
}
canvas {
  transform: scaleX(-1);
}
</style>
