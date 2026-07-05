<template>
  <div class="home-container">
    <!-- Background -->
    <ParallaxSection class="background-animation" />

    
    <!-- Content -->
    <div class="content-overlay">
      <section
        v-for="(item, index) in sections"
        :key="index"
        class="section-wrapper"
        :ref="el => setSectionRef(el, index)"
      >
        <div
          class="content"
          :class="{ docked: docked[index] }"
          :style="getDockStyle(index)"
          :ref="el => setContentRef(el, index)"
        >
          <h1>{{ item.title }}</h1>
          <p>{{ item.text }}</p>
          
        </div>
      </section>
    </div>

    <!-- Separate 3D section -->
    <div class="pillar-wrapper">
      <PillarScene />
    </div>
    
  </div>
  
  
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import ParallaxSection from '../components/IntroAnimation.vue'
import PillarScene from '../components/Pillar.vue'


type Section = {
  title: string
  text: string
}

const sections: Section[] = [
  { title: 'Scalability', text: 'Handles growth efficiently' },
  { title: 'Customer Support', text: 'Fast and reliable help' },
  { title: 'Quality', text: 'High engineering standards' }
]

const sectionRefs = ref<HTMLElement[]>([])
const contentRefs = ref<HTMLElement[]>([])
const isAnimating = ref<boolean[]>([false, false, false])

const docked = ref<boolean[]>([false, false, false])

const setSectionRef = (el: Element | null, index: number) => {
  if (el instanceof HTMLElement) {
    sectionRefs.value[index] = el
  }
}

const setContentRef = (el: Element | null, index: number) => {
  if (el instanceof HTMLElement) {
    contentRefs.value[index] = el
  }
}

// Pre-calculate dock positions to avoid layout thrashing
const getDockRect = (index: number): DOMRect => {
  // These values match the docked CSS styles
  const viewportWidth = window.innerWidth
  const left = index * (viewportWidth / 3)
  const width = viewportWidth / 3
  
  return new DOMRect(left, 0, width, 180)
}

const animateToDocked = async (index: number) => {
  const contentEl = contentRefs.value[index]
  if (!contentEl || isAnimating.value[index]) return

  const sectionEl = sectionRefs.value[index]
  if (!sectionEl) return

  // Capture FIRST state (current position and dimensions)
  const firstRect = contentEl.getBoundingClientRect()
  
  // Immediately apply docked styles but keep it hidden briefly to prevent flicker
  docked.value[index] = true
  isAnimating.value[index] = true
  
  await nextTick()
  
  // Get the target docked rect (where it should end up)
  const targetRect = getDockRect(index)
  
  // Calculate the transform needed to go from FIRST to target position
  const deltaX = firstRect.left - targetRect.left
  const deltaY = firstRect.top - targetRect.top
  const scaleX = firstRect.width / targetRect.width
  const scaleY = firstRect.height / targetRect.height
  
  // Set initial transform to match FIRST state while element is already in docked position
  contentEl.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${scaleX}, ${scaleY})`
  contentEl.style.transition = 'none'
  
  // Force a reflow to ensure the transform is applied before transitioning
  contentEl.offsetHeight // read property to force reflow
  
  // Now animate to the final docked position
  requestAnimationFrame(() => {
    contentEl.style.transition = 'transform 0.5s cubic-bezier(0.2, 0.9, 0.4, 1.1)'
    contentEl.style.transform = 'translate(0, 0) scale(1, 1)'
  })
  
  // Clean up after animation completes
  setTimeout(() => {
    if (contentEl) {
      contentEl.style.transform = ''
      contentEl.style.transition = ''
    }
    isAnimating.value[index] = false
  }, 500)
}

const animateToOriginal = async (index: number) => {
  const contentEl = contentRefs.value[index]
  if (!contentEl || isAnimating.value[index]) return
  
  // Capture current docked position
  const firstRect = contentEl.getBoundingClientRect()
  
  // Remove docked state to get natural position
  docked.value[index] = false
  isAnimating.value[index] = true
  
  await nextTick()
  
  // Get the natural position after layout
  const targetRect = contentEl.getBoundingClientRect()
  
  // Calculate inverse transform
  const deltaX = firstRect.left - targetRect.left
  const deltaY = firstRect.top - targetRect.top
  const scaleX = firstRect.width / targetRect.width
  const scaleY = firstRect.height / targetRect.height
  
  // Start from docked position visually
  contentEl.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${scaleX}, ${scaleY})`
  contentEl.style.transition = 'none'
  
  contentEl.offsetHeight // force reflow
  
  // Animate to original position
  requestAnimationFrame(() => {
    contentEl.style.transition = 'transform 0.5s cubic-bezier(0.2, 0.9, 0.4, 1.1)'
    contentEl.style.transform = 'translate(0, 0) scale(1, 1)'
  })
  
  setTimeout(() => {
    if (contentEl) {
      contentEl.style.transform = ''
      contentEl.style.transition = ''
    }
    isAnimating.value[index] = false
  }, 500)
}

