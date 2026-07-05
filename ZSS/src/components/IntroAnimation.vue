<template>
  <div class="scroll-container">
    <canvas ref="canvas"></canvas>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger)

const canvas = ref<HTMLCanvasElement | null>(null)

const FRAME_COUNT = 600 // adjust if needed

const images: HTMLImageElement[] = []
const state = { frame: 0 }

let lastFrame = -1

function getFrameSrc(index: number) {
  return `/frames/frame_${String(index).padStart(5, '0')}.jpg`
}

function preloadImages(): Promise<void> {
  return new Promise((resolve) => {
    let loaded = 0

    for (let i = 1; i <= FRAME_COUNT; i++) {
      const img = new Image()
      img.src = getFrameSrc(i)

      img.onload = () => {
        loaded++
        if (loaded === FRAME_COUNT) resolve()
      }

      images.push(img)
    }
  })
}

function render(ctx: CanvasRenderingContext2D) {
  const index = Math.min(FRAME_COUNT - 1, Math.floor(state.frame))

  // skip if same frame
  if (index === lastFrame) return
  lastFrame = index

  const img = images[index]
  if (!img) return

  const canvasEl = ctx.canvas

  ctx.clearRect(0, 0, canvasEl.width, canvasEl.height)

  const scale = Math.max(
    canvasEl.width / img.width,
    canvasEl.height / img.height
  )

  const x = (canvasEl.width - img.width * scale) / 2
  const y = (canvasEl.height - img.height * scale) / 2

  ctx.drawImage(img, x, y, img.width * scale, img.height * scale)
}

onMounted(async () => {
  const canvasEl = canvas.value!
  const ctx = canvasEl.getContext('2d')!

  // set canvas resolution properly
  canvasEl.width = window.innerWidth
  canvasEl.height = window.innerHeight

  await preloadImages()

  render(ctx)

  gsap.to(state, {
    frame: FRAME_COUNT - 1,
    ease: 'none',
    scrollTrigger: {
      trigger: canvasEl,
      start: 'top top',
      end: `+=${FRAME_COUNT * 10}`, // slower animation
      scrub: 10, // smoothing
      pin: true,
      anticipatePin: 1
    },
    onUpdate: () => render(ctx)
  })
})

</script>

<style scoped>
.scroll-container {
  height: 800vh; /* more scroll space */
}

canvas {
  position: sticky;
  top: 0;
  width: 100%;
  height: 100vh;
  display: block;
}
</style>