<template>
  <div ref="container" class="pillar-scene-container">
    <canvas ref="canvas"></canvas>

    <!-- Optional overlay title -->
    <div class="overlay">
      <h1>Our 7 Pillars</h1>
      <p>Explore the foundation of our company</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from "vue"
import * as THREE from "three"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js"
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"

const container = ref<HTMLDivElement | null>(null)
const canvas = ref<HTMLCanvasElement | null>(null)

let renderer: THREE.WebGLRenderer
let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let controls: OrbitControls
let animationId: number

onMounted(() => {
  if (!container.value || !canvas.value) return

  // ---------------------------
  // Scene
  // ---------------------------
  scene = new THREE.Scene()
  scene.background = new THREE.Color("#050505")

  // ---------------------------
  // Camera
  // ---------------------------
  camera = new THREE.PerspectiveCamera(
    55,
    container.value.clientWidth / container.value.clientHeight,
    0.1,
    1000
  )

  camera.position.set(0, 5, 18)

  // ---------------------------
  // Renderer
  // ---------------------------
 renderer = new THREE.WebGLRenderer({
  canvas: canvas.value,
  antialias: true,
})

  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(
    container.value.clientWidth,
    container.value.clientHeight
  )

  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap

  // ---------------------------
  // Lights
  // ---------------------------
  const ambientLight = new THREE.AmbientLight(0xffffff, 1.2)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 2.5)

  directionalLight.position.set(10, 20, 10)

  directionalLight.castShadow = true

  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048

  directionalLight.shadow.camera.near = 0.5
  directionalLight.shadow.camera.far = 100

  scene.add(directionalLight)

  // ---------------------------
  // Ground
  // ---------------------------
  const groundGeometry = new THREE.CylinderGeometry(
    12,
    12,
    1,
    64
  )

  const groundMaterial = new THREE.MeshStandardMaterial({
    color: "#2b2b2b",
    roughness: 0.8,
    metalness: 0.1,
  })

  const ground = new THREE.Mesh(
    groundGeometry,
    groundMaterial
  )

  ground.position.y = -0.5
  ground.receiveShadow = true

  scene.add(ground)

  // ---------------------------
  // Roof
  // ---------------------------
  const roofGeometry = new THREE.CylinderGeometry(
    11,
    11,
    1,
    64
  )

  const roofMaterial = new THREE.MeshStandardMaterial({
    color: "#1b1b1b",
    roughness: 0.7,
  })

  const roof = new THREE.Mesh(
    roofGeometry,
    roofMaterial
  )

  roof.position.y = 10.5
  roof.castShadow = true

  scene.add(roof)

  // ---------------------------
  // Center platform
  // ---------------------------
  const centerGeometry = new THREE.CylinderGeometry(
    3,
    3,
    0.6,
    32
  )

  const centerMaterial = new THREE.MeshStandardMaterial({
    color: "#3f3f3f",
  })

  const center = new THREE.Mesh(
    centerGeometry,
    centerMaterial
  )

  center.position.y = 0.3

  center.receiveShadow = true
  center.castShadow = true

  scene.add(center)

  // ---------------------------
  // Load Pillar Model
  // ---------------------------
  const loader = new GLTFLoader()

  loader.load(
    "/ZSSpillar.glb",
    (gltf) => {
      const radius = 12

      for (let i = 0; i < 7; i++) {
        const angle = (i / 7) * Math.PI * 2

        const pillar = gltf.scene.clone(true)

        pillar.position.x = Math.cos(angle) * radius
        pillar.position.z = Math.sin(angle) * radius

        pillar.position.y = 0

        // Face center
        pillar.lookAt(0, 0, 0)

        // Scale if needed
        pillar.scale.set(2.2, 2.2, 2.2)

        pillar.traverse((child: any) => {
          if (child.isMesh) {
            child.castShadow = true
            child.receiveShadow = true
          }
        })

        scene.add(pillar)
      }
    },
    undefined,
    (error) => {
      console.error("Error loading GLB:", error)
    }
  )

  // ---------------------------
  // Fog for atmosphere
  // ---------------------------
  scene.fog = new THREE.Fog("#0d0d0d", 18, 40)

  // ---------------------------
  // Orbit Controls
  // ---------------------------
  controls = new OrbitControls(camera, renderer.domElement)

  // Smooth movement
  controls.enableDamping = true
  controls.dampingFactor = 0.05

  // User can rotate mostly sideways
  controls.minPolarAngle = Math.PI / 2.6
  controls.maxPolarAngle = Math.PI / 1.9

  // Disable panning
  controls.enablePan = false

  // Limited zoom
  controls.minDistance = 10
  controls.maxDistance = 24

  // Auto rotate slowly
  controls.autoRotate = true
  controls.autoRotateSpeed = 0.4

  // Target center
  controls.target.set(0, 4, 0)

  // ---------------------------
  // Resize
  // ---------------------------
  const onResize = () => {
    if (!container.value) return

    camera.aspect =
      container.value.clientWidth /
      container.value.clientHeight

    camera.updateProjectionMatrix()

    renderer.setSize(
      container.value.clientWidth,
      container.value.clientHeight
    )
  }

  window.addEventListener("resize", onResize)

  // ---------------------------
  // Animation
  // ---------------------------
  const animate = () => {
    animationId = requestAnimationFrame(animate)

    controls.update()

    renderer.render(scene, camera)
  }

  animate()

  // ---------------------------
  // Cleanup
  // ---------------------------
  onBeforeUnmount(() => {
    cancelAnimationFrame(animationId)
    window.removeEventListener("resize", onResize)

    renderer.dispose()
    controls.dispose()
  })
})
</script>

<style scoped>
.pillar-scene-container {
  position: relative;
  width: 100%;
  height: 50vh;
  overflow: hidden;
  background: #0d0d0d;
}

canvas {
  width: 100%;
  height: 100%;
  display: block;
}

.overlay {
  position: absolute;
  top: 40px;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
  color: white;
  pointer-events: none;
  z-index: 5;
}

.overlay h1 {
  font-size: 3rem;
  margin-bottom: 10px;
  letter-spacing: 2px;
}

.overlay p {
  font-size: 1rem;
  opacity: 0.8;
}

.pillar-scene-container {
  width: 100%;
  height: 100%;
  min-height: 600px;
  position: relative;
  overflow: hidden;
}
</style>