const handleScroll = () => {
  const scrollY = window.scrollY
  const viewportH = window.innerHeight

  sectionRefs.value.forEach((sectionEl, index) => {
    if (!sectionEl) return
    
    const rect = sectionEl.getBoundingClientRect()
    const absoluteTop = scrollY + rect.top
    const height = sectionEl.offsetHeight

    const shouldDock = scrollY > absoluteTop + height - viewportH

    // Only trigger animation when state actually changes and not already animating
    if (shouldDock !== docked.value[index] && !isAnimating.value[index]) {
      if (shouldDock) {
        animateToDocked(index)
      } else {
        animateToOriginal(index)
      }
    }
  })
}

// Throttle scroll events for better performance
let ticking = false
const throttledHandleScroll = () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      handleScroll()
      ticking = false
    })
    ticking = true
  }
}

onMounted(() => {
  window.addEventListener('scroll', throttledHandleScroll)
  // Initial check
  handleScroll()
})

onBeforeUnmount(() => {
  window.removeEventListener('scroll', throttledHandleScroll)
})

const getDockStyle = (index: number) => {
  if (!docked.value[index]) return {}

  return {
    position: 'fixed',
    top: '0px',
    left: `${index * 33.3333}%`,
    width: '33.3333%',
    height: '180px',
    zIndex: 100 + index, // Ensure proper stacking order
    margin: '0',
    padding: '0',
    borderRadius: '0',
    backdropFilter: 'none',
    boxShadow: 'none',
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'center'
  }
}
</script>

<style scoped>
.home-container {
  position: relative;
  width: 100%;
}

/* Background */
.background-animation {
  position: fixed;
  inset: 0;
  z-index: 1;
}

.content-overlay {
  position: relative;
  z-index: 2;
}

/* Scroll area - each section takes 140vh of scroll space */
.section-wrapper {
  height: 140vh;
  display: flex;
  justify-content: center;
  align-items: flex-start;
}

/* Default content styling - sticky behavior */
.content {
  position: sticky;
  top: 180px;
  width: 100%;
  max-width: 1000px;
  padding: 40px;
  border-radius: 0px;
  color: white;
  display: flex;
  flex-direction: column;
  justify-content: center;
  background: linear-gradient(
    135deg,
    rgba(0, 0, 0, 0.8),
    rgba(0, 0, 0, 0.6)
  );
  backdrop-filter: blur(8px);
  will-change: transform;
  margin: 0 auto;
}

/* Docked state overrides - applied via inline styles and class */
.content.docked {
  position: fixed !important;
  top: 0 !important;
  width: 33.3333% !important;
  height: 180px !important;
  margin: 0 !important;
  padding: 0 !important;
  border-radius: 0 !important;
  backdrop-filter: none !important;
  box-shadow: none !important;
  background: linear-gradient(
    135deg,
    rgba(0, 0, 0, 0.9),
    rgba(0, 0, 0, 0.7)
  ) !important;
  cursor: default;
}

/* Typography adjustments for docked state */
.content.docked h1 {
  font-size: 1.2rem;
  margin: 0;
}

.content.docked p {
  font-size: 0.9rem;
  margin: 4px 0 0;
}

/* Default typography */
.content h1 {
  font-size: 2rem;
  margin-bottom: 0.5rem;
}

.content p {
  font-size: 1rem;
}

/* Mobile responsiveness */
@media (max-width: 768px) {
  .section-wrapper {
    height: 120vh;
  }
  
  .content {
    padding: 30px 20px;
    top: 120px;
  }
  
  .content.docked h1 {
    font-size: 0.9rem;
  }
  
  .content.docked p {
    font-size: 0.7rem;
  }
}



.home-container {
  position: relative;
  width: 100%;
  overflow-x: hidden;
}

/* KEEP background fixed */
.background-animation {
  position: fixed;
  inset: 0;
  z-index: 0;
}

/* Main content above background */
.content-overlay {
  position: relative;
  z-index: 2;
}

/* Separate pillar section */
.pillar-wrapper {
  position: relative;
  z-index: 5;

  width: 100%;
  height: 60vh;
  min-height: 600px;

  background: #050505;

  margin-top: 100px;
}
</style